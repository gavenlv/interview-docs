# 后端 · Go（面试题库）

本文件聚焦 Go 后端工程师的真实工程能力考察，覆盖 goroutine 与 channel 实践、GMP 调度、内存模型与同步、GC 调优、pprof 性能剖析、内存泄漏定位、优雅退出、服务端与微服务实践等关键领域。题目均为线上或项目中可复现的场景化问题，重点考察候选人的排查思路、量化依据与取舍判断，而非概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

### Q1. goroutine 数量持续增长不回落，怎么定位是否泄漏？

**问题：** 线上服务跑了一天后 goroutine 数量从几千涨到几十万，内存也跟着涨，怎么确认是不是 goroutine 泄漏，具体怎么定位到泄漏点？

**期望加分项：**
- 能给出定位链路：先看增长曲线区分"流量高峰"与"真泄漏"，再用 pprof 采样 goroutine 栈
- 会用 runtime.NumGoroutine 打点监控，知道结合内存曲线一起看
- 能按阻塞类型分类定位（channel 收发无配对、select 永久阻塞、锁等待、timer 回调）并对应到代码模式
- 知道 goroutine 泄漏最终也表现为内存泄漏，两者要联动排查
- 能说出修复后如何验证（曲线回落、pprof 采样对比）

**减分项：**
- 只会背"用 pprof"，说不出具体看哪个视图、怎么读 goroutine 栈
- 把临时流量高峰当泄漏，没有先做趋势判断
- 不知道泄漏常见根因模式（channel 无消费者、select 缺超时、ticker 未 Stop）
- 答不出从栈信息（如 gopark、chan send/recv）反推泄漏点的技巧

**解答：**
思路分三步：先确认"是泄漏"，再"抓现场"，最后"对代码"。第一步，监控 runtime.NumGoroutine 与 RSS 曲线：如果 goroutine 数随流量同涨同落是正常伸缩，持续爬升不回落才是泄漏。第二步，接入 pprof（net/http/pprof 建议挂独立端口并加访问控制），抓 `go tool pprof http://host/debug/pprof/goroutine`，或直接看 `pprof/goroutine?debug=2` 的完整栈；重点看处于 `gopark` 状态、等待在 `chan send/recv`、`select`、`sync.Mutex` 上的 goroutine——它们已经"卡死"了。第三步，按栈信息反查代码：常见泄漏模式包括向无消费者的 channel 写（无人接收）、select 中缺少超时分支、time.NewTicker 未 Stop 导致后台任务永不退出、for 循环内 `go func()` 无并发上限且任务阻塞、goroutine 内调用没有超时保护的下游接口。实践中的坑：泄漏往往要跑一段时间才显形，建议压测时同时打 NumGoroutine 与 pprof 快照做基线对比；修复后不能只看内存回落，要同时观察 goroutine 曲线回到基线才算闭环。

**延伸考点：** 有缓冲的 channel 就不会泄漏吗？在设计 API 时怎么通过 context 贯穿来从根上防止 goroutine 泄漏？

---

### Q2. 大量 goroutine 并发写同一个 map，线上直接 fatal error，怎么办？

**问题：** 线上服务在高并发下报 `fatal error: concurrent map writes` 直接崩溃，为什么 recover 救不了？线上怎么避免、开发期怎么提前发现？

**期望加分项：**
- 知道这是 runtime fatal error，不是 panic，recover 无效，进程必然退出
- 能解释原因：map 内部结构（bucket、hmap 状态字段）被并发写破坏，检测到即终止
- 能按场景给方案并说清取舍：RWMutex、sync.Map、分片 map（sharded）
- 能指出 `-race` 可以在开发期/CI 提前捕获，而非等线上崩溃
- 主动提并发 map 的替代思路：channel 串行化写、COW（copy-on-write）整表替换

**减分项：**
- 只会说"加锁"，说不出三种方案各自的适用场景与代价
- 误以为 recover 能兜住 fatal error
- 不知道 sync.Map 适合读多写少、key 稳定分散的场景，滥用后性能反而更差
- 不知道 map 的"并发读不写"是安全的这一边界

**解答：**
先说结论：`concurrent map writes` 是 runtime 层的致命错误，只能靠进程退出+自动拉起兜底，所以重点是"防"。开发期在 CI 用 `go test -race`，本地 `go run -race`，数据竞争在运行时报出具体文件和行号，这是成本最低的防线。线上已发生，只能从崩溃日志和监控恢复后补排查。方案按读写比例选：读多写少用 `sync.RWMutex` 保护；读多写多、key 分散且不断增长，用 `sync.Map`（内部空间换时间，写时复制，适合"写一次读多次"的配置类数据）；热点集中、写频繁，用分片锁：把 map 按 key 的 hash 高几位分成 N 片，每片一把 Mutex，锁粒度降到 1/N，分片数取 2 的幂便于位运算。坑：`sync.Map` 不是万能，读写都频繁时锁 map + 合理分片往往更快；只读不写确实安全，但"判断存在再写"这种复合操作不原子，仍需锁；修复后要压测确认热点路径无回归，别为了消除 panic 引入更重的锁竞争。

**延伸考点：** sync.Map 与"分片 map + 锁"在读写混合场景下性能差异的量化依据是什么？分片数怎么选？

---

### Q3. 并发请求多个上游，要求 200ms 内返回最快结果，怎么写？

**问题：** 需要同时调用两个上游接口（或本地两个数据源），取最快到达的那个结果返回，整个函数要在 200ms 内完成，超时则返回兜底，代码怎么组织？

**期望加分项：**
- 能写出 select 多路复用：两个 goroutine + channel + select 收最快的
- 主动处理慢 goroutine 的泄漏：channel 要带缓冲（容量 1），否则 select 返回后慢 goroutine 写阻塞
- 结合 context.WithCancel：拿到最快结果后 cancel 掉慢的那个
- 能覆盖超时：select 加 time.After（并指出循环里 time.After 的累积问题）
- 能说清"谁先到谁生效"与"结果还要继续用 ctx"的边界

**减分项：**
- 只会用 time.Sleep 轮询或串行调用
- 不知道 select 是随机多路、会优先满足就绪分支
- 忽略慢 goroutine 泄漏，只答"select 一下就完事"
- 不会用 context 做连带取消

**解答：**
核心是 select 多路复用 + 缓冲 channel + context 取消，三层缺一不可。骨架：

