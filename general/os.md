# 通用 · 操作系统（面试题库）

本文件面向后端、运维与 SRE 岗位，考察候选人在操作系统工程实践中的真实能力：进程线程与并发、内存与虚拟内存、I/O 模型与磁盘性能、文件系统与容器隔离、系统启动与故障排查。不考八股文定义，而是以线上事故和性能调优场景切入，重点看候选人能否用工具链定位问题、用量化口径做判断、用取舍思维做方案选型，并沉淀出可复用的排查方法论。

### Q1. 为什么说"线程不是越多越好"？你的服务线程数是怎么定的？

**问题：** 你们的服务并发上去了，有人提议把线程池线程数从 200 调到 1000 来提升吞吐，你同意吗？给出你的依据。

**期望加分项：**
- 能区分 CPU 密集与 I/O 密集场景，给出推导逻辑（如 CPU 密集 ≈ 核数+1；I/O 密集结合等待时间/计算时间估算），而不是拍脑袋给数字
- 能指出线程过多的代价：栈内存占用、上下文切换开销，最好有量化证据（vmstat 的 cs 列、perf 统计的 context-switches）
- 能联系线上实践：自己调过线程池、做过压测，或处理过"线程一多吞吐反而下降"的现象
- 主动考虑边界：瓶颈可能在锁竞争、下游连接池或 socket 缓冲，而非线程数本身

**减分项：**
- 只会背"上下文切换开销大"却没有数据或案例支撑
- 给一个拍脑袋的数字，说不清推导过程
- 忽略业务类型差异（计算型 vs 等待型）
- 全程没有提到任何线上调优或排查经历

**解答：**
先给判断：线程数不是越多越好，超过某个临界点后吞吐反而下降，这是典型的扩展性陷阱。核心逻辑：每个线程要占栈空间（默认约 8MB 虚拟内存，按需提交），就绪线程一多，CPU 大量时间花在上下文切换上——保存/恢复寄存器、切换内核栈、TLB 失效，这些开销都是实打实的。判断口径：CPU 密集任务线程数取"核数+1"即可；I/O 密集任务的理论公式是 线程数 ≈ 核数 ×（1 + 等待时间/计算时间），但实际更可靠的做法是压测找拐点。排查手段：`vmstat 1` 观察 `cs`（上下文切换）列，当 CPU 使用率不高但 cs 每秒几十万时，多半是线程过多；`pidstat -w -p <pid> 1` 看单进程的 voluntary/involuntary context switches；Java 服务可结合线程池 activeCount 与队列堆积情况判断。实践坑：很多服务的真实瓶颈在下游连接池或锁竞争，盲目调大线程数只会放大排队效应；Go 的 goroutine 虽轻（初始 2KB 栈），但百万级 goroutine 带来的调度和内存占用同样不可忽视，不能因为有协程就无脑开。

**延伸考点：** 协程相比线程到底省了哪些开销？上下文切换成本高，具体高在哪些环节（内核/用户态切换、TLB 失效）？

### Q2. 同机两个进程要持续交换 MB 级数据，共享内存、消息队列还是 Socket？

**问题：** 需要 A 进程持续把一批批 MB 级数据交给同一台机器上的 B 进程处理，且延迟敏感。IPC 方案上你会怎么选？

**期望加分项：**
- 能说清各 IPC 方式的性能量级与取舍：共享内存（无内核拷贝、微秒级）＞ Unix Domain Socket（经内核但无协议栈开销）＞ TCP loopback；管道/消息队列适合小数据
- 能指出共享内存的同步问题（需要信号量/自旋锁配合），以及 mmap 的缺页与残留风险
- 能联系实践：kafka/rocketmq 的 mmap 索引、nginx 共享内存缓存等真实选型
- 主动考虑边界：同机还是跨机、可靠性要求、是否需要消息不丢失

**减分项：**
- 罗列了一堆 IPC 名词但说不清各自适用场景
- 完全忽略同步与并发安全问题
- 把 IPC 和网络通信混为一谈，不考虑同机/跨机的本质差异
- 无实践佐证，纯背教科书分类

**解答：**
先按两个维度切分：数据规模和可靠性要求。同机、MB 级、延迟敏感，首选共享内存（shm / mmap）——本质是映射同一块物理内存，省掉全部内核拷贝，微秒级延迟，但必须用信号量或自旋锁做同步，并处理好写者崩溃留下脏数据的恢复（信号量与共享内存的分配顺序要固定，否则崩了之后无法清理，只能 ipcrm）。次选 Unix Domain Socket，走内核但比 TCP 少协议栈开销，代码简单、天然可靠。而管道和 System V 消息队列单条消息上限约 8KB，不适合大块数据；TCP loopback 要过完整协议栈，延迟明显更高。工程上常见组合是"mmap + 无锁环形队列"（RocketMQ 的索引、nginx 共享内存缓存都是这个路子）。实践坑：mmap 首次访问会触发缺页有瞬时延迟；大页（HugeTLB）能减少 TLB miss，但对配置要求高。一旦需求变成跨机器，或要求消息绝对不丢、可回溯，结论立刻反转——退回到 Kafka/RocketMQ 这类持久化消息系统，共享内存方案失效。

**延伸考点：** mmap 相比 read/write 为什么快？零拷贝（sendfile）省掉的到底是哪几次拷贝？

### Q3. 服务发布后端口起不来，报 Address already in use，但 lsof 又看不到进程监听，怎么处理？

**问题：** 线上服务发布重启失败，日志报 bind: Address already in use，可 `lsof -i :<port>` 又看不到 LISTEN 的进程，怎么定位和解决？

**期望加分项：**
- 能区分两种根因：端口真被其他进程占用，vs 大量 TIME_WAIT 连接占着四元组导致 bind 失败
- 能说清 SO_REUSEADDR 的语义（监听端在 TIME_WAIT 时可立即 bind）及 Java/Go 中默认开启情况
- 能提到 tcp_tw_reuse 的适用前提（仅对发起连接方有效）及其争议，不乱调 tcp_tw_recycle
- 联系容器发布场景：旧实例未完全退出、K8s 新老 Pod socket 冲突
- 给出临时方案与根治方案两层答案

**减分项：**
- 只会"kill -9 进程"或"换端口"的粗暴处理
- 分不清 TIME_WAIT 与 ESTABLISHED 对 bind 的影响
- 乱调 tcp_tw_reuse/tcp_tw_recycle，讲不清前提与风险
- 不查是谁占的端口（监控 agent、其他实例、残留进程）