```go
func fastest(ctx context.Context, a, b func(ctx context.Context) (string, error)) (string, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    ch := make(chan string, 2) // 关键：缓冲，避免慢分支写阻塞泄漏
    go func() { if r, err := a(ctx); err == nil { ch <- r } }()
    go func() { if r, err := b(ctx); err == nil { ch <- r } }()
    select {
    case r := <-ch:
        return r, nil // 这里 cancel() 由 defer 触发，另一个 goroutine 的 ctx.Done() 会让其尽早返回
    case <-time.After(200 * time.Millisecond):
        return "", context.DeadlineExceeded
    }
}
```

坑有三个：一是 channel 必须带缓冲，否则"最快的那个被 select 拿走"后，慢 goroutine 的写操作永远无人接收而永久阻塞——这就是 goroutine 泄漏；二是超时用 `time.NewTimer` 配合 defer Stop 更严谨，`time.After` 每次调用都会新建 Timer，若函数在热循环里调用会累积未到期计时器；三是失败分支也要往 channel 发或直接 return，否则两个都失败时 select 会等满超时。如果业务要求"多数派"或合并结果，可以收集到切片再统一判，但复杂度明显上升，需要权衡。

**延伸考点：** 如果两个上游返回的失败也要区分处理，怎么设计结果结构？select 的随机就绪选择在负载均衡里能怎么用？

---

### Q4. 外部调用要 3 秒超时、链路每层都要传递，context 怎么用？

**问题：** HTTP 接口内要调下游 RPC，链路要求：接口级 3 秒总超时，且中间任何一层取消都要传递下去，不能让下游调用"超时了还继续跑"，代码怎么组织？

**期望加分项：**
- 能从 handler 入口 `context.WithTimeout` 建根 ctx 并 defer cancel，贯穿整个调用链
- 知道 http.NewRequestWithContext、grpc 调用传 ctx 的用法
- 能谈超时语义：父取消自动传播给子，子 context 是父的派生
- 知道 ctx 传值的边界：只放请求元数据（trace_id、user_id），不放业务对象
- 能指出不 defer cancel 会泄漏计时器/资源，以及"ctx 存 struct"是反模式

**减分项：**
- 不会创建/派生 context，只会用 context.Background()
- 不知道 WithTimeout 返回的 cancel 必须调用（defer），泄漏了还一脸无辜
- 把业务对象塞进 context 传参
- 只说"加超时"说不出取消是怎么一路传导到真正阻塞的 goroutine 的

**解答：**
原则：context 从请求入口创建、沿调用链单向传递、在真正阻塞的 IO 点生效。入口处 `ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second); defer cancel()`，之后所有下游调用都传这个 ctx：HTTP 用 `http.NewRequestWithContext(ctx, ...)`，gRPC 直接传 ctx（Deadline 会随请求线序列化给服务端）。关键机制：父 ctx 一旦超时或取消，所有派生子 ctx 的 `Done()` 通道都会关闭，阻塞在 `select`/IO 调用上的 goroutine 会立刻返回——这就是"取消能穿透链路"的原因。三层实践要点：一是每层还可以按剩余时间派生更短的子超时（如 `context.WithTimeout(ctx, 1*time.Second)`），但要理解这是"更严格的子约束"；二是 context 传值只放请求级元数据（trace_id、用户态），放业务对象会导致函数难复用、难测试；三是永远不要把 ctx 存进 struct 当字段——ctx 是方法参数。坑：漏掉 `defer cancel()` 会让 WithTimeout 的计时器活到到期，高并发下积累大量计时器拖垮 runtime；ctx 取消只能打断"配合 ctx 的调用"，第三方库不认 ctx 时（如裸 net.Dial）仍会卡住，需要额外手段。

**延伸考点：** 服务端怎么感知客户端已断开（r.Context() 的 Done 何时关闭）？父取消后子 context 的值还能读到吗？

---

### Q5. 高频读取、低频更新的配置缓存，怎么做到读路径最快？

**问题：** 有一份配置（如黑白名单、开关），读 QPS 几十万、更新一天几次，需要保证并发安全且读路径尽量快，你会怎么实现？为什么？

**期望加分项：**
- 能给出 RWMutex + map 与 atomic.Value + COW（copy-on-write）两种方案并量化对比
- 知道 atomic.Value 的用法限制：存的对象类型必须一致、初次必须 Store 非 nil 值、读端拿快照
- 能说清 COW 的代价：每次更新全量拷贝，适合低频更新
- 能主动提一致性要求：业务可接受短暂读到旧配置才选无锁方案
- 提到用 benchmark 验证而非拍脑袋选型

**减分项：**
- 一律全局 Mutex，不知道读多写少的锁开销
- 滥用 atomic.Value，答不出它"类型一致""首次不能存 nil"的坑
- 不知道 RWMutex 写饥饿问题（Go 1.9+ 写优先，但仍可能被持续读拖住）
- 只给方案不解释为什么，没有取舍依据

**解答：**
先说判断：更新极低频（天级）、读极高频的场景，最优解是无锁读的 copy-on-write。实现：写路径深拷贝 map 做修改，然后 `atomic.Value.Store(newMap)` 整体替换；读路径 `v.Load()` 拿到快照引用后直接读 map，全程无锁。注意 atomic.Value 的坑：第一次 Store 前 Load 会返回 nil，初始化时就要先 `Store` 一次空 map；所有 Store 的类型必须完全一致（如都是 `map[string]bool`），混用类型会 panic；读端必须只读快照，不能在读的同时写同一份 map。备选方案是 `sync.RWMutex`：读用 RLock，写用 Lock——读多写少时争用很低，实现简单、心智负担小，绝大多数业务够用。分界线：如果更新频率上升到秒级或分钟级，COW 的拷贝成本就不划算了，此时退回 RWMutex 或改用分片。实践建议：先用 pprof/benchmark 确认这里确实是热点（几十万 QPS 未必需要极致优化），多数服务 RWMutex 即可，COW 是"确定热"之后再上的优化，别为不存在的问题提前设计。

**延伸考点：** atomic.Value 与 RWMutex 的分界怎么用数据判断？COW 更新期间读到旧数据是否符合你们的一致性要求？

---

### Q6. 错误到处 fmt.Errorf("%v") 包装，丢了类型信息，线上怎么改？

**问题：** 项目里错误处理很乱，到处都是 `fmt.Errorf("xxx: %v", err)`，上游用 errors.Is 判断失效、日志也看不出链路，线上排障困难，怎么系统性改进？

**期望加分项：**
- 能指出 %w 与 %v 的区别：%w 保留错误链使 errors.Is/As 可用
- 有分层策略：错误在边界处理（打印/转换），中间层包装传递，不重复打日志
- 能设计语义化错误：sentinel error（ErrNotFound）或带业务 code 的错误类型
- 日志用结构化字段（zap/logrus fields：trace_id、耗时、上游），不拼字符串
- 能谈错误日志的采样/限流，防止刷屏拖垮日志系统

**减分项：**
- 只知道 `err != nil` 判断，说不出 wrapping 的价值
- 每个函数都打印一遍错误，日志重复爆炸
- 不知道 errors.Is/As 与 %w 的关系
- 把 SQL、用户敏感信息打进错误日志

**解答：**
核心原则是"错误只在一个边界处理，中间层只包装传递"。改造分三层：第一层，统一包装语法，链路关键点用 `fmt.Errorf("调用 user rpc 失败: %w", err)`，%w 把原始错误包进链中，上层就能用 `errors.Is(err, ErrNotFound)` / `errors.As` 判断具体类型；不要再用 %v 裸拼。第二层，定义语义化错误：无参数区分用 sentinel error（`var ErrUserNotFound = errors.New("user not found")`）；需要携带上下文（code、消息、HTTP 状态）用自定义类型实现 `Error()` 与 `Unwrap()`。第三层，日志在边界打：最外层 handler/拦截器统一记录结构化日志（`zap.Error(err).Str("trace_id", tid)`），中间层不再 print；error 日志全量记录但加采样保护（如每 10s 最多 100 条同一错误摘要），info 日志按比例采样。实践坑：格式化里拼接敏感参数会泄漏用户数据；同一错误在 recover、返回、拦截器各打一遍会让日志膨胀三倍——打日志的位置要收敛；errors.Is 判断的对象必须是链上的 sentinel，判断前先确认包装用的是 %w。

**延伸考点：** 什么时候该 panic 而不是返回 error？错误码体系怎么设计才能同时服务 HTTP、gRPC 与前端？

---

### Q7. 每次发版都有请求被掐断，怎么实现优雅下线？

**问题：** 服务每次发布或扩容缩容，总有请求报连接被重置/超时，你们是怎么做优雅停服的？SIGTERM 之后发生了什么？

**期望加分项：**
- 能讲 signal.Notify 监听 SIGTERM/SIGINT 后的完整流程：摘流量 → 停止接收 → 排空在途 → 超时强杀
- 知道 http.Server.Shutdown 与 Close 的区别（Shutdown 等存量请求完成）
- 结合部署平台：K8s readiness 探针先摘流量、preStop 延迟，注册中心注销的时序
- 能覆盖 rpc server、db 连接池、消息消费端的关闭顺序
- 提到 Shutdown 带超时 context，超时后强制退出兜底

**减分项：**
- 只监听信号不排空在途请求
- 不知道 Shutdown 不等 WebSocket/长连接
- 答不出"摘流量"与"进程退出"之间的时序配合
- 用 os.Exit 直接退出

**解答：**
优雅下线的本质是"让新流量不再进来，把存量流量处理完再退出"。标准流程：`signal.Notify` 捕获 SIGTERM/SIGINT 后触发，依次执行：第一步摘流量——K8s 里 readiness 探针返回 false 让 Service 摘掉该 Pod，同时从注册中心注销、把负载均衡权重降零，这一步必须早于进程退出，否则新请求还会打进来；第二步停止接收新连接：`http.Server.Shutdown(ctx)`（带超时，如 10s），它关闭监听、拒绝新连接、等待在途请求完成；不要用 `Close()`，那会直接掐断存量连接；第三步关下游资源：先停止 rpc server、停止消费端拉取（但把已拉到的批内消息处理完）、再关 DB 连接池；第四步兜底：整体超时（如 30s）后仍没退干净，`os.Exit(1)` 强杀并告警，避免进程永不退出。实践坑：Shutdown 不处理 WebSocket 与 gRPC 流式连接，需要额外的 drain 机制；优雅退出期间要拒绝新请求并返回明确错误（503），而不是挂住；K8s 里 preStop hook 可以加 sleep 配合 LB 摘除；signal 处理要注册为幂等（二次 SIGTERM 直接强退）。

**延伸考点：** K8s 中 readiness 探针、preStop、SIGTERM 的时序怎么配合才不会丢流量？优雅退出期间进来的新请求怎么处理？

---

### Q8. CPU 核数很多但吞吐上不去，几十万 goroutine 压在上面，问题可能在哪？

**问题：** 服务跑在 32 核机器上，goroutine 数量几十万，但吞吐始终上不去、CPU 也没吃满，从调度器角度看可能是什么原因？怎么验证和调优？

**期望加分项：**
- 能讲 GMP 核心机制：M 与 P 绑定、P 本地队列+全局队列、work stealing、hand off
- 能分析容器场景：GOMAXPROCS 默认取宿主核数导致 P 过多（automaxprocs）
- 知道 syscall/锁阻塞会挂起 M 或让 goroutine 排队，需要新 M 或调度等待
- 能算量级：每 goroutine 栈起步 2KB，十万级就是 200MB+ 内存放大
- 会用 pprof/trace 观察 Runnable 排队与锁等待，而非凭空猜测

**减分项：**
- 只会背"GMP"三个字母，讲不出调度等待的成因
- 不知道容器里 GOMAXPROCS 的坑
- 答不出 goroutine 过多的量化代价（栈内存、调度器扫描成本）
- 推荐无脑调大 GOMAXPROCS

**解答：**
先建立判断框架：CPU 没吃满 + 延迟高 = 大量时间耗在"等待"上——等锁、等 channel、等 IO、等 P（调度排队）。用 `go tool trace` 看 goroutine 状态分布，用 pprof 的 block/mutex profile 看争用点。常见根因按概率排：第一，容器环境 GOMAXPROCS 问题——runtime 默认取宿主 CPU 核数，容器只有 4 核时却开了 32 个 P，调度器空转、抢占频繁，用 `github.com/uber-go/automaxprocs` 或显式 `runtime.GOMAXPROCS(N)` 修正；第二，goroutine 数量失控——每个 goroutine 初始栈 2KB，十万级内存放大，且调度器对大量可运行 goroutine 的维护成本上升，用信号量/worker pool 限制并发度；第三，阻塞型调用占着 M——锁、磁盘 IO、cgo、第三方库 syscall 会阻塞 M，runtime 需要拉起新 M 顶上，线程频繁创建销毁消耗 CPU，把阻塞调用异步化或批量处理；第四，P 被某个 goroutine 长时间占用（如 GOMAXPROCS 太小或死循环），其它 goroutine 在本地/全局队列排队。调优顺序：先修 GOMAXPROCS，再压并发度，再消除阻塞点，每一步用压测对比验证，别一次全改。