**解答：**
先判断方向：`ss -ltnp | grep <port>` 看是否有进程处于 LISTEN。如果没有 LISTEN 但有一堆 TIME_WAIT，说明是老连接在 TIME_WAIT（默认 60s）期间占着四元组，导致新的 bind 失败——此时给监听 socket 加 SO_REUSEADDR 即可（Java 默认开启，Go 的 net.Listen 在 Linux 上也默认开启）。同时检查发布流程：旧进程是否真的退出（kill 后没等退完就启动新的，或容器旧实例还在 graceful shutdown），K8s 场景常见"旧 Pod 的 socket 未释放、新 Pod 已启动"的冲突。如果确认是 TIME_WAIT 大量堆积，再考虑 `net.ipv4.tcp_tw_reuse=1`——注意它只对"发起连接的一方"生效，NAT 环境存在序列号复用丢包风险；tcp_tw_recycle 在 NAT 下丢包严重且新内核已移除。根治方向是减少短连接：用连接池、长连接复用，而不是一味调内核。实践坑：K8s 里改 sysctl 需要容器特权与 kubelet 配置，镜像里写 /etc/sysctl.conf 常常不生效；改参数要说明生效范围，别在无依据时全机器改。

**延伸考点：** TIME_WAIT 存在的意义是什么（防止旧报文串扰新连接）？2MSL 里的 MSL 具体指什么？

### Q4. 服务"假死"，接口超时，load 不高但线程大量 BLOCKED，怎么定位是哪个锁、谁持锁不放？

**问题：** Java 服务接口偶发超时，`jstack` 一看大量线程处于 BLOCKED（waiting to lock），怎么找到锁目标、谁持有锁、卡在哪？

**期望加分项：**
- 会读 jstack：BLOCKED 线程栈里的 `- waiting to lock <0x...>` 地址就是锁对象，全局搜该地址即找到持锁线程
- 知道要多抓几次 dump（间隔 3-5 秒、连抓 5 次左右）对比，区分瞬时抖动与持续持锁
- 会用工具补充：Arthas `thread -n 3` / `thread -b`、jstack -l、async-profiler
- 主动考虑隐蔽持锁点：DB 连接池获取、日志锁、静态初始化、JDK 内部同步
- 有修复到验证的闭环（锁粒度、临界区范围）

**减分项：**
- 只回答"用 jstack 看"但说不清具体怎么看
- 单次 dump 就下结论，把秒级等待误判成事故
- 只盯业务锁，忽略连接池、文件锁等底层资源
- 没有修复方案和验证步骤

**解答：**
定位分三步：确认现象 → 找锁 → 找持锁方。先连抓几次线程 dump（间隔几秒），确认 BLOCKED 线程不是瞬时抖动。dump 里 BLOCKED 线程的栈顶通常在 `LockSupport.park` 或 synchronized 处，下方会有 `- waiting to lock <0x…>`（monitor 锁）或 `parking to wait for <0x…>`，这个十六进制地址就是锁目标。全局搜索该地址，就能找到持有它的线程栈，看它卡在哪：最常见的是等 DB 连接、等下游 HTTP 响应、死循环，甚至 `Long.toString` 这类 JDK 内部同步点。补充手段：Arthas `thread -n 3` 直接列出最忙线程，`thread -b` 主动找死锁；`jstack -l` 打印更完整锁信息。修复方向：把持锁期间的重活移出临界区、锁粒度细化、用读写锁或 CAS 无锁结构；如果卡在连接池，往往是池大小配错或下游响应慢传导。实践坑：单次 dump 看到 BLOCKED 就定性，会把正常的秒级锁等待误判成死锁事故；定位到锁后还要看持锁时长趋势，别把偶发当成常态。

**延伸考点：** 怎么确认是系统性死锁（线程互相持锁等待）？jstack 出现 "Found one Java-level deadlock" 说明什么，怎么结合 dump 还原死锁环？

### Q5. 进程 RSS 一直在涨但没崩、GC 也正常，怎么判断是内存泄漏还是正常增长？

**问题：** Java 服务 RSS 从 2G 涨到 8G，没有 OOM，GC 也正常，你怀疑内存泄漏，怎么一步步确认？

**期望加分项：**
- 先看趋势不看快照：涨到水位后稳定是缓存/池化正常扩张，持续线性上涨才是泄漏
- 能分口径排查：堆（jstat -gc）、堆外（DirectBuffer/线程栈/Metaspace）、page cache 别混淆
- Java 会用 jmap 抓 dump、MAT 看 Dominator Tree 找 GC root 引用链；Go 用 pprof
- 会用 pmap 看 RSS 的地址段分布，直接判断哪块内存大
- 联系线上：比对发布记录、上线时间点

**减分项：**
- 不看堆内/堆外就把锅全甩给"内存泄漏"
- 只拍一张 RSS 快照就下结论，没有趋势数据
- 只答"调大内存"或"重启缓解"，没有定位动作
- 分不清 page cache 与进程私有内存，把系统缓存算进泄漏

**解答：**
先纠正口径再动手。第一步看趋势：RSS 涨到一定水位就稳定，大概率是缓存、连接池、线程池的正常扩张；无上限持续上涨才是泄漏。第二步分口径：Java 用 `jstat -gc <pid>` 看堆使用率，堆稳定而 RSS 涨，重点查堆外——DirectByteBuffer（Netty 场景高频）、线程数（每线程约 1MB 栈）、Metaspace、mmap 文件映射；`pmap <pid>` 看 RSS 的地址段分布，能直观定位哪段内存大（堆、匿名映射、文件映射）。第三步确证：Java 抓 heap dump（生产建议配合重启窗口，jmap 长时挂起有风险，优先 Arthas/jcmd），用 MAT 的 Dominator Tree 找大对象及其 GC root 引用链，确认是谁在引用、为什么收不掉；Go 用 pprof 对比两个时间点快照。实践坑：RSS 高不等于泄漏——JVM 堆缩容后内存不一定立即归还 OS（glibc arena 缓存、JVM 不主动返还），RSS 稳定在高位是常态；判断泄漏的硬标准是"持续增长 + GC 无法回收 + dump 里找到强引用链"。

**延伸考点：** glibc 的 malloc 为什么释放后不立即把内存还给 OS？这对容器内存上限（cgroup limit）有什么影响？