**延伸考点：** GOMAXPROCS=1 时 goroutine 还能并发吗？什么场景下调小 GOMAXPROCS 反而更快？

---

### Q9. 热点函数频繁分配对象、GC 压力大，怎么用逃逸分析优化？

**问题：** 一个被高频调用的函数里频繁 new 对象，压测发现 GC 频繁、分配量大，怎么确认哪些变量逃逸到堆了？有哪些优化手段？

**期望加分项：**
- 会用 `go build -gcflags="-m -l"` 查看逃逸分析结果
- 能列常见逃逸原因：返回局部变量指针、interface{} 装箱、闭包捕获、切片扩容
- 优化手段具体：值传递、预分配容量、避免 interface 装箱、sync.Pool 复用
- 有取舍意识：逃逸到堆不一定是坏事（大对象、跨函数生命周期），优化要看 alloc 热点
- 用 pprof alloc 定位后再动手，而非盲改

**减分项：**
- 不知道逃逸分析是什么，把锅全甩给 GC
- 乱改导致更差：把该逃逸的改成全局变量反而更糟
- 答不出编译器内联会影响逃逸判断
- 没有量化验证，改完不跑 benchmark

**解答：**
先定位再优化：第一步 `go build -gcflags="-m -l"` 看哪些变量 `escapes to heap`；更精确的是用 pprof 的 `alloc_objects`/`alloc_space` 找分配热点函数。常见逃逸原因与对策一一对应：函数返回局部变量指针（改为值返回或由调用方传入目标结构体指针）；参数/返回值是 interface{} 造成装箱（用具体类型或泛型）；闭包捕获外部变量（避免在循环里创建闭包）；`append` 扩容导致底层数组重新分配（`make([]T, 0, cap)` 预分配）。实操顺序：先做"零成本"优化——预分配、值传递、泛型替代 interface{}；仍不够再用 sync.Pool 复用热点小对象。三个坑：一是逃逸与否和编译器优化（内联）强相关，升级 Go 版本后结论可能变，别把优化建立在脆弱的逃逸结论上；二是 fmt.Sprintf、log 打印这类隐式 interface{} 调用必然逃逸，热路径上要避免；三是"逃逸到堆"不等于要消除——对象生命周期天然长于函数时堆上分配反而是正确选择，关键是减少"不必要的分配次数"。

**延伸考点：** fmt.Sprintf 为什么必然引起逃逸？热路径上避免日志格式化开销的做法是什么？

---

### Q10. 服务 RSS 3GB、GC 频繁导致 CPU 抖动，怎么用 GOGC/GOMEMLIMIT 调？

**问题：** 服务 RSS 稳定在 3GB（堆 2GB），GC 每几十秒触发一次，mark 阶段 CPU 抖动明显，你怎么调整 GC 参数？判断依据是什么？

**期望加分项：**
- 能讲 GOGC 语义：触发阈值 = 存活堆 × GOGC/100，调大减少 GC 次数、提高内存峰值
- 知道 Go 1.19+ 的 GOMEMLIMIT 软限制：接近上限才触发兜底 GC，不保证 RSS 一定不超
- 能给出 Go 1.21+ 推荐组合：GOGC=off + GOMEMLIMIT（平时不抖、逼近限制兜底）
- 容器场景：GOMEMLIMIT 设容器 limit 的 90% 左右，留出非堆内存余量
- 用 gctrace/监控（GC 次数、pause、RSS 曲线）验证调整效果

**减分项：**
- 只会说"调大 GOGC"，说不出和 CPU、内存的交换关系
- 不知道 GOMEMLIMIT 是软限制，以为设了就一定不 OOM
- 答不出 GC 内存延迟归还 OS、非堆内存（栈、cgo）也占 RSS 的边界
- 调完不看指标，无法验证

**解答：**
先明确交易：GC 是"CPU 换内存"——调大 GOGC，触发阈值变高、GC 次数变少、CPU 抖动下降，但堆峰值上涨，容器里可能 OOM；调小则相反。所以第一步看现状：用 `GODEBUG=gctrace=1` 或监控确认 GC 频率、每轮 mark 耗时、堆目标大小，判断是"GC 次数太多"还是"单次 GC 太慢"。第二步选择：如果内存有余量，`GOGC=200`（默认 100）这类保守放大即可；Go 1.21 起更推荐组合拳——`GOGC=off` + `GOMEMLIMIT=2.7GiB`（容器 limit 的 90%）：平时堆不主动 GC，消除周期性抖动，堆逼近软限制时 runtime 自动触发兜底 GC 防止超限，这在"延迟敏感、内存宽松"的服务上收益明显。三个坑：一是 GOMEMLIMIT 是软限制，Go 只保证尽力不超——堆外内存（goroutine 栈、cgo、syscall 分配）不受它约束，且内存归还 OS 有延迟，所以必须留 10% 余量，别贴满；二是 GOGC=off 配合 GOMEMLIMIT 适合长驻堆、分配稳定的服务，若分配速率极高，兜底 GC 反而频繁触发，得不偿失；三是改参数要小步验证：观察 24h 的 GC 次数、P99、RSS 曲线三者，任何一项劣化都要回退。

**延伸考点：** GOMEMLIMIT 设了之后 RSS 仍然超限，可能的原因有哪些？GOGC=off 的风险场景是什么？

---

### Q11. 线上服务内存持续增长但不回落，怎么用 pprof 定位泄漏？

**问题：** 线上服务 RSS 曲线持续爬升、从不回落，重启后恢复，再跑几天又涨上去。你怀疑是内存泄漏，用 pprof 排查的具体步骤是什么？

**期望加分项：**
- 能讲完整步骤：暴露 pprof（独立端口+鉴权）→ 间隔采样 → 对比两次 heap profile 的 diff
- 会看 inuse_space（当前占用）与 alloc_space（累计分配）的区别
- 能区分三类根因：goroutine 泄漏、全局缓存只写不清理、闭包/指针持有大对象
- 主动怀疑 time.NewTicker、channel、第三方 SDK 内部 goroutine 等常见点
- 知道 heap profile 是采样数据，小对象可能不显示；RSS 增长也可能是内存归还延迟

**减分项：**
- 只打开 pprof 页面不知道该看哪个视图
- 把"分配多"当"泄漏"，不区分 inuse 与 alloc
- 不知道 goroutine 泄漏最终也表现为内存增长，漏掉 goroutine 视图
- 不会做两次采样对比，只看一次快照下结论

**解答：**
定位链路分四步。第一步，暴露采样入口：`import _ "net/http/pprof"` 挂到独立端口并加 IP 白名单/鉴权（生产别裸奔在公网）。第二步，做时间序列对比而不是看单次快照：间隔 10 分钟分别执行 `curl .../debug/pprof/heap > heap1.out` 与 `heap2.out`，然后 `go tool pprof -diff_base=heap1.out heap2.out` 看 `inuse_space` 的 top——增长点就是泄漏嫌疑；对比 `goroutine` profile 同步确认是否 goroutine 泄漏（看 gopark 在 channel/select/timer 上的栈）。第三步，按嫌疑点对代码：全局 cache 只 Put 不淘汰、`time.NewTicker` 未 Stop、goroutine 闭包长期持有大切片/大 map、第三方 SDK（kafka/redis client）内部启动的 goroutine 或缓冲未释放。第四步，验证：修复后观察 RSS 曲线是否回落并趋于平台。三个坑：heap profile 是采样（默认每 512KB 记录一次），低频小对象可能漏，可以结合 `runtime.MemStats` 与 `-test.memprofile` 交叉验证；RSS 持续增长但 heap 平稳，可能是内存碎片、cgo 分配或栈增长未归还——别急着改业务代码；线上采样要控制频率（几分钟一次），采样本身有开销。

**延伸考点：** inuse_space 与 alloc_space 分别适合回答什么问题？怎么区分"真泄漏"与"GC 还没来得及回收"？

---

### Q12. 高 QPS 下频繁 new 大对象，sync.Pool 怎么用？有什么坑？

**问题：** 服务每秒处理上万请求，每个请求都要分配一个较大的临时缓冲（如 json 解码 buffer），GC 压力大。你想用 sync.Pool 复用，它适合吗？注意什么？

**期望加分项：**
- 能讲 sync.Pool 语义：Get 无可用时调 New、Put 归还、每次 GC 会清空 Pool——它不是缓存
- 适用边界：复用"同类型、无状态、生命周期短"的对象；有状态对象绝不入池
- 知道无法手动控制池容量，靠负载自适应
- 对象取出后必须 Reset/清零再放回，否则数据串扰
- 用 benchmark 验证收益，而不是无脑上 Pool

**减分项：**
- 以为 sync.Pool 能当 LRU 缓存用（GC 清空机制不懂）
- 把带状态对象（如连接、写了一半的 buffer）放进 Pool 导致脏数据
- 不知道 Pool 大小不可控
- 不先 profile 确认分配热点就上优化

**解答：**
sync.Pool 适合"昂贵、无状态、创建销毁频繁"的对象，典型是 json/protobuf 解码缓冲、字节切片、临时聚合结构。用法：

```go
var bufPool = sync.Pool{New: func() any { return &bytes.Buffer{} }}

func handle() {
    b := bufPool.Get().(*bytes.Buffer)
    b.Reset() // 必须：清掉上次的残留数据
    defer bufPool.Put(b)
    // 使用 b ...
}
```

三个关键坑：第一，每次 GC 都会清空 Pool 中的对象——sync.Pool 的设计目标是"平滑瞬时分配尖峰"，不是持久缓存，别指望命中率长期稳定，更不能把需要跨请求保留的数据放进去；第二，对象必须无状态或取出后彻底 Reset，否则上一个请求的数据会泄漏到下一个请求（曾出现真实事故：buffer 残留导致响应串数据）；第三，容量不可控——没有设置大小的 API，它通过 Victim/私有 slot 机制自适应，负载下降时对象自然被回收。选型判断：先跑 `pprof alloc_objects` 确认分配确实是热点；如果是大对象+高频率，Pool 收益明显；如果是小对象，Pool 的管理开销可能超过收益。Go 1.19+ 可用泛型封装减少类型断言样板，但别引入额外抽象负担。连接类资源不要用 sync.Pool，用专门的连接池库（它需要管理生命周期与健康检查）。

**延伸考点：** 为什么 sync.Pool 不能替代本地缓存？GC 清空 Pool 的机制（victim cache）具体是怎样的？

---

### Q13. 下游只能扛 100 QPS，上游突发 1 万 QPS，怎么保护下游？

**问题：** 你们服务要调用一个第三方接口，它只能承受约 100 QPS，而你们上游流量高峰能到 1 万 QPS，怎么设计这层保护？如果下游开始大量报错，又该怎么办？

**期望加分项：**
- 能分层：出口限流（x/time/rate 令牌桶）+ 并发控制（信号量/worker pool）+ 熔断 + 重试退避
- 能说清限流与并发数的区别，以及排队 vs 快速失败的取舍（延迟敏感选拒绝+降级）
- 知道熔断状态机（closed/open/half-open）与半开探测的实现思路
- 重试必须指数退避+抖动，防重试风暴雪崩
- 限流参数从压测/下游实际吞吐得出，不拍脑袋

**减分项：**
- 只会说"加队列"，队列无限膨胀照样打爆下游和内存
- 限流熔断混淆，说不清各自解决什么问题
- 重试无退避、无抖动，下游挂了之后重试风暴把服务打死
- 拒绝策略缺失：不知道超限请求怎么处理（丢弃/排队/降级返回）