### Q6. 容器里应用被 OOM kill，但进容器看 free 内存还有剩余，怎么回事？

**问题：** K8s 里容器频繁重启，事件显示 OOMKilled，但进容器执行 free 看还有几个 G 空闲，为什么？

**期望加分项：**
- 能说出 OOM 判定依据不是容器内 free，而是 cgroup 内存账本（memory.current vs memory.max）
- 分清两层 OOM：内核全局 OOM 与 cgroup 级 OOM，容器场景命中后者
- 关键点：page cache 计入 cgroup 内存额度，容器读写文件/日志都会占额度
- 给出排查命令：memory.max / memory.current / memory.stat、dmesg 里 "Memory cgroup out of memory"
- 给出修复方向：limits 设置、cache 回收、应用层优化

**减分项：**
- 答"内存真的不够了"却不解释 cgroup 口径
- 分不清宿主机内存与容器内存
- 不看 dmesg 和 cgroup 事件就下结论
- 没有调整建议输出

**解答：**
核心认知：容器的内存限制由 cgroup 管控，OOM 判定的口径是 cgroup 内 memory.current 是否超过 memory.max，而这个口径包含 page cache——容器里读文件、写日志产生的缓存页都算进额度。`free` 看到的是宿主机视角（容器未隔离 /proc/meminfo），完全不能反映 cgroup 真实水位，所以"free 还有剩余"与"被 OOM kill"可以同时成立。排查步骤：`cat /sys/fs/cgroup/memory.current /sys/fs/cgroup/memory.max` 对比实际用量；`cat /sys/fs/cgroup/memory.stat` 看 cache 占比，若 cache 占大头，说明日志/文件读写把额度吃满了；`dmesg` 或 K8s 事件里找 `Memory cgroup out of memory: Killed process` 确认是 cgroup OOM 而非全局 OOM。修复方向：合理设置 limits（request 与 limit 之间留缓冲，别设置相等导致无回旋余地）；cache 压力大时用 memory.high 提前触发回收；应用层减少无效文件读取、控制日志量。实践坑：Java 堆外、Go 栈增长在 cgroup 视角都会暴增，只调 JVM 堆参数不够；关闭 swap 的容器里，内存压力会直接走向 OOM，没有缓冲。

**延伸考点：** cgroup v2 里 memory.high 与 memory.max 的区别是什么？为什么生产推荐用 cgroup v2？

### Q7. 服务器 load average 15 但 CPU 使用率才 20%，怎么解释、怎么查？

**问题：** 低峰期某台机器突然响应变慢，top 显示 load average 40+，但 CPU 使用率不到 20%，怎么解释？排查步骤是什么？

**期望加分项：**
- 能说清 load 的口径：R（可运行）+ D（不可中断睡眠）状态的线程数均值，CPU 低但 load 高大概率是 D 状态堆积
- 会用 top 按 D 过滤、ps 看 STAT 列、iostat/vmstat 看磁盘与 swap
- 知道 D 状态常见根因：磁盘故障/慢盘、NFS 挂死、设备驱动问题、swap 抖动
- 给出有序的排查命令链，而不是想到哪查到哪
- 主动说明 D 状态进程 kill 不掉，只能等内核解除阻塞

**减分项：**
- 只答"CPU 被占满了"（与题目事实矛盾）
- 分不清 load average 与 CPU 使用率的口径
- 不知道 D 状态意味着什么
- 没有排查动作，只有概念复述

**解答：**
先纠正口径：load average 是 R（可运行）+ D（不可中断睡眠）线程数的滑动平均，CPU 使用率低而 load 高，说明大量进程阻塞在不可中断的等待上——最常见是磁盘 I/O，其次 NFS/网络文件系统、swap 抖动、设备驱动异常。排查顺序：`top` 里按 `D` 键过滤，看 D 状态进程是什么；`ps -eo stat,pid,wchan,comm | grep '^D'` 看它们阻塞在内核哪个函数（wchan 列很有用）；`iostat -x 1` 看 %util、await、rrqm/wrqm，%util≈100% 且 await 高说明磁盘打满；`vmstat 1` 看 bi/bo、si/so 判断是否 swap 抖动；`dmesg` 查 I/O error、掉盘（/dev/sdX I/O error）、NFS server 不响应等硬件与协议层线索。实践坑：D 状态进程 kill 不掉（不可中断，SIGKILL 也无效），只能等内核解除阻塞，所以排查要快，别把时间花在反复 kill 上；常见根因按概率排：慢盘/掉盘、日志突增打满磁盘、NFS/云盘断连、swap 在慢盘上引发的回收风暴。修复后 load 不会立刻回落，要等 D 队列消化，属正常现象。

**延伸考点：** R 状态与 D 状态在 load 里各自代表什么？为什么 D 状态连 kill -9 都杀不掉，内核层面它停在哪里？

### Q8. 网关连接数上万、响应变慢，怎么判断是应用瓶颈还是内核 socket 瓶颈？

**问题：** 网关服务连接数上万后出现延迟抖动，怎么分辨是应用处理慢，还是内核网络栈（backlog、缓冲区）在丢包排队？

**期望加分项：**
- 会用 ss/netstat 看 Recv-Q/Send-Q 积压，以及监听队列溢出计数（netstat -s 里的 overflowed）
- 会看 TCP 重传、乱序、SYN 重试等内核计数器，判断是否在丢包
- 能区分"连接总数多"与"活跃并发高"：epoll 模型下大量空闲连接开销有限，活跃连接才是瓶颈
- 给出分层判断逻辑：内核丢包计数增长且应用 CPU 不高 → 内核层；应用线程全忙 → 应用层
- 知道 backlog 与 somaxconn 的生效关系（取较小值）

**减分项：**
- 只会说"连接数太多要加机器"
- 不看任何计数器就下结论
- 混淆并发连接数与吞吐量
- 不知道半连接/全连接队列溢出的现象差异