**解答：**
这是经典的"保护下游"问题，答案是层层设防。第一层限流：用 `golang.org/x/time/rate` 令牌桶 `rate.NewLimiter(rate.Limit(100), 20)`，调用下游前 `limiter.Wait(ctx)`（排队）或 `Allow()`（快速失败）——选择依据是业务对延迟的容忍度：可接受排队用 Wait + 队列上限，延迟敏感用快速失败 + 降级返回兜底数据。第二层并发控制：用 buffered channel 做信号量限制同时在途调用数（如 50），防止慢下游拖起一堆 goroutine。第三层熔断：监控最近 N 秒的错误率，超过阈值（如 50%）打开熔断，后续请求直接快速失败不再打下游，经过冷却期进入 half-open 放少量探测流量，成功率恢复则关闭——这是防雪崩的关键，很多公司直接引入 go-zero 的 breaker 或 Hystrix 类库。第四层重试：只对幂等请求重试，指数退避 + 随机抖动（如 `min(2^n, 10s) + rand`），避免所有客户端同时重试形成"重试风暴"。参数来源：对下游做压测得到真实吞吐与 P99，限流值=吞吐×安全系数（如 0.8）。实践坑：限流只保护"调用量"，熔断保护"失败传染"，两者都要；下游恢复后要观察半开探测流量，别一次全量放开。

**延伸考点：** 令牌桶与漏桶的区别？熔断器半开状态怎么选探测流量、怎么决定恢复阈值？

---

### Q14. 线上大量 TIME_WAIT、客户端报连接被重置，http 服务端怎么排查优化？

**问题：** 线上服务最近出现大量 TIME_WAIT 连接，部分客户端报"connection reset"，从 http.Server 和 http.Client 两侧分别怎么排查和优化？

**期望加分项：**
- 服务端：知道 ReadTimeout/WriteTimeout/IdleTimeout/ReadHeaderTimeout 默认 0（无超时）的风险与 Slowloris 攻击
- 客户端：http.Client 要复用单例，讲 MaxIdleConnsPerHost、MaxIdleConns 与 keep-alive
- 能解释 TIME_WAIT 成因：短连接过多（每次 new client）或服务端主动关空闲连接
- 中间件实践：recover、统一超时（context.WithTimeout 包 handler）、结构化 access log、限流
- 知道 handler 并发写 ResponseWriter 会 panic/数据错乱

**减分项：**
- 只会调内核参数（tcp_tw_reuse）治标不治本
- 不知道 http.Server 超时参数默认 0 的含义
- 客户端每请求 new http.Client 不知道复用
- 不知道连接被重置常见于服务端超时或 handler 超时中断写响应

**解答：**
先明确排查方向：TIME_WAIT 大量出现几乎都是"短连接"——要么客户端每次请求都 `http.Client{}` 新建（连接不复用，用完即关），要么服务端主动关闭空闲/超时连接。服务端侧：把 `ReadTimeout`、`WriteTimeout`、`IdleTimeout`、`ReadHeaderTimeout` 都设置上（默认 0 是永不超时，慢速连接能挂死 handler goroutine，即 Slowloris 攻击面）；MaxHeaderBytes 限制头部大小；keep-alive 保留给高频客户端。客户端侧：http.Client 设为包级单例复用，`Transport` 里调大 `MaxIdleConnsPerHost`（默认 2，高并发下会不停建新连接造成 TIME_WAIT 暴涨）与 `MaxIdleConns`，开启 keep-alive。连接被重置的常见根因：服务端 WriteTimeout/ReadTimeout 触发中断、handler 内部 context 超时后仍在写、中间件在写响应后继续操作。中间件规范：最外层 recover（防 panic 裸奔导致连接异常关闭）、用 `context.WithTimeout` 包 handler 统一超时、结构化 access log 记录状态码与耗时、请求级限流。坑：ResponseWriter 只能由持有它的 goroutine 写，并发写会 panic 或响应错乱；中间件要在写响应前完成所有检查。

**延伸考点：** ReadTimeout 与 ReadHeaderTimeout 分别防什么？客户端复用连接时为什么还会出现新建连接？

---

### Q15. 服务 A 调服务 B（gRPC）偶发超时失败，怎么排查和加固？

**问题：** 服务 A 通过 gRPC 调服务 B，线上偶发超时/失败，错误码五花八门。从客户端、服务端、部署三个角度，你会怎么定位和加固？

**期望加分项：**
- 定位思路：先分耗时（网络往返 vs 服务端处理 vs 排队等待），用 trace/日志量化
- 客户端：合理 Dial 超时、调用超时（ctx deadline 传递）、幂等接口才配重试、指数退避
- 服务端：拦截器统一超时与日志、限流防被拖垮、用 codes 语义化错误（NotFound/DeadlineExceeded）
- 部署层：知道 gRPC 长连接在 K8s 下负载不均的问题（连接粘在少数 Pod），需要 client-side LB/headless service
- 能谈 gRPC 的 deadline 会序列化传给服务端，服务端感知 client 取消（ctx.Done）

**减分项：**
- 只会说"加超时"；盲目对所有接口重试（非幂等造成重复扣款）
- 不知道 K8s 多副本下 gRPC 长连接负载不均的坑
- 所有错误都映射成 Internal，错误码没有语义
- 不在客户端看 P99 曲线就下结论

**解答：**
定位先分层：客户端日志或 trace 里看每次调用的耗时拆解——是建连慢、排队等线程、还是服务端处理久、或是网络重传。用 grpc 的拦截器（client 端和 server 端都加）统一记录 latency、code、元数据，问题就藏在"偶发"的模式里：如果错误集中在流量高峰，多半是服务端限流或排队；如果集中在部署发布期，多半是摘流不及时。加固分三层：客户端——设置合理的 Dial 超时（建连失败快速失败而非阻塞）、每次调用带 `context.WithTimeout` 的 deadline、只对幂等接口重试（重试次数 2-3 次 + 指数退避，非幂等接口只告警）；服务端——拦截器统一超时处理、错误码用 gRPC codes 语义化（NotFound、DeadlineExceeded、Unavailable 各归各），错误消息脱敏；部署——这是最容易踩的坑：gRPC 长连接建立后固定不换，K8s 里 DNS 轮询只发生在建连时，流量会粘在少数 Pod 上，需要 client-side LB（grpc-go 的 resolver/balancer）或通过 headless service + 自研 resolver 实现按连接均衡，否则部分 Pod 打满、部分空闲。验证：压测复现"偶发"，看两个服务的 CPU/连接数分布与错误率曲线是否对齐。

**延伸考点：** gRPC 的 deadline 怎么传递到服务端？服务端如何感知客户端已取消并停止无用计算？

---

### Q16. 两个 goroutine 一个读一个写同一变量，偶发读到旧值，怎么检测修复？

**问题：** 代码里两个 goroutine 共享一个 bool 标志，一个写一个读，偶发读到旧值导致行为异常。怎么证明是数据竞争？怎么修？

**期望加分项：**
- 会用 `go test -race` / `go run -race` 检测，知道 race 是 happens-before 分析、没检出≠没有
- 修复按场景选：原子操作（atomic.Bool/atomic.Int64）、锁、channel 传递数据
- 能讲 atomic 的可见性语义：保证原子性与内存序，但不是"读到的总是最新值"的魔法
- 知道 i++ 这类读-改-写复合操作 atomic 单条不够，需要 CAS 循环或锁
- 修复后用 race 跑满测试 + 压测验证清零

**减分项：**
- 只会说"加个锁"，说不出为什么 atomic 可能更合适
- 以为 -race 没报就安全
- 不知道 atomic 与锁的本质差异（内存序 vs 互斥）
- 修复后不验证，靠"感觉"

**解答：**
第一步证明：`go test -race ./...` 或线上加 `-race` 临时跑一版（注意 2-10 倍性能开销，只能测试环境/灰度），报告会给出冲突的读写行号。原理：race detector 基于 happens-before 模型，没检测到不代表无竞争——goroutine 内同步路径外的读写它才报。第二步修复，按语义选工具：纯计数器/标志位用 `sync/atomic`（`atomic.Bool`、`atomic.Int64`），读写都是原子操作，且 atomic 带内存序保证可见性；需要"读-改-写"复合操作（如 `i++`、CAS 前置判断）用 `atomic.CompareAndSwap` 循环或 Mutex；数据整体传递用 channel（channel 收发建立 happens-before 边界，是 Go 推荐的通信方式）。第三步设计层面：并发设计原则是"能别共享就别共享"——把可变状态收进单一 goroutine，其它 goroutine 通过 channel 请求操作，杜绝竞争。三个坑：atomic 只保证单条指令原子，不保证复合逻辑原子；无锁读可能读到"旧但合法"的值，业务必须容忍或配版本号；race 检测会改变时序，线上偶发的问题在 -race 下更容易暴露，这是特性不是 bug。

**延伸考点：** atomic 与 Mutex 在内存序语义上的差别是什么？什么场景下无锁读（可能读旧值）是可接受的？

---

### Q17. 用 time.NewTicker 做定时上报，线上内存和 goroutine 都在涨，为什么？

**问题：** 服务里用 `time.NewTicker(5 * time.Second)` 启动了一个定时上报 goroutine，跑几天后内存和 goroutine 数量持续增长，重启才好。问题可能出在哪？

**期望加分项：**
- 知道 Ticker 必须 Stop：不 Stop 则底层 timer 常驻 runtime 计时器堆，goroutine 永不退出
- 能看出"后台 goroutine 没随服务生命周期管理"的设计问题：应结合 ctx.Done() 退出
- 知道 time.After 在循环里的累积问题（每次分配 Timer），应复用 time.NewTimer + Reset
- 会用 pprof goroutine 看到 wait on time.Timer 的栈来定位
- 知道 Ticker.Stop 后 t.C 不会自动关闭，读端要配合 select 退出

**减分项：**
- 不知道 ticker 要 Stop
- 在 for 循环里反复 time.After / NewTimer 不 Stop
- 答不出 Stop 与 channel 的关系（Stop 不关闭 channel）
- 后台任务没有统一的生命周期管理范式

**解答：**
根因几乎可以锁定：定时器未正确释放。`time.NewTicker` 创建的 ticker 不调用 `Stop()`，它的底层 timer 会一直挂在 runtime 的定时器堆里，并且阻塞在 `t.C` 上的 goroutine 永远读不到退出信号——于是每启动一个就泄漏一个定时器 + 一个 goroutine，内存和 goroutine 双双增长。规范写法：

```go
ctx, cancel := context.WithCancel(parentCtx)
defer cancel() // 服务停时触发
ticker := time.NewTicker(5 * time.Second)
defer ticker.Stop()
for {
    select {
    case <-ticker.C:
        report()
    case <-ctx.Done():
        return // 优雅退出
    }
}
```

配套的坑：第一，`time.After(d)` 每次调用都新建 Timer，若放在循环里且 d 较长，未到期的 Timer 会累积——改用 `time.NewTimer` + `Reset`（Reset 前先 Stop + drain channel）；第二，`Ticker.Stop()` 不会关闭 `t.C`，所以读端必须依赖 select 的退出分支，不能期望 channel 关闭；第三，`time.AfterFunc` 返回的 Timer 可以 Stop 取消回调，别漏。更根本的改进是统一生命周期管理：所有后台任务接收 ctx、由主程序统一 cancel 并 WaitGroup 等待退出，避免每个功能各自为政。定位手段：pprof goroutine 栈里能看到 `time.Timer` 相关的 gopark，配合代码审查即可确认。

**延伸考点：** Ticker.Stop 之后 t.C 还能收到值吗？后台任务的生命周期管理有哪些通用范式？

---

### Q18. 线上 P99 偶发毛刺、CPU 不高内存正常，怎么用 go tool trace 分析？

**问题：** 服务 CPU 和内存都正常，但 P99 延迟偶发飙升到几秒。你用 go tool trace 分析的具体步骤是什么？重点看什么？

**期望加分项：**
- 会开 runtime/trace（或按需采样），知道 trace 看"等待"、pprof CPU 看"执行"，两者互补
- 能根据 goroutine 状态归类：Runnable（排队等 P）、SyncBlock（等锁/等 channel）、Syscall（阻塞 IO）
- 会展开毛刺时间窗口看 goroutine 时间线，找"卡在哪一步"
- 能结合 block/mutex profile 确认争用点
- 知道 trace 有开销不能全量常开

**减分项：**
- 只做 CPU profile，看不到调度等待类问题
- 不会看 trace 的 goroutine 分析视图，只会看火焰图
- 答不出"CPU 不高但延迟高 = 在等"的核心推断
- 把 GC 停顿当唯一嫌疑，不会用 trace 区分 GC 与调度等待