**解答：**
先分两个层面：应用层慢（CPU、锁、业务逻辑）与内核栈慢（队列溢出、丢包、重传）。命令组合：`ss -lnt` 看监听 socket——Recv-Q 接近 Send-Q（backlog 上限）说明全连接队列溢出，客户端表现为 connect 慢或超时；`netstat -s` 看 `times the listen queue of a socket overflowed` 计数和 TCP 重传数；`ss -tan` 按需过滤后重点看 established 连接的 Recv-Q/Send-Q 是否持续积压。再配合 `top`/`vmstat` 排除 CPU 瓶颈、`strace -p` 看应用是否阻塞在 recv。判断逻辑：内核丢包计数持续增长而应用 CPU 不高 → 内核层问题，调 `net.core.somaxconn`、`net.ipv4.tcp_max_syn_backlog` 与应用侧 backlog 参数（Java 的 listen backlog 默认很小）；应用 CPU 打满、线程全忙 → 应用瓶颈，走代码优化。实践坑：很多框架默认 backlog 只有 128，高并发下必溢出；调大 backlog 必须同时调 somaxconn（两者取小生效）；还要区分半连接队列——SYN flood 打满半连接队列时，客户端连三次握手都完不成，现象完全不同。另外上万连接里大部分可能是空闲 keepalive 连接，先看活跃连接数再下结论。

**延伸考点：** 半连接队列溢出与全连接队列溢出各自的现象、定位计数器是什么？分别怎么调优？

### Q9. 磁盘 I/O 很慢，iostat %util 长期 100%，但业务读写量并不大，可能是什么原因？

**问题：** 数据库服务器 iostat 显示 %util 100%，但业务 QPS 不高、读写字节数不大，磁盘却慢，怎么分析？

**期望加分项：**
- 能说清 %util 的量化口径：设备忙的占比，随机小 I/O 即使字节少也能打满 %util
- 会看更多指标：await（含排队）、avgrq-sz（平均请求大小）、rrqm/wrqm 合并率，判断是随机小 IO 还是吞吐问题
- 会用 pidstat -d / iotop 找谁在打盘（慢查询、binlog/redo 刷盘、日志、备份、kswapd 回写）
- 结合数据库场景：innodb_flush_log_at_trx_commit、刷脏页、fsync 频率
- 主动考虑边界：SSD 与机械盘对随机 IO 的差异、云盘限流

**减分项：**
- 只看 %util 就断言"磁盘坏了"或"磁盘不够用"
- 不理解 %util 只代表"有请求在排队"而非带宽打满
- 不查系统里还有谁在写盘
- 没有任何优化方案输出

**解答：**
先厘清 %util 语义：它表示设备忙的时间占比，随机小 I/O 即使字节数少，每次请求都有寻道和排队，也能让 %util 逼近 100%。所以别只看这一个数：`iostat -x 1` 里 await（平均响应含排队）高、avgrq-sz 很小（8-16KB 级别）、rrqm/wrqm 合并率低，就是典型的随机小 I/O 场景。接着用 `pidstat -d 1` 或 `iotop` 找到底谁在打盘，常见元凶按概率排：慢查询引发的全表扫描、数据库每事务 fsync（innodb_flush_log_at_trx_commit=1 时写 redo 是同步落盘）、应用日志每行 flush、备份任务与业务高峰期重叠、以及内核 kswapd/pdflush 在做脏页回写。优化方向：数据库合并小事务、调大 innodb_io_capacity、日志批量写、数据盘与日志盘分离、机械盘换 SSD（随机 IO 提升两个数量级）。实践坑：%util 100% 不代表盘坏了或带宽打满，判断"慢"要看 await 和业务延迟；云盘场景 %util 还要考虑底层限流（云盘 IOPS 配额），本地 iostat 看到的高 util 可能是云盘排队导致。别一上来就扩容磁盘，先确认是 IO 模式问题还是容量问题。

**延伸考点：** 顺序写与随机写分别在机械盘和 SSD 上差多少？为什么数据库常用"独立日志盘 + 顺序写"？

### Q10. 服务报 "Too many open files"，部分请求失败，怎么定位、怎么根治？

**问题：** 服务日志出现 "Too many open files"，部分请求开始失败，怎么定位是哪些 fd、哪些代码路径在泄漏？

**期望加分项：**
- 能说清 fd 的三层限制：进程 ulimit -n、系统 fs.file-max、容器/cgroup 限制
- 会用 lsof -p / ls /proc/<pid>/fd 统计 fd 类型分布（TCP、文件、pipe），多次观察确认是否只增不减
- 用 lsof 看 TCP fd 的对端 IP 与状态，定位连到哪个下游
- 区分"峰值不够用"与"真泄漏"，根治靠代码关闭逻辑而非单纯调大
- 知道容器内 ulimit 视角与真实限制可能不一致

**减分项：**
- 只答"ulimit -n 调大"
- 不查 fd 类型与来源
- 把进程级与系统级限制混为一谈
- 没有代码层面的定位方法

**解答：**
先区分"峰值不够"还是"真泄漏"：`ls /proc/<pid>/fd | wc -l` 或 `lsof -p <pid> | wc -l` 看当前数量，连续观察几次，只增不减才是泄漏。再看分布：`lsof -p <pid>` 输出按 TYPE 列统计——TCP 多说明连接泄漏（Java 里没 close 的 HTTP client、MySQL 连接没归还、Netty 通道没释放）；REG 文件多说明文件句柄没关；pipe/eventfd 多说明线程或子进程异常。`lsof -p <pid> | grep TCP` 能看到对端 IP 和连接状态，能定位连的是哪个下游、是不是在无限建连。限制层级要分清：进程级 `ulimit -n`（软/硬限制）、系统级 `fs.file-max`、容器场景还有 pids 限制和 systemd LimitNOFILE——容器里 `ulimit -n` 看到的可能是宿主值，与容器实际限制不一致，要查 cgroup 或 systemd unit。根治靠代码：所有流用 try-with-resources/close、连接池设合理 max、检查异常路径是否漏关；临时调大只是兜底（`ulimit -n 1048576` 或 systemd 配 LimitNOFILE），要同时评估 fs.file-max。实践坑：fd 耗尽不只是请求失败——日志写不进去、健康检查异常，服务会表现为"整体卡死"，需要从根上避免泄漏。

**延伸考点：** Java 进程在容器里看到的 ulimit 与实际限制不一致是怎么回事？fd 耗尽为什么会导致服务整体"卡死"？

### Q11. 磁盘还有 50% 空间，但写文件报 "No space left on device"，怎么回事？

**问题：** 服务写临时文件报 No space left on device，但 `df -h` 看磁盘还剩 50%，怎么排查？

**期望加分项：**
- 第一时间想到 inode 耗尽：`df -i` 看 inode 使用率
- 知道 inode 耗尽的常见来源：海量小文件（临时文件、日志残留、容器 overlay 层）
- 会用 find/du --inodes 定位小文件大头目录
- 能区分"目录/挂载点"维度与"空间"维度（每个挂载点独立 inode 池）
- 给出清理与预防方案（轮转、tmpfs、mkfs 时调 inode 密度）

**减分项：**
- 只会重启服务或换目录
- 不检查 df -i
- 不知道容器内 df 看到的是宿主视角
- 忽略"文件被进程打开、删除后空间不释放"的场景

**解答：**
思路很直接：空间够但写不进，说明不是块空间不够，而是元数据耗尽——`df -i` 看各挂载点 inode 使用率，IUse% 接近 100% 就是 inode 耗尽。inode 是每个文件一份的元数据，小文件数量暴增时最先耗尽。常见来源：应用临时文件没清理、日志轮转残留（按小时切分且不删旧档）、缓存目录、以及 Docker 容器 overlay 层里的小文件（注意容器内 `df -i` 是宿主视角，要在对应挂载点上看，或看宿主上容器目录）。定位大头：`find /data -xdev -type f | wc -l` 按目录统计文件数（或用 `du --inodes`），找到数量最多的目录再针对性清理。预防：日志加轮转与保留期、临时文件放 tmpfs（内存文件系统，重启即清）、建文件系统时按场景调 inode 密度（mkfs 的 `-i bytes-per-inode`）。实践坑：删除大量小文件时 `rm -rf` 本身也会很慢（目录项多）；被进程打开着的已删除文件，空间和 inode 要等进程关闭才释放——`lsof +L1` 能列出这类 deleted 但被占用的文件，这是"删了空间没释放"的经典原因。

**延伸考点：** lsof +L1 查"已删除但被占用"的文件有什么实际应用场景？为什么日志轮转后磁盘空间依然不减？

### Q12. 服务器 free 显示 cache 占了几十个 G，有人建议清 cache，你怎么看？

**问题：** `free -h` 里 cache 占 30G+，业务说内存紧张，运维提议 `echo 3 > /proc/sys/vm/drop_caches` 清掉，你同意吗？

**期望加分项：**
- 能说清 page cache 的作用与"可用内存"的正确口径：MemAvailable 已包含可回收 cache
- 理解内核回收机制：内存有压力时内核自动按 LRU 回收干净 cache，drop_caches 是手动强制、治标不治本
- 会判断真内存压力：MemAvailable 低、swap 使用上升、sar -B 的 pgscand 直接回收频繁
- 指出清 cache 的代价：热数据命中下降、I/O 变慢、期间抖动
- 主动考虑边界：dirty page 不能随便回收（涉及数据一致性）

**减分项：**
- 直接同意或反对"清 cache"，不讲原理
- 把 MemAvailable 与 free 混为一谈
- 不知道内核回收 cache 的优先级与水位机制
- 无实践，只背"cache 是好东西别清"的结论

**解答：**
先给判断：不用急着清，先分清楚"内存紧张"是不是真的。`free` 的 MemAvailable 才是内核估算的可分配量，它已经把"可回收的 cache"算进去了；只有 MemAvailable 持续很低、swap 使用上升、`sar -B` 显示 pgscand（直接回收）频繁时才是真压力。page cache 由内核自动管理：内存有压力时，内核按 LRU 优先回收干净 cache 满足分配请求，drop_caches 只是把这一动作手动提前，属于"表面缓解"。风险：清了之后读缓存全失效，热数据回源读盘，磁盘 I/O 反而变慢，线上甚至可能看到明显的延迟抖动。正确动作：先确认是不是真压力（看 MemAvailable 趋势和 swap），如果是，查谁在大量读写文件撑大 cache、dirty 页比例是否过高（`/proc/sys/vm/dirty_ratio`、`dirty_expire_centisecs`），按需调整 swappiness 与 dirty 参数，或优化应用读写模式；生产上清 cache 只适合"需要立即释放内存给大任务"的临时场景，且应在低峰执行。实践坑：`echo 3` 同时清 page cache、dentry、inode，期间可能有明显 I/O 抖动；别把"cache 大"当故障告警，正确告警应该盯 MemAvailable 与 swap。

**延伸考点：** MemAvailable 是怎么估算出来的？dirty page 为什么不能随便回收（对数据一致性意味着什么）？

### Q13. 高并发网络服务，你们怎么选 I/O 模型？为什么用 epoll 而不是多线程阻塞 I/O？

**问题：** 给高并发网关选 I/O 模型：阻塞多线程、select/poll、epoll、异步 I/O（io_uring），你依据什么选？为什么主流组件都用 epoll？

**期望加分项：**
- 能说清各模型差异与量级：阻塞模型每连接一线程的代价；select 的 1024 fd 上限与 O(n) 扫描；poll 无上限但仍线性扫描；epoll 事件驱动 O(就绪数)；io_uring 减少系统调用
- 联系真实组件：nginx/redis 用 epoll 的理由、Netty/Java NIO、Go runtime 的 netpoll
- 说清"多路复用 + 非阻塞"与"异步 I/O"的本质区别
- 主动考虑边界：连接数规模、业务逻辑是否 CPU 密集（事件循环里不能做阻塞操作）
- 能指出 epoll 的坑：LT/ET、惊群、回调里做重活

**减分项：**
- 只会背 epoll 三个函数名
- 说不清 select/poll 为什么不行（fd 上限、全量拷贝、线性扫描）
- 把 epoll 与异步 I/O 混为一谈
- 无实践佐证，没说过自己服务的模型与理由

**解答：**
先讲决策逻辑：连接数少（几十到几百）时阻塞多线程最简单，每连接一线程，编码直观、易调试；连接数上万后线程数成为瓶颈（栈内存、上下文切换），必须"少量线程 + 多路复用"。select 的问题是 fd 集合上限 1024、每次调用要把整个 fd 集合从用户态拷贝进内核、还要线性扫描所有 fd，O(n)；poll 去掉了上限但仍是"全量拷贝 + 线性扫描"。epoll 用红黑树注册 + 就绪链表 + 回调机制，一次注册多次等待、只取就绪 fd，复杂度 O(就绪数)，不需要每次全量拷贝，这是 nginx、redis、Netty 选它的核心原因。io_uring 更进一步：用共享内存环形队列批量提交请求与收割完成事件，大幅减少系统调用次数，适合纯 I/O 密集的高性能存储，但复杂度高、生态还在成熟。实践要点：事件驱动模型要求处理逻辑快且不阻塞，回调里做同步 I/O、加锁、或 CPU 重活会拖垮整个事件循环——所以重逻辑要丢到线程池；Java 的 Netty EventLoop 是单线程循环，handler 里绝不能有阻塞调用。选型边界：CPU 密集的离线任务根本不需要 epoll，多线程分片更合适。