**解答：**
CPU 不高但延迟高，推断只有一个：goroutine 在"等"——等锁、等 channel、等 IO、等 P。分析链路：第一步开启采样，代码里 `runtime/trace.Start(f)` 对目标接口采样 30-60s，或按需注入（trace 有 20%+ 开销，不能全量常开，配合 pprof 的 block 采样更经济）。第二步用 `go tool trace trace.out` 打开分析器，重点看 Goroutine analysis 与时间线：切到毛刺发生的时段，把出问题的 goroutine 状态归类——`Runnable` 说明它在排队等 P（调度瓶颈：GOMAXPROCS 不足、有 goroutine 长占 P），`SyncBlock` 说明在等 channel/锁（用 `pprof block`、`pprof mutex` 定位具体锁），`Syscall` 说明在等系统调用（磁盘、网络慢）。第三步对因施治：等锁→锁粒度拆分、改用 RWMutex/分片；等 channel→扩缓冲、批量消费、检查生产者是否被拖死；等 P→检查是否有 goroutine 死循环占满 P、调 GOMAXPROCS；等 IO→检查下游或磁盘（往往同时能看到 syscall 时间异常）。坑：trace 文件可能很大（几百 MB），采样窗口别太长；偶发问题要"抓现场"——平时常开轻量采样（如每 10s 采 1s），问题复现时已有数据；不要只看默认视图，Goroutine analysis 的 waiting 时长排序是最快的切入点。

**延伸考点：** Runnable 等待与 Syscall 阻塞对调度的不同影响？怎么找出"谁长期占着 P 不放"？

---

### Q19. 依赖的第三方库出安全漏洞，快速升级却编译失败/行为变化，怎么处理？

**问题：** 你们依赖的某个三方库爆出安全漏洞，需要紧急升级，但升级后发现编译不过、或行为变化影响业务。你怎么操作和决策？

**期望加分项：**
- 能梳理依赖影响面：go mod graph 看直接/传递依赖，go list -m -u 查版本
- 掌握升级命令与流程：go get pkg@version、go mod tidy、go.sum 一致性
- 知道 major 版本（v2+）路径带 /v2 后缀，语义化版本断裂时的处理
- 会用 replace 做临时手段（本地 fork/修复），但知道要注明原因、避免长期
- 能讲 vendor 与 GOPROXY 的作用，以及 go.sum 校验失败的含义

**减分项：**
- 只会 go mod tidy，不知道它在某些情况下会把依赖"升级"得更乱
- 乱用 replace 指向私人 fork 且不留注释，后续无人能维护
- 升级失败就回滚，不做根因分析
- 不知道 go.sum 校验机制，出错时直接删掉校验项

**解答：**
流程分四步。第一步评估影响面：`go list -m all | grep pkg` 确认它是直接依赖还是传递依赖，`go mod graph | grep pkg` 看谁引用了它；如果只是传递依赖，升级引入方版本即可，不用动自己代码。第二步升级：`go get example.com/lib@v1.2.3`，然后 `go mod tidy` 收敛；编译失败先看错误：API 变更查 CHANGELOG 和迁移文档，major 版本升级（v1→v2）路径会变成 `example.com/lib/v2`，import 路径、package 名都要同步改，这一步最容易漏。第三步兜底策略：上游修复慢或不发版，用 `replace example.com/lib => example.com/fork v1.2.4` 指向自研 fork 或本地 patch，但 go.mod 里必须注释原因、跟踪上游版本、设清理 deadline，防止 replace 长期挂着变成技术债；如果公司要求审计依赖，切 `vendor` 模式（-mod=vendor）把依赖源码入库，升级后要同步 `go mod vendor` 更新目录。第四步验证：升级后全量测试 + 回归压测（三方库行为变化常表现在超时、重试、日志格式上），并复查 `go.sum`——校验失败说明代理或源码与锁定内容不一致，不要 `go mod tidy` 一把梭删掉校验，要查代理配置（GOPROXY）与是否被篡改。坑：`go get` 会同时升级传递依赖造成连锁变更，升级前先 `go mod download` 锁好基线；团队要锁 Go 工具链版本（go.mod 的 toolchain 字段），否则 module graph pruning 行为在不同版本间不一致。

**延伸考点：** replace 的常见误用和审计风险？私有 GitLab 依赖的 GOPROXY 与认证怎么配？

---

### Q20. 高并发高一致要求的服务单实例扛不住，怎么架构演进？（开放性）

**问题：** 一个服务读写 QPS 都很高、业务要求强一致，单实例性能不达标。你会怎么设计方案让它扩展？请给出你的分析框架和取舍。

**期望加分项：**
- 先问需求再谈方案：一致性等级（线性一致 vs 最终一致）、读写比例、数据量、延迟预算、成本
- 分情况给方案：读多写少走缓存（含一致性方案）、写多走分库分表/消息队列削峰、强一致走单写多读或共识
- 能谈缓存一致性：先写库后删缓存、版本号对比、双删、binlog 订阅
- 有容量规划意识：按 QPS×单请求成本估算实例数与资源，压测验证
- 主动谈故障预案：限流、降级、熔断、多活；承认方案都有 trade-off

**减分项：**
- 一上来就"上 Redis、上 Kafka"，不先问约束
- 不考虑一致性，缓存导致脏数据还振振有词
- 答不出读写分离的延迟窗口与一致性问题
- 方案没有量化依据（不说 QPS、数据量、成本），或拒绝承认代价

**解答：**
这是开放题，考察的是"解题框架"而非标准答案。第一步明确约束：一致性到底要什么（业务上能接受 100ms 的读延迟吗？能接受短暂读到旧值吗？）、读写比例、数据量级、P99 预算、成本上限——很多"强一致"诉求在追问后发现其实是最终一致可接受，方案复杂度完全不同。第二步按读写特征分流：读多写少：加缓存层（Redis 或本地缓存），重点解决缓存一致性——先更新库再删缓存（配合版本号对比兜底）、双删防旧缓存回填、低频热点可用订阅 binlog 同步，同时评估"读旧值窗口"是否在可接受范围；写多：分库分表（一致性哈希 + 虚节点，按业务键水平拆分）摊薄写压力，或消息队列削峰（写路径异步化，但要重新设计一致性与补偿）；强一致要求高：任何多副本方案都有延迟成本，要么单写多读 + 读延迟可接受，要么引入分布式共识组件（etcd/自研 quorum），要么接受"主库单点写 + 同步复制"。第三步容量规划：QPS × 单请求 CPU/内存成本 → 实例数，压测验证吞吐与 P99 是否达标，规划留 50% 余量应对高峰。第四步兜底：限流、熔断、降级、容量告警，以及发布/故障时的降级预案。全程的考察点：你是否会追问约束、是否能量化、是否说得出每个方案的代价，而不是背一套"缓存+队列+分库"的组合拳。

**延伸考点：** "先删缓存再更新库"和"先更新库再删缓存"各自的坑与正确姿势？如果要你给这个方案做容量估算，你会怎么算？

---