**延伸考点：** epoll 的 ET（边缘触发）与 LT（水平触发）区别？为什么 Redis 用 ET？Go 的 netpoll 相比 epoll 解决了什么问题？

### Q14. 发布时 kill 掉旧实例，在途请求丢了怎么办？你们怎么做优雅停机的？

**问题：** 发布时把旧实例 kill 掉，kill 瞬间正在处理的请求会失败，怎么设计优雅停机，保证在途请求不丢？

**期望加分项：**
- 能说清信号机制：SIGTERM 可捕获处理、SIGKILL 无法捕获，kill -9 是最后手段
- 完整流程：先摘流量（LB/注册中心标记不可用）→ 处理完在途请求（带超时）→ 释放资源 → 退出
- 联系 K8s：preStop hook、terminationGracePeriodSeconds、readiness 探针摘流、优雅排空的顺序
- 联系框架：Spring Boot graceful shutdown、ShutdownHook、Netty channel 关闭、消息队列的 shutdown 信号
- 主动考虑边界：长事务/长轮询的强制超时，避免无限等待拖死发布

**减分项：**
- 只会说"kill -9 就完事"
- 不知道 SIGTERM 与 SIGKILL 的区别
- 没有"摘流量"概念，以为服务端优雅就够了
- 不考虑超时兜底，或者无脑等所有请求处理完

**解答：**
核心认知：优雅停机是"摘流量 + 排空在途请求 + 超时兜底"三步，不是单靠一个 kill。流程：1) 摘流量——先让 LB/注册中心/网关把该实例标记为不可用（readiness 探针返回失败、Dubbo 下线通知、nginx upstream 剔除），停止新请求进入；2) 信号处理——应用收到 SIGTERM 后停止接受新任务，等待在途请求处理完（Spring Boot 的 graceful shutdown、JVM ShutdownHook、Netty 的 channel 关闭逻辑），这个等待要设上限（如 30s），避免长事务把发布拖死；3) 超时兜底——到点强制退出，K8s 里对应 terminationGracePeriodSeconds，配 preStop hook 做摘流和 sleep 缓冲。实践坑：只做服务端优雅没用——如果 LB 还在往旧实例转发，排空永远完不成，所以"先摘流再等"的顺序不能反；kill -9 是 SIGKILL 无法被捕获，直接终止，可能丢数据（比如正在写 WAL 但没 fsync 的事务），只在 graceful 超时后才用；Java 的 ShutdownHook 里别做重活（JVM 等待时间有限，约几秒到十几秒）；有状态服务（消息消费者）还要考虑 ACK 与重投机制兜底，避免"处理了但没 ACK"的重复消费。

**延伸考点：** K8s 中 preStop 与 terminationGracePeriodSeconds 的配合逻辑是什么？为什么摘流后常要 sleep 几秒再退出？

### Q15. CPU 持续 100%，怎么在不停服的情况下定位到具体函数/方法？

**问题：** 线上某服务 CPU 持续 100%，怎么不停服就定位到是哪个线程、哪段代码在烧 CPU？

**期望加分项：**
- 有清晰的定位链路：top → top -Hp / pidstat → 采样线程栈（perf、Arthas profiler、Go pprof）
- 会区分用户态与内核态 CPU（us/sy 占比），方向不同（业务热点 vs 系统调用/锁自旋）
- 了解火焰图：perf record -g 与 async-profiler 的用法
- 有"先留证据再处理"的意识：采样留存后再重启，避免无法复盘
- 主动考虑边界：多核下 %CPU 含义、短时 vs 持续高 CPU、容器内采样权限

**减分项：**
- 只会 top 看到进程，定位不到代码层
- 不知道 perf/Arthas/pprof 等采样工具
- 不看 us/sy 占比，方向错了还在查业务代码
- 无复现/压测手段，纯靠猜

**解答：**
定位链路是三层：`top`（或 `ps aux --sort=-%cpu`）先锁定进程 PID → `top -Hp <pid>` 或 `pidstat -p <pid> 1` 看到底是哪个线程在烧 CPU → 采样这个线程的调用栈定位代码。采样工具是核心：Linux 原生 `perf top -p <pid>` / `perf record -g -p <pid>`（能拿到用户态+内核态调用栈，生成火焰图；容器里跑需要宿主权限或 CAP_PERFMON）；Java 服务用 Arthas `profiler start`（基于 async-profiler，输出火焰图最直观，JDK 9+ 还可用 jfr）；Go 用 `pprof` CPU profile；Python 用 py-spy。拿到栈后看热点函数，常见根因：死循环、正则回溯、JSON/序列化、GC 频繁（GC 线程 CPU 高说明堆压力大，别只盯业务线程）。先看 us/sy 比例：sy 高说明内核态忙——系统调用密集、中断风暴、锁自旋、内存分配频繁，方向完全不同。实践坑：采样要多抓几份、隔几秒对比，短时热点与持续热点结论不同；容器里很多工具抓不到宿主线程，要在宿主机上跑或开相应权限；先留火焰图证据再处理，否则下次复现又得重查。另外 %CPU 在 top 里默认是"单核百分比"，多核整体 100% 与单核打满含义不同，先看清楚再说。

**延伸考点：** perf 火焰图怎么读（x 轴、y 轴、宽度含义）？CPU 高但业务无明显异常，可能是哪类"隐性消耗"（伪共享、偏向锁、自旋）？

### Q16. 新版本上线后服务整体变慢，怎么判断是应用问题还是系统问题？系统调用的开销在哪？

**问题：** 新版本上线后服务整体变慢，怎么分层判断瓶颈在应用代码还是系统层面？为什么系统调用本身就有开销？

**期望加分项：**
- 会用工具分层：top 看 us/sy/wa、vmstat 看 r/cs/bi/bo、mpstat -P ALL 看单核、pidstat 看单进程
- 能说清系统调用开销的来源：用户态→内核态切换（寄存器保存/恢复、栈切换）、可能的上下文切换、TLB 与缓存失效
- 会用 strace -c 统计各 syscall 的次数与耗时分布，并知道 strace 对性能的放大影响
- 主动考虑边界：strace 有 10 倍级性能损耗，只在低峰短时使用，或用 perf trace
- 有新旧版本对比的验证思路

**减分项：**
- 笼统答"查日志、加监控"没有具体方法
- 不区分用户态/内核态耗时
- 滥用 strace，不考虑代价
- 没有系统化的排查路径

**解答：**
先分层再看数据：`top` 看 us/sy/wa 占比——sy 高优先怀疑系统调用与内核路径；`vmstat 1` 看 r（运行队列）、cs（上下文切换）、bi/bo（I/O），判断 CPU 排队 vs I/O 瓶颈；`mpstat -P ALL` 看是否单核打满（负载不均，比如一个线程锁热点）；`pidstat -p <pid> 1` 看单进程 CPU 与切换。系统调用开销的本质：每次 syscall 都要从用户态切到内核态——模式切换要保存/恢复寄存器、切内核栈，更贵的是阻塞型 syscall 会引发线程上下文切换，频繁调用还会造成 TLB 失效和缓存污染。定位手段：`strace -c -p <pid>` 跑几秒，能给出各 syscall 的调用次数与耗时占比，一眼看出是不是在读文件、在 recv、在 fsync 上耗掉大量时间；注意 strace 本身会让目标进程慢 10 倍以上，只能在低峰短时用，或者用 `perf trace`、bpf 工具（如 bpftrace 统计 syscall 分布）替代。常见"系统慢"的隐蔽根因：频繁小写入触发 fsync、大量短连接建连、锁自旋（用户态忙）、内存分配频繁触发缺页。验证闭环：新老版本压测数据对比、确认变慢时间点与发布/配置变更对齐，用证据说话而不是感觉。

**延伸考点：** vDSO 是什么？为什么 gettimeofday 这类 syscall 可以通过 vDSO 避免内核切换，省了什么？

### Q17. 容器里改 sysctl 不生效、应用看到的 CPU 核数也不对，怎么回事？

**问题：** 容器里执行 `sysctl -w net.ipv4.tcp_tw_reuse=1` 报错或重启失效；16 核宿主机上容器只分 2 核，Java 的 Runtime.availableProcessors() 却返回 16，会有什么问题？

**期望加分项：**
- 理解 namespace 隔离边界：net/pid/mount/uts/ipc/user 隔离"视角"，但 sysctl 大多全局，net.* 部分可按 netns 隔离
- 知道容器默认没挂 lxcfs 时 /proc 是宿主视角，availableProcessors 会"骗人"
- 说清后果：线程池、GC 并行线程、Go GOMAXPROCS 按宿主核数初始化 → 容器内超卖、锁竞争加剧
- 给出方案：Java 的 -XX:ActiveProcessorCount / -XX:ParallelGCThreads；Go 用 automaxprocs；或挂 lxcfs
- 知道 K8s sysctl 的 safe/unsafe 分类与生效方式

**减分项：**
- 不知道容器与 /proc 的关系
- 说不清 namespace 隔离了什么、没隔离什么
- 没有给出可落地的解决方案
- 把 sysctl 当成容器内可随意改的全局配置

**解答：**
两个问题本质是"容器隔离但不完全"：namespace 隔离了进程视角（PID、网络、挂载、主机名），cgroup 限制资源用量，但 /proc 默认仍是宿主视角。所以 Java 的 `Runtime.availableProcessors()` 读到宿主核数，默认线程池（Tomcat 的 maxThreads、ForkJoinPool 的并行度）、GC 并行线程数、Go runtime 的 GOMAXPROCS 全都按宿主核数初始化——容器只有 2 核时这等于超卖，CPU 时间片打满、锁竞争加剧，服务"莫名变慢"。解法：Java 设 `-XX:ActiveProcessorCount=2`（JDK 9+）和 `-XX:ParallelGCThreads`；Go 引 Uber 的 automaxprocs 自动读 cgroup；或者给容器挂 lxcfs，让 /proc/cpuinfo 呈现容器视图。sysctl 方面：`net.*` 部分参数按 netns 隔离（新网络命名空间有自己初始值，容器内改自己的 netns 可以，但重启容器即丢，要配 init 容器或 K8s sysctl 策略持久化）；vm/fs 等参数是宿主全局的，容器里改要么报权限错误、要么影响整台宿主机。K8s 支持通过 pod securityContext 的 sysctls 声明 safe sysctl（如 ip_local_port_range）和 unsafe sysctl（需管理员在 kubelet 显式启用）。实践坑：镜像 Dockerfile 里写 /etc/sysctl.conf 通常无效（很多基础镜像没有 sysctl 应用机制）；诊断容器"看到几核"先执行 `nproc` 和 `cat /sys/fs/cgroup/cpu.max` 对比。

**延伸考点：** cgroup v2 下 /proc/cpuinfo 对容器 CPU 视图有什么改进？为什么说"容器内 nproc 不可靠"？

### Q18. 安全评审被问容器隔离边界：namespace 和 cgroup 各防什么？容器逃逸的常见路径有哪些？

**问题：** 安全评审时被问：namespace 和 cgroup 分别解决什么问题、防不住什么？你了解哪些容器逃逸路径，怎么防御？

**期望加分项：**
- 能说清分工：namespace 隔离"看得见的"（进程、网络、挂载视图），cgroup 限制"能用多少"（CPU/内存），但容器与宿主共享同一个内核
- 能列出常见逃逸路径：危险挂载（docker.sock、宿主根目录）、特权容器（--privileged/SYS_ADMIN）、内核漏洞、错误的能力/安全配置
- 有防御清单：非 root 运行、drop capabilities、只读根文件系统、禁止危险挂载、seccomp/AppArmor、高隔离用 gVisor/Kata
- 主动考虑边界：逃逸=拿到宿主权限，不只是打破 cgroup 限额

**减分项：**
- 以为容器是虚拟机、完全隔离
- 只会背 namespace/cgroup 两个名词
- 不了解特权容器与 docker.sock 挂载的风险
- 无防御实践输出

**解答：**
先立认知：容器与宿主机共享内核，namespace 负责隔离"视图"（PID、网络、挂载、UTS、IPC、user），cgroup 负责"限额"（CPU、内存、IO），但两者都不提供内核级隔离——同一个内核、同一套 syscall 接口，这是逃逸的根源。常见逃逸路径按现实概率排：1) 特权容器（--privileged 或给了 SYS_ADMIN）配合 mount 宿主根目录，往 /etc/crontab、/root/.ssh 写入后门；2) 把宿主 docker.sock 挂进容器，容器内直接调 Docker API 控制宿主机（经典 `docker run -v /:/host` 卷土重来）；3) 危险挂载：/proc/sys、宿主 / 根目录、/var/run 等敏感路径；4) 内核漏洞（如 CVE-2022-0185、脏牛系列），本地提权；5) 配置错误：容器内以 root 跑、未启用 seccomp、capabilities 全开。防御实践：镜像与运行态强制非 root（runAsNonRoot）、securityContext 里 `capabilities: drop: [ALL]` 只加必要项、根文件系统只读（readOnlyRootFilesystem）、禁止挂载 docker.sock 与宿主目录、启用 seccomp 默认配置与 AppArmor/SELinux、镜像定期扫描漏洞；对高隔离要求场景（多租户）直接用 gVisor 或 Kata 虚拟化。实践坑：容器内 root 在 user namespace 未开启时就是宿主 UID 0，配合内核漏洞直接提权，"容器内是 root"不是安全边界；团队应该按 K8s Pod Security Standards（baseline/restricted）定基线。

**延伸考点：** user namespace 未开启时，容器内 root 的权限边界到底在哪？seccomp 与 AppArmor 各自限制的是什么？

### Q19. 服务启动超过 systemd 默认超时被 kill，启动过程卡在哪？怎么排查？

**问题：** 发布时 systemd 服务启动超过超时时间被 kill（Type 判定启动失败），业务启动很慢，怎么定位卡在哪个环节？

**期望加分项：**
- 会看 journalctl -u <svc> 启动日志、systemd-analyze blame / critical-chain 看耗时单元
- 理解 systemd 判断"启动完成"的机制：Type=simple/forking/notify 决定何时算成功，TimeoutStartSec 语义
- 应用侧分阶段排查：JVM 初始化、连接 DB/Redis 超时重试、拉配置、预热逻辑、健康检查
- 主动考虑边界：启动慢 vs 一直起不来；调大超时前先确认业务有"启动完成标志"
- 联系常见坑：DNS 解析慢、连接池对每个节点逐次超时、日志刷盘

**减分项：**
- 只会重启，不定位
- 不知道 systemd 按 Type 判断启动完成
- 不查 journald 日志
- 不区分启动脚本阶段与应用初始化阶段

**解答：**
分两层看。systemd 判断"启动完成"的时机取决于 Type：simple 是 exec 即算成功；forking 等父进程退出；notify 等进程调用 sd_notify 通知。先确认是不是 Type 配置与进程行为不匹配导致误判超时（比如 Type=forking 但进程不 fork）。排查动作：`journalctl -u <svc> -f` 看启动期间日志与报错时间线；`systemd-analyze blame`、`systemd-analyze critical-chain` 看系统启动各单元耗时；应用侧看启动日志各阶段时间戳，常见卡点：JVM 类加载与 Metaspace、连数据库/Redis 时 socket 连接超时（配错地址时逐个节点重试、默认几十秒）、初始化期间拉配置中心、预热缓存/线程池、以及"进程活着但健康检查一直不过导致被反复重启"。TimeoutStartSec 默认 90s，确需调大时要同时确认业务侧有明确的启动完成标志，否则超时配置形同虚设。实践坑：很多"启动慢"其实是依赖慢——连接池初始化会对每个节点做连接尝试，配错地址会逐节点超时；DNS 解析慢（没有本地缓存、外部 DNS 抖动）也常见；systemd 超时后默认发 SIGTERM，配合 ExecStop 与 journald 的退出日志能判断是"被超时杀"还是"自己崩"。定位要趁启动窗口抓证据，别等服务起来了再查。

**延伸考点：** Type=notify 的服务（很多中间件）的启动完成信号由谁发出？为什么会出现"进程活着但 systemd 判定失败"？

### Q20. 综合题：线上业务大面积超时，给你 10 分钟，你会按什么顺序排查？

**问题：** 线上业务大面积超时/报错，多实例同时出问题，你的第一反应是什么？给出一份可执行的排查顺序清单。

**期望加分项：**
- 先讲"止血优先"策略：恢复优先于分析，摘流量/回滚/重启是合法动作
- 有固定排查框架：影响面与故障性质 → 最近变更 → 监控与日志 → 依赖与系统资源
- 把"最近变更"列为第一怀疑对象（发布、配置、扩容、依赖方变更），做时间线对齐
- 能区分故障模式：全量 vs 单实例、延迟型 vs 错误型，先分类再定向
- 处置前保留现场（dump/日志/监控截图），避免"重启即失忆"，复盘要有证据
- 有协作意识：统一指挥口径，避免多人乱改配置

**减分项：**
- 一上来深挖代码，不先止血
- 没有固定框架，东查一下西查一下
- 不查变更历史，纯猜原因
- 忘了保留现场就重启
- 无复盘与改进动作

**解答：**
给一个固定框架，按优先级执行：1) 确认影响面与性质——看监控大盘（入口 QPS/延迟/错误率曲线），区分"全量故障"还是"单实例故障"、延迟型还是错误型，这决定后续方向（全量+错误率上升 → 依赖或基础设施；单实例 → 该实例资源与日志）；2) 查最近变更——发布记录、配置变更、依赖方公告，把变更时间线与故障开始时间对齐，这是效率最高的一步，问题十有八九在变更里；3) 与变更无关再按"应用→系统→依赖"查：top/vmstat 看资源、日志看报错模式（连接拒绝？超时？OOM？）、再到下游看连接池与慢查询；4) 止血动作随时可做：摘流量、重启实例、回滚版本、限流降级——恢复优先于根因分析，但动作前先留现场（抓线程 dump、日志段、监控截图），避免"重启后无法复盘"；5) 复盘闭环：产出根因、补充监控盲区、沉淀预案。事故模式速查：多实例同时超时 → 依赖或基础设施；延迟上升但错误少 → 慢查询/GC/锁；OOM 反复 → 查 cgroup 与内存参数。实践坑：故障中最忌讳多人各自重启和改配置——要有单人口径（指挥者），所有动作记录在案，否则"修好了但不知道哪步生效"，复盘也没法做。

**延伸考点：** 讲一次你经历过最典型的线上故障：从发现到恢复的时间线、根因、以及事后加了哪些措施？（重点考察复盘能力与真实经历，编造的一问细节就会露馅）
