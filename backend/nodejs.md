# 后端 · Node.js（面试题库）

本文件考察 Node.js 真实工程实践能力，而非八股背诵。重点覆盖：事件循环与异步执行模型、事件循环阻塞与性能问题的定位手段、内存管理与线上泄漏排查、多进程/多线程选型与高并发架构、流式处理与背压、错误处理与进程稳定性，以及工程化实践（TypeScript、容器化部署、限流缓存、框架选型等）。所有问题均来自一线生产场景，期望候选人用真实项目经验、量化手段与取舍判断作答，而不是复述教科书定义。

### Q1. 事件循环各阶段如何流转？微任务与宏任务的优先级到底怎么排？

**问题：** 定时任务里日志时序"诡异"：`setTimeout` 回调中创建的 Promise 微任务，总在 `setImmediate` 之后才执行。请解释 Node.js 事件循环的完整流程，以及为什么会出现这种顺序。

**期望加分项：**
- 能准确描述阶段流转：timers → pending callbacks → idle/prepare → poll → check → close callbacks，且微任务在每个阶段切换时清空
- 能说清 `process.nextTick` 优先于 Promise 微任务，以及它"饿死事件循环"的副作用
- 能解释 `setTimeout(0)` 与 `setImmediate` 在主模块与 I/O 回调上下文中顺序相反的根因
- 能结合线上排查经历说明如何验证（打点、`--trace-events`）
- 能讲清 libuv 线程池（异步 I/O、DNS、文件系统）与主事件循环的关系边界

**减分项：**
- 只背"微任务优先于宏任务"，讲不清阶段概念
- 把 `process.nextTick` 与 Promise 混为一谈
- 只画示意图，没有任何实际验证或踩坑经历

**解答：**

先给判断：这类"顺序诡异"问题，九成是混淆了"事件循环阶段"与"微任务清空时机"两个概念。Node 的主循环基于 libuv，按 timers → pending callbacks → idle/prepare → poll → check → close callbacks 六个阶段往复。关键规则：微任务（`nextTick` 与 Promise）不占用阶段，而是在**每个阶段切换时**将队列全部清空，其中 `nextTick` 队列先于 Promise 队列。

这就解释了你的现象：`setTimeout` 回调在 timers 阶段执行，它内部创建的微任务要等当前阶段结束、进入下一阶段前才会执行；而 `setImmediate` 在 check 阶段。如果事件流先推进到了 check，`setImmediate` 自然先跑。在主模块中，`setTimeout(0)` 与 `setImmediate` 谁先执行取决于进程启动耗时；但在 I/O 回调（poll 阶段）内，poll 之后必进 check，所以 `setImmediate` 稳定先于 `setTimeout(0)`。

```js
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
setTimeout(() => console.log('timer'), 0);
setImmediate(() => console.log('check'));
// 稳定输出：nextTick → promise → timer / check（后两者顺序视上下文而定）
```

实践中的坑：在循环里反复使用 `process.nextTick` 会饿死 I/O（它永远插队到 I/O 前），应改用 `setImmediate` 让出。排查日志乱序时，给关键点打上 `Date.now()`/`process.hrtime` 时间戳即可快速确认属于阶段问题还是业务问题。

**延伸考点：** 如果在 timers 回调里写一个同步死循环，`setImmediate` 还执行吗？为什么？如何用 `perf_hooks.monitorEventLoopDelay` 度量事件循环延迟？

### Q2. 接口响应突然变慢，怀疑事件循环被阻塞，怎么定位与修复？

**问题：** 某接口平时 50ms，高峰期涨到 5s，CPU 居高不下，怀疑事件循环被某个任务阻塞了。请给出从确认到修复的完整排查路径。

**期望加分项：**
- 先量化确认"被阻塞"，而不是拍脑袋：`monitorEventLoopDelay` 采样、setInterval 打点看间隔漂移
- 能给出定位手段：`--cpu-prof` + `--prof-process`、clinic doctor/flame、`--inspect` 抓热点
- 能列举真实元凶：CPU 密集计算、正则灾难性回溯、同步 I/O、大 JSON 解析、TTY 下 console.log 刷屏
- 修复时能说清取舍：worker_threads 拆分、分片 + setImmediate 让出、异步化，各自适用场景
- 主动考虑"阻塞"与"延迟高"的区分（GC 停顿、依赖慢也可能是根因）

**减分项：**
- 直接背"把同步改异步"的套话，给不出定位证据链
- 不知道有哪些现成的 profile 工具与命令
- 忽略正则回溯、console.log 这类隐蔽元凶

**解答：**

第一步是确认"事件循环真的被阻塞"，而不是外部依赖慢。最直接的方式是采样事件循环延迟：`const h = perf_hooks.monitorEventLoopDelay(); h.enable();` 定期读取 `h.max`/`h.mean`，或简单地在 `setInterval` 里打点，观察回调间隔是否大幅漂移。延迟高且 CPU 高 → 主线程有计算热点；延迟高但 CPU 低 → 大概率在等 I/O（同步 I/O 或依赖调用），方向完全不同。

第二步定位热点函数。不侵入代码的做法：启动时加 `--cpu-prof --cpu-prof-dir=./prof`，压测复现后 `node --prof-process isolate-*.log > out.txt`，看自顶向下占比；或直接用 `clinic flame -- node app.js` 生成火焰图，一眼看到热点。常见元凶按频率排序：大 JSON 的 `JSON.parse`/`stringify`、正则灾难性回溯（如 `(a+)+` 配恶意输入）、加密/压缩、同步读大文件、海量 `console.log`。

```bash
node --cpu-prof --cpu-prof-dir=./prof app.js
node --prof-process ./prof/isolate-*.log > profile.txt
```

第三步修复，按取舍选方案：纯计算任务丢 `worker_threads` 分片处理；超大 JSON 改流式解析（JSONStream/分段）或在上游压缩；同步 I/O 全部换异步 API；`console.log` 在生产用结构化日志框架且异步落盘。修完用同样的采样手段复测，验证事件循环延迟回落到基线，再谈优化收益。

**延伸考点：** 如何构造一个正则回溯攻击（ReDoS）的样例？为什么 GC 停顿也会让事件循环延迟飙升，如何从火焰图上区分 GC 与业务热点？

### Q3. forEach 里 await 没生效，请求瞬间全打出去把数据库打挂了，怎么回事？

**问题：** 代码里 `list.forEach(async (item) => { await save(item); })`，期望逐条入库，结果 1000 条记录同时发起写库，数据库连接被打满。请解释根因并给出正确写法。

**期望加分项：**
- 能讲清 `forEach` 只启动不等待的本质（回调立即返回，`await` 只作用于回调内部）
- 能给出三种层次的方案：`for...of` 串行、`Promise.all` 全并发、p-limit/信号量限并发，并说明各自适用场景
- 能给出 p-limit 或手写并发池的代码，说明并发数如何定（结合 DB 连接池大小）
- 主动考虑失败处理（allSettled）、结果顺序保持、超时控制
- 联系线上实践：批量任务场景常见这个坑

**减分项：**
- 只会说"用 for...of"，讲不清为什么
- 不知道限制并发的手段（p-limit、async 队列、信号量）
- 并发数拍脑袋，没有依据

**解答：**

根因一句话：`forEach` 对每个元素调用回调后**不等待返回的 Promise**，1000 个异步任务全部立即启动，`await` 只约束回调内部的顺序。这在 Node 单线程下表现为"看似并发执行"的密集 I/O，把数据库连接池瞬间打满。

正确写法分三个层次，按场景取舍：数据量小、对顺序有要求 → `for...of` 串行（简单但慢，N 个串行等待）；任务互相独立、目标系统扛得住 → `Promise.all` 全并发（快但要评估峰值）；绝大多数生产场景 → 限并发，用 p-limit 或自写并发池：

```js
const { default: pLimit } = require('p-limit');
const limit = pLimit(10); // 并发数参考目标库连接池上限
await Promise.all(list.map((item) => limit(() => save(item))));
// 手写信号量版本：维护 running 计数 + 等待队列，slot 释放时出队
```

几个容易踩的坑：一是失败处理——`Promise.all` 任一失败即整体失败，批量任务更推荐 `allSettled` 并记录失败项重试；二是结果顺序——并发结果与输入顺序不一致时，按索引归位再返回，别让下游错乱；三是超时——任务无超时上限会一直占用 slot，用 `Promise.race` 或 AbortController 兜底。最后，并发数不是越大越好，要跟数据库连接池（max）、对端 QPS 上限对齐，否则只是把压力换个位置爆发。

**延伸考点：** 如果任务里还有重试逻辑，信号量/并发池的 slot 什么时候归还才不会算错？并发场景下如何保证批量结果顺序稳定？

### Q4. 老项目三层嵌套回调，新需求加不动了，怎么安全重构？

**问题：** 接手一个历史项目，核心流程是三层回调嵌套，每加一个分支都要动最外层。线上稳定但改动风险大。请给出重构思路与节奏。

**期望加分项：**
- 能先讲清"为什么回调地狱难维护"：错误路径分散、无法 try/catch、流程顺序不直观
- 能给出 `util.promisify` 的正确用法与边界（多返回值回调需自定义、事件式 API 不可用）
- 有渐进式重构策略：不一次推翻，先抽纯函数、再逐层 Promise 化，每步都有测试兜底
- 能给出重构前后对比代码，错误处理集中、`finally` 清理资源
- 主动提防：重构期间行为漂移（错误码、超时语义、回调可能被调用多次）

**减分项：**
- 直接说"全改成 async/await"，讲不出边界与风险
- 不知道 promisify 的适用条件，拿到事件式 API 也硬套
- 没有测试/对照策略，重构等同于重写

**解答：**

先明确重构目标：不是炫技，而是让"错误路径收敛、流程可读、改动可控"。核心手法是把回调函数用 `util.promisify` 转成 Promise，再以 async/await 重写流程。注意边界：promisify 假定回调形如 `(err, value)`；回调多返回值（如 `fs` 某些 API）需自定义 `Symbol.for('nodejs.util.promisify.custom')`；事件式 API（如 `EventEmitter`、`stream` 的 on 事件）不能直接 promisify。

```js
// 重构前
function step1(id, cb) { fs.readFile(id, (err, data) => {
  if (err) return cb(err);
  step2(data, (err2, r) => { if (err2) cb(err2); else cb(null, r); });
}); }
// 重构后
const readFile = promisify(fs.readFile);
const step2p = promisify(step2);
async function step1(id) {
  const data = await readFile(id);
  return step2p(data); // 错误自动向上抛，由调用方统一捕获
}
```

节奏上建议"外科手术式"：先用现有测试或对照输出锁定行为基线；从最内层叶子函数开始 Promise 化，每层转换后跑一遍对照；纯函数先行抽出，让流程函数只做编排。风险点要提前识别：老回调可能被多次调用（promisify 后变成 resolve 后再次调用被忽略，行为变化）、错误码与重试语义、`finally` 中的资源释放（连接、文件句柄）在回调版本里极易遗漏。

**延伸考点：** 遇到 stream 与 EventEmitter 这类"回调+事件"混合 API 怎么 Promise 化？重构如何利用类型与测试保证行为不漂移？

### Q5. 线上进程偶发崩溃重启，日志里有 uncaughtException，怎么正确处理？

**问题：** 线上服务偶发崩溃，PM2 自动拉起了，但用户看到 502。日志显示 `uncaughtException`。请说明正确的错误处理体系与进程崩溃后的恢复策略。

**期望加分项：**
- 能讲清关键判断：`uncaughtException` 后进程状态不可信，正确姿势是记录现场 + 优雅退出 + 外部拉起（PM2/Docker restart），而不是吞掉继续跑
- 能说明 `unhandledRejection` 在 Node 15+ 默认直接崩溃，应显式兜底
- 能给出分层错误处理：主流程 try/catch、框架中间件（Express next(err)/Koa ctx.onerror/NestJS 异常过滤器）、进程级兜底
- 主动考虑：非 0 退出码（否则 supervisor 可能不重启）、日志落盘、堆栈与现场信息
- 能联系实践：兜底 handler 里只做记录与退出，不碰业务状态

**减分项：**
- 主张在 `uncaughtException` 里继续运行（经典错误观点）
- 不知道 `unhandledRejection` 的默认行为与处置
- 只有进程级兜底，没有分层错误处理概念

**解答：**

核心判断：`uncaughtException` 意味着栈展开到最顶层，调用栈、内存状态、锁、连接都可能处于不一致状态，"捕获后继续跑"只会带来更隐蔽的故障——正确的做法是**记录现场、尽快退出、由外部进程拉起**。这也是为什么 PM2/容器/K8s 的自动重启机制存在：崩溃恢复不是进程内自愈，而是快速重启 + 不丢请求（配合负载均衡与重试）。

分层设计：第一层是业务代码的 try/catch 与框架错误中间件，处理"可预期的错误"；第二层是 `unhandledRejection` 兜底——Node 15 起未处理的 rejected Promise 直接导致进程崩溃，如果这是预期外的，统一转成错误日志；第三层才是进程级兜底：

```js
process.on('uncaughtException', (err) => {
  logger.error('uncaughtException', err);        // 先落盘（异步落盘要 flush 或同步写）
  gracefulShutdown(1);                            // 停止接新请求 → 等在途(超时) → 关连接 → 非0退出
});
process.on('unhandledRejection', (reason) => { throw reason; }); // 统一转成 uncaughtException
```

实践中的坑：一是退出码——`process.exit(0)` 会让 PM2 认为"正常退出"而不重启，必须非 0；二是日志必须先 flush 再退出，否则进程死了现场丢了；三是兜底 handler 里不要再做业务清理（可能二次异常），只做最小收尾；四是与监控联动，崩溃率、重启次数要进告警。最后，任何兜底都不该成为常态路径——崩溃频发说明根因（如三方库 bug、内存爆炸、数据结构异常）没解决，兜底只是止血。

**延伸考点：** 优雅退出时在途请求怎么保证完成（超时如何设定）？Node 15 后 `unhandledRejection` 默认行为变化对存量代码有什么影响？

### Q6. 大文件导出一次性读进内存导致 OOM，怎么用流解决？

**问题：** 一个导出 2GB CSV 的接口，`fs.readFile` 后拼字符串再返回，内存直接打爆。请说明流式方案与背压（backpressure）机制。

**期望加分项：**
- 能说清 OOM 根因：整文件驻留内存 + 字符串拼接产生多份拷贝
- 能正确使用 `Readable.pipe`/`pipeline` 实现"边读边写"，并说明 pipe 如何自动处理背压（write 返回 false 时暂停读、drain 后恢复）
- 能讲清 `pipe` 与 `pipeline` 的差异：pipeline 正确传播错误并自动销毁流
- 能处理"流式生成"而非仅文件流：数据库游标 → transform → response
- 主动考虑：内存峰值如何量化、highWaterMark 调整的影响、压缩链（zlib）插入

**减分项：**
- 只会说"用 stream"，讲不出背压机制
- 不知道 pipe 的错误处理缺陷
- 内存问题只归因于"文件太大"，讲不清多份拷贝

**解答：**

先讲根因：`readFile` 把整个文件读进 Buffer，字符串拼接（如 `+=`）在 V8 里会产生多次大块拷贝，2GB 数据轻松翻几倍内存。流式方案的核心不是"用流"，而是**让内存占用与数据总量解耦、只与单批数据大小相关**。

`pipe` 之所以能防 OOM，靠的是背压：`Writable.write` 返回 `false` 表示内核缓冲已满，此时 `pipe` 会暂停源 `Readable` 的读取，等目标触发 `drain` 再恢复，形成"生产-消费"的节流闭环。手写时务必遵守这个约定，否则数据会无界堆积。注意 `pipe` 不传播错误（错误事件丢失会挂起），生产一律用 `pipeline`：

```js
const { pipeline } = require('stream');
const { createReadStream, createWriteStream } = require('fs');
const { createGzip } = require('zlib');
// 文件 → gzip → 目标，自动背压 + 错误统一回调
pipeline(createReadStream('big.csv'), createGzip(), createWriteStream('big.csv.gz'),
  (err) => err ? logger.error(err) : logger.info('done'));
```

更常见的导出场景是"数据在数据库里"：用游标分批读取（如 MySQL 流式查询、MongoDB cursor）接入 Transform 拼 CSV 行，再 `pipeline` 到 `res`。坑点：一是忘了处理 response 断连（`res.on('close')` 时要销毁源流，否则连接释放后数据库还在跑）；二是压缩链里 gzip 的背压同样靠 pipeline 协调；三是 `highWaterMark` 调大不等于内存安全，反而拉高峰值；四是对响应流做 `res.end()` 的时机要交给 pipeline 回调。

**延伸考点：** 手动实现流时，`_write` 里背压如何体现？`drain` 事件在什么时机触发、如何自测背压是否生效？

### Q7. 接口返回的中文偶尔乱码，字符串按字节截断后出现 �，怎么回事？

**问题：** 某接口把数据库文本返回给前端，偶发乱码；另有一个功能按长度截断字符串，截出来的内容有时带 `�` 替换符。请分析原因并给出正确处理。

**期望加分项：**
- 能讲清 Node 中字符串与字节的关系：UTF-8 中文 3 字节，按字节切分会切断多字节字符
- 能给出流式解码的正确姿势：`string_decoder.StringDecoder` 处理跨 chunk 的字符边界
- 能说明乱码的另外两类根因：编码声明不一致（Content-Type charset）、读取时未指定 encoding
- 截断场景能给出按字符而非字节截断的方案（Intl.Segmenter / 数组展开 / 迭代器）
- 主动考虑 GBK 等历史编码（iconv-lite）

**减分项：**
- 只会说"用 utf8"，讲不清字节与字符的关系
- 截断用 `substring` 按索引硬切
- 不知道 StringDecoder 存在的意义

**解答：**

两个问题其实同源：**Node 的 `string` 是 UTF-16 字符序列，而 I/O 边界（文件、Socket、HTTP body）是字节流**。UTF-8 下中文占 3 字节、emoji 占 4 字节，任何"按字节数切分"的操作都可能落在字符中间，切出来的半截字节解码失败就成了 `�`。同理，网络分包时一个字符的字节可能横跨两个 chunk，如果对每个 chunk 独立 `toString()`，就会在边界处产生乱码。

正确做法是用 `StringDecoder`，它内部维护残片缓冲，等后续字节补齐再输出完整字符：

```js
const { StringDecoder } = require('string_decoder');
const decoder = new StringDecoder('utf8');
let text = '';
res.on('data', (chunk) => { text += decoder.write(chunk); });
text += decoder.end(); // 处理末尾残片
```

按"长度"截断也要分清单位：想按用户感知的字符数截断，用 `Array.from(str).slice(0, n)`（按码点）、`Intl.Segmenter`（按字素簇，emoji 组合正确），而 `str.substring(0, n)` 是按 UTF-16 码元切，恰好在代理对中间就会产生 `�`。乱码排查的另外两个高发点：一是响应头 `Content-Type: text/plain; charset=utf-8` 缺失或与数据实际编码不符；二是 `fs.readFile` 不传 encoding 得到 Buffer，直接拼进字符串。历史存量系统遇到 GBK 数据，用 iconv-lite 显式转码，别指望 `buffer.toString('utf8')` 兜底。

**延伸考点：** 从 TCP 流里解析二进制协议时，怎么用 Buffer 手动组装、避免把半个包当整包解析？

### Q8. 内存持续上涨，重启就恢复，怎么系统性排查内存泄漏？

**问题：** 服务运行几天后内存缓慢爬升，重启后恢复，反复出现。请给出从"确认泄漏"到"定位对象"的完整排查流程。

**期望加分项：**
- 先区分"泄漏"与"正常 GC 锯齿"：监控 RSS/堆大小的曲线形态，锯齿状回落是正常，持续阶梯上涨才是泄漏
- 能给出抓堆快照的方法：`heapdump` 模块、`--heapsnapshot-near-heap-limit`、`clinic heapprofiler`，并说明对比多份快照找"只增不减"的对象
- 能列举常见泄漏源：全局缓存不清理、闭包误捕获、事件监听器未移除、定时器未清除、Stream 未销毁、第三方模块内部缓存
- 能区分 V8 堆内存与堆外内存（Buffer、原生模块），Buffer 泄漏抓 heapdump 是看不到的
- 生产环境的坑：抓快照的触发时机、对线上性能的影响

**减分项：**
- 一上来就"加内存"或"重启"，没有定位
- 只会背"闭包泄漏"之类概念，给不出工具链
- 不知道 Buffer/原生内存是堆外的

**解答：**

第一步：**确认是不是真泄漏**。把 `process.memoryUsage()` 的 heapUsed 与 RSS 按分钟级采样画曲线：周期性回落到基线是正常 GC；单调爬升不回落才是泄漏。注意区分——RSS 涨而 heapUsed 平稳，说明泄漏在堆外（Buffer 池、原生模块），此时 heapdump 看不到，要用 `--trace_gc`、`process.report` 或 `napi` 相关工具看。

第二步：抓多份堆快照做对比。常用命令是运行时触发 `heapdump.writeSnapshot()`，或启动时加：

```bash
node --heapsnapshot-near-heap-limit=3 --heapsnapshot-signal=SIGUSR2 app.js
```

在低峰期抓快照 A，运行 24h 后再抓 B，用 Chrome DevTools 的 Memory 面板或 `devtools` CLI 加载两份，重点看**分配统计的差值**：增长最快的构造函数与保留路径，顺着引用链找谁持有它——通常是某个模块级缓存、事件监听器（对长生命周期对象注册的）、未 `clearInterval` 的定时器、闭包捕获了大型局部变量。

第三步：修复后验证。最常见的两类实战坑：一是"监听器泄漏"——每次请求给全局 EventEmitter（或数据库连接、进程级 bus）注册监听而不移除，对象被 emitter 引用住；二是 Buffer 泄漏——`Buffer.alloc` 不释放、stream 未 destroy、`zlib` 压缩实例未 `end`，这类问题 heapUsed 正常但 RSS 爬升。排查工具链里 `clinic heapprofiler` 能自动对比多个采样点并给出增长对象，效率比手抓快照高，适合快速定位。

**延伸考点：** `heapdump` 抓出来的快照怎么找"无根对象"？为什么说 WeakRef/FinalizationRegistry 可以解决一部分缓存泄漏，但别依赖它做关键清理？

### Q9. 接口 CPU 占比高、延迟大，怎么找到热点函数并优化？

**问题：** 压测发现某接口 CPU 占用 80% 以上，延迟远高于同类接口。请说明性能剖析的完整流程与工具选择。

**期望加分项：**
- 能给出完整流程：压测复现 → 采样 → 分析热点 → 优化 → 复测对比
- 熟悉至少两条工具链：`--cpu-prof` + `--prof-process`、`--inspect` + Chrome DevTools、`clinic doctor/flame`、0x
- 能结合火焰图解读：宽条是热点、纵向看调用链、留意 GC 占比
- 知道 async/await 会"抹平"调用栈导致火焰图失真，如何用 async_hooks/AsyncLocalStorage 辅助
- 主动考虑压测与线上的差异（数据分布、缓存命中率、GC 停顿）

**减分项：**
- 只会"代码走读猜热点"，没有采样证据
- 不知道 Node 自带 profile 能力，只依赖第三方
- 解读火焰图只会看"最宽"，讲不清调用关系

**解答：**

思路是"先测量、再动手"，别靠代码走读猜。标准流程四步：**复现（压测脚本打满流量）→ 采样 → 分析 → 优化并复测**。最小侵入的采样方式是 V8 自带的 CPU profiler，无需改代码：

```bash
node --cpu-prof --cpu-prof-dir=./prof app.js   # 运行一段时间（配合压测）
node --prof-process ./prof/isolate-*.log > profile.txt
```

看输出里按 self time 排序的函数，占比高的就是热点。更直观的是火焰图：`clinic flame -- node app.js` 或 `0x app.js`，横向宽度代表耗时占比，纵向是调用链，一眼定位"又宽又深的调用路径"。交互式定位可以用 `node --inspect` 配合 Chrome DevTools Performance 面板录制，适合排查"某次请求"级别的热点。

```bash
npx clinic doctor -- node app.js   # 一键出诊断报告，含事件循环延迟、GC、CPU 分析
```

几个经验坑：一是 **async/await 会让火焰图失真**——异步函数的栈在 await 处断裂，热点显示在 `processTicksAndRejections` 之类的内部函数上，配合 `async_hooks` 追踪或 `AsyncLocalStorage` 记录调用来源才能还原业务归属；二是 GC 占比高要单独看——`--trace-gc` 确认，优化方向是减少瞬时大对象分配，而不是优化算法；三是压测数据要贴近线上分布（缓存命中率、参数分布），否则优化的可能是假热点。修完务必用同一压测脚本复测，量化对比 CPU 占比与 P99，别只看平均延迟。

**延伸考点：** 火焰图里出现大段 `v8::internal::...` 或 GC 相关帧时，说明什么问题？如何用 `--inspect` 在线对生产实例做短时采样而不重启？

### Q10. 单进程只用一个 CPU 核，怎么用 Cluster/PM2 榨干多核？有哪些坑？

**问题：** 压测发现服务单进程 CPU 只能到 100%（一核），吞吐上不去。请说明 Node 多进程方案（Cluster/PM2）的原理、配置与常见陷阱。

**期望加分项：**
- 能讲清 Cluster 原理：master fork 多个 worker，共享端口（内核/主进程分发），IPC 通信，worker 崩溃自动拉起
- 能给出 PM2 的 cluster 配置：`-i max`、`--max-memory-restart`、`reload` 与 `restart` 的区别（滚动重启 vs 全停）
- 能主动指出关键陷阱：内存不共享（session 需 Redis）、WebSocket/长连接需要 sticky（PM2 默认不支持，需 ip_hash 或网关层处理）、进程数 ≠ 核数（避免超卖）
- 能联系实践：多进程下定时任务重复执行（leader 选举或独立进程）
- 对 stateless 改造有认识：本地缓存、内存锁在多进程下失效

**减分项：**
- 以为 PM2 cluster 是"魔法"，讲不清原理
- 不知道 sticky 问题，WebSocket 上线就崩
- 忽略定时任务重复执行、内存共享等架构影响

**解答：**

先讲原理：Cluster 由主进程（master）`fork` 出 N 个 worker，所有 worker 监听同一端口（Node 依靠 `SO_REUSEPORT` 或主进程转发），请求被分摊到各进程，单进程的 CPU 上限随之解除；master 还负责 worker 崩溃重启与 IPC 通信。最省心的运维方式是 PM2：

```bash
pm2 start app.js -i max --max-memory-restart 500M   # -i max = CPU 核数
pm2 reload app                                      # 滚动重启：逐个拉起新 worker，不停服
```

真正的难点在"无状态化"。首当其冲是**内存不共享**：session、内存缓存、进程内锁在 cluster 下各自独立，必须外置到 Redis/DB；其次**定时任务重复执行**——N 个 worker 会同时跑 N 份 cron，要么抽成独立进程，要么用分布式锁（如 Redis SETNX）保证单点；第三是 **sticky 问题**：WebSocket/SSE 这类长连接如果被分发到不同 worker，服务端状态就对不上了，PM2 cluster 默认随机分发，需要在前置层按 IP 哈希（nginx `ip_hash`）或让网关做 sticky，否则连接必断。

```js
// 手动 cluster 时核心骨架
const cluster = require('cluster');
if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  cluster.on('exit', (w) => cluster.fork()); // 崩溃自动拉起
} else { require('./app'); }
```

常见坑还有：进程数开得比核数多反而引入上下文切换开销（CPU 密集场景尤其明显）；`pm2 restart` 是全停全起（有秒级断服），线上要用 `reload`；`--max-memory-restart` 触发的是"重启止损"而非修复，频繁重启要去查泄漏根因。

**延伸考点：** cluster 模式下优雅退出与滚动发布怎么配合（先摘流量再关 worker）？多进程下分布式限流的计数为什么不能放本地？

### Q11. 一段 CPU 密集计算卡死主进程，用 child_process 还是 worker_threads？

**问题：** 图像处理/加解密这类 CPU 密集任务一跑，主进程就卡住，接口全部超时。请说明 child_process 与 worker_threads 的选型与工程化要点。

**期望加分项：**
- 能给出选型判断：同进程内做 CPU 密集计算、要频繁传数据 → `worker_threads`；要强隔离（崩溃不影响主进程）、调外部程序 → `child_process`
- 能讲清 worker 的关键能力：`parentPort` 消息、`transferable` 转移 ArrayBuffer 避免拷贝、`SharedArrayBuffer` 共享内存
- 知道"线程池"的必要性：worker 创建成本高，用 piscina 或自写池复用
- 能讲清 spawn 与 exec/fork 的差异（exec 有 maxBuffer 输出上限、fork 专门跑 Node 脚本）
- 主动考虑：线程数别超核数、消息大小与序列化开销、异常/超时处理

**减分项：**
- 两个概念分不清，选型拍脑袋
- 不知道 transferable/SharedArrayBuffer 的存在，只会"postMessage 传 JSON"
- 每次都 new 一个 worker，不知道池化
- 忽略 worker 内异常处理与超时兜底

**解答：**

先给判断依据：**要看"计算与数据"的耦合方式和隔离需求**。CPU 密集且数据就在本进程内、需要高频往返（比如把大数组分片算完汇总）→ `worker_threads`：同一进程、`postMessage` 通信，还能用 `transferable` 转移 `ArrayBuffer`（转移零拷贝）或 `SharedArrayBuffer` 直接共享内存，避免大数据量序列化开销。而需要**崩溃隔离**（worker 崩了不能让主进程陪葬）、要调用外部可执行文件（ffmpeg、Python 脚本）→ `child_process`，子进程独立崩溃、主进程毫发无损，但通信要走 IPC 序列化，大数据量下开销明显。

```js
// worker_threads 最小骨架
const { Worker, parentPort } = require('worker_threads');
const w = new Worker(`
  const { parentPort } = require('worker_threads');
  parentPort.on('message', (data) => {
    const result = heavyCompute(data);
    parentPort.postMessage(result);   // 大结果可改用 transferList 转移 buffer
  });
`, { eval: true });
w.postMessage(bigData);
w.on('message', onResult);
```

工程化上有三个高频坑：第一，**worker 必须池化**——每次 new Worker 有固定启动开销（新 V8 isolate），用 piscina 或自写池（固定 N 个 worker + 任务队列）复用；第二，**线程数量要克制**——worker 数 ≈ CPU 核数（或核数-1），开多了反而上下文切换；第三，**异常与超时**——worker 抛错只触发 'error' 事件，主进程要捕获并重建；计算卡死（死循环）没有默认超时，要自己包超时逻辑。选 child_process 时注意 `exec` 有 `maxBuffer`（默认 1MB）输出上限，超了直接报错，长输出用 `spawn` 流式读。

**延伸考点：** `transferable` 转移后原侧数据变成什么状态？`SharedArrayBuffer` 并发写如何保证原子性（Atomics）？

### Q12. 热点 key 缓存一过期，请求全部打到数据库，怎么防缓存击穿？

**问题：** 某个热点商品详情 key 缓存过期瞬间，成千上万的请求同时回源查询，数据库 CPU 飙升。请给出系统性解法与取舍。

**期望加分项：**
- 能区分并说清三个概念：击穿（热点 key 过期）、雪崩（大量 key 同时过期）、穿透（查不存在的数据）
- 能给出至少两种方案并比较：互斥锁重建（分布式锁）、逻辑过期+异步刷新、singleflight 请求合并（同 key 只放一个请求去 DB）
- 能给出 singleflight 的核心实现思路（Map<key, Promise>）或代码
- 主动考虑：锁的过期时间与重建耗时、失败后如何兜底、分布式环境的一致性
- 结合实践谈雪崩防护：过期时间加随机抖动

**减分项：**
- 三个概念混为一谈
- 只有"加锁"一句话，讲不清锁粒度、锁过期、降级
- 不知道 singleflight 思想，也讲不出"合并请求"的方案
- 忽略"查询不存在的数据"这类穿透问题

**解答：**

先分清问题：**击穿**是单个热点 key 失效瞬间流量直击 DB；**雪崩**是大批 key 同时失效（或缓存服务整体不可用）；**穿透**是查询根本不存在的 key，每次都回源。你的场景是击穿，核心思路是"同一时刻只允许一个请求重建缓存，其余请求等待或复用"。

方案一：互斥锁（分布式场景用 Redis `SETNX` + 过期时间）。拿到锁的请求查 DB 回填，其余请求短暂等待后直接读缓存；锁过期时间要大于重建耗时，否则会重复击穿。方案二（更推荐用于热点数据）：**逻辑过期**——缓存里存 `{data, expireAt}`，读到"逻辑过期"的数据直接返回旧值并触发异步刷新，用户无感知，且天然免锁。方案三：**singleflight（单飞/请求合并）**——进程内按 key 合并并发请求，同 key 只有一个真正执行：

```js
const inflight = new Map();
async function getProduct(id) {
  const cached = await redis.get(`p:${id}`);
  if (cached) return cached;
  if (inflight.has(id)) return inflight.get(id);      // 已有请求在途，直接等它的结果
  const p = db.query(`SELECT * FROM product WHERE id = ?`, [id]).then(async (row) => {
    await redis.set(`p:${id}`, JSON.stringify(row), 'EX', 600 + Math.random() * 300);
    return row;
  }).finally(() => inflight.delete(id));              // 无论成败都清理占位
  inflight.set(id, p);
  return p;
}
```

坑点提醒：singleflight 是进程内生效，多实例要靠分布式锁补充；合并后要处理"DB 查询失败"——占位必须 finally 清理，否则后续请求全部永久等待；缓存回填统一加随机抖动（`600 + random(300)`）防雪崩；穿透问题要用"空值缓存"（把 null 也缓存短时间）或布隆过滤器拦截。方案选择上，读多写少的热点数据优先"逻辑过期+异步刷新"，一致性要求高时回退到互斥锁，singleflight 适合短时突发。

**延伸考点：** 多实例部署时 singleflight 失效，分布式锁的"加锁-重建-释放"时序如何设计才能避免锁误删？缓存与 DB 一致性怎么取舍（先更库还是先删缓存）？

### Q13. EventEmitter 用得多，内存一直涨，监听器越积越多，怎么办？

**问题：** 业务里大量使用 EventEmitter 做事件驱动，线上内存缓慢上涨，`warning` 日志提示 MaxListenersExceededWarning。请定位原因并给出工程化治理方案。

**期望加分项：**
- 能指出泄漏机理：每次注册监听器都让 emitter 持有回调引用，对象不销毁、回调不释放
- 能说清默认 `maxListeners = 10` 警告的含义与定位手段（`--trace-warnings`）
- 能给出治理规范：`once` 优先、用后 `removeListener`、AbortController 统一取消、订阅生命周期与对象生命周期绑定
- 能讲清 `error` 事件语义：无监听器的 error 事件会直接抛异常导致进程崩溃
- 能谈事件驱动架构：事件命名、版本化、异步错误通道、事件溯源/编排的取舍

**减分项：**
- 只会"把 maxListeners 调大"掩盖问题
- 不知道 error 事件无监听的崩溃行为
- 只讲 API 不讲生命周期治理

**解答：**

先说机理：`EventEmitter` 泄漏的本质是**引用链**——emitter 持有每个回调的引用，而回调（闭包）往往又持有业务对象。典型场景：在请求处理函数里给"全局事件总线/数据库连接/长生命周期模块"注册监听，请求结束后忘记移除，于是每次请求都新增一个回调节点，且把请求上下文整个闭包住，内存只增不减。`MaxListenersExceededWarning` 是 V8 给到 10 个监听器的默认告警，**它是症状不是病根**，用 `--trace-warnings` 能看到告警触发的注册堆栈，能快速锁定泄漏点。

治理三板斧。第一，**注册即管理**：优先 `once`；手动注册的回调保存句柄，对象销毁或流程结束时 `removeListener`；复杂生命周期用 `AbortController`/`AbortSignal` 统一取消。第二，**订阅与对象同生命周期**：谁创建谁销毁——比如数据库连接池对象内部的监听随池销毁，业务对象订阅外部事件要显式 `off`，别依赖 GC（emitter 引用着回调，根本回收不了）。

```js
const handler = (msg) => { ... };
bus.on('order:created', handler);
// 业务结束时：
bus.off('order:created', handler);      // 必须显式移除，否则泄漏
// 用 AbortController 统一管理多个订阅
const ac = new AbortController();
bus.on('order:created', fn, { signal: ac.signal });
ac.abort(); // 一次性清理
```

第三，**error 事件是硬约束**：`EventEmitter` 规定发射 `error` 时若无监听器，直接 `throw` 让进程崩溃——这是防呆设计，要求所有自定义 emitter 必须带 error 通道或兜底监听。架构层面，事件驱动要立规范：事件名全局收敛（前缀 + 领域，如 `order:paid`）、携带版本（`order:paid.v2`）、异步事件的错误要落日志并进监控，事件编排（Saga）要设计补偿，别让事件流变成隐式调用链。

**延伸考点：** `EventEmitter` 与 Stream 的 error/close 事件语义有什么共同点？事件驱动与消息队列（MQ）怎么取舍，本地事件何时必须升级为 MQ？

### Q14. 新项目选框架：Express、Koa、NestJS 怎么选？洋葱模型怎么理解？

**问题：** 团队要启动一个中大型后端服务，成员熟悉 JS/TS，要求可测试、可维护。请给出框架选型分析与关键机制（洋葱模型）的实践理解。

**期望加分项：**
- 能对比三者定位：Express 生态成熟但回调式/弱约束；Koa 洋葱模型 + async/await + ctx 封装；NestJS 结构化（DI/装饰器/模块化）、类型安全、适合中大型
- 能讲清洋葱模型本质：`next()` 之前的代码在"进入"时执行、之后的代码在"返回"时执行（先入后出），中间件按注册顺序嵌套
- 能用代码说明日志/计时/错误捕获中间件的写法，以及中间件顺序对错误处理的影响
- 能给出选型决策依据：项目规模、团队背景、是否需要框架内置能力（校验、鉴权、OpenAPI）
- 能指出实践坑：Koa 的 ctx/body 解析、Express 4 错误中间件必须 4 个参数、NestJS 抽象成本

**减分项：**
- 只会报框架名，说不出各自适用边界
- 洋葱模型只能画图，写不出应用它的中间件
- 不知道中间件注册顺序会改变错误处理行为

**解答：**

先给选型判断：**小团队、快速出活、生态优先 → Express**（生态最大、资料最多、Express 5 已原生支持 Promise 错误）；**追求简洁的 async 中间件模型、团队熟悉函数式 → Koa**（洋葱模型最纯粹，但路由/body 解析都要自己拼装）；**中大型项目、多人协作、要可测试性与架构约束 → NestJS**（DI 容器、装饰器、模块边界、内置校验/序列化/OpenAPI，配上 TypeScript 是加分组合）。选型本质是"约束程度"的取舍：NestJS 用框架约束换工程一致性，代价是概念多、上手曲线陡。

洋葱模型的核心是一句话：**`next()` 之前的代码在请求进入时执行，之后的代码在响应返回时执行，整体先入后出**。这让"横切关注点"（日志、计时、鉴权、错误捕获）以嵌套顺序生效：

```js
// Koa 中间件：外层计时，内层业务
app.use(async (ctx, next) => {
  const start = Date.now();
  try {
    await next();            // 进入内层，业务/后续中间件在这里执行
  } finally {
    ctx.set('X-Response-Time', `${Date.now() - start}ms`);
  }
});
app.use(async (ctx) => { ctx.body = await service.getData(); });
```

实践中的坑集中在顺序与边界：一是**错误中间件必须放在最外层**（最先注册），否则内层抛错不会经过它；Express 4 的错误中间件签名必须是 `(err, req, res, next)` 四个参数，少一个会被当成普通中间件；二是 Koa 的 `ctx` 是对 req/res 的封装，别和 Express 的 req/res 混着写；三是 NestJS 里中间件（middleware）、守卫（guard）、拦截器（interceptor）、管道（pipe）职责不同，用错了会在错误处理时序上出问题。选型后最该做的是把错误处理、日志、鉴权三个中间件的注册顺序在项目文档里钉死。

**延伸考点：** Koa 中间件里 `await next()` 抛错时，`finally` 与 catch 的执行顺序如何？NestJS 的拦截器与中间件在洋葱模型里的位置有什么区别？

### Q15. 大促流量暴涨，接口撑不住，怎么设计限流？

**问题：** 大促期间流量是平时的 20 倍，下单接口出现雪崩式超时。请设计一套限流方案，并说明单机与分布式的取舍。

**期望加分项：**
- 能区分并实现基础算法：令牌桶（允许突发）与漏桶/固定窗口（平滑）、滑动窗口（边界精确）
- 能给出单机方案（内存令牌桶，如 rate-limiter-flexible 或自写）与全局限流（Redis + Lua 原子脚本）的适用边界
- 能说明限流维度：按用户/IP 防滥用 vs 按接口/全局保系统，两者要同时存在
- 能讲清超限后的反馈：429 + Retry-After 头、客户端指数退避，而不是静默丢弃
- 能主动考虑限流器自身的可靠性：Redis 故障时 fail-open 还是 fail-closed，取舍得说清
- 有真实取舍经验：限流阈值怎么定（压测得出水位）、误伤如何规避

**减分项：**
- 只会背"令牌桶"概念，写不出实现，也不知道滑动窗口为何更准
- 单机方案套到多实例上，计数全乱
- 只限流不降级，忽略"限流后业务怎么办"

**解答：**

先立框架：限流要分层——**入口层（网关/nginx）做全局限流保命，应用层做业务维度限流保公平**。算法选择上：令牌桶允许突发（桶容量 + 速率），适合接口限流；固定窗口有边界毛刺（窗口临界点可放 2 倍流量），滑动窗口用时间片计数消除毛刺；漏桶输出恒速，适合保护下游。单机场景用内存实现即可：

```js
// 令牌桶最小实现（单机可用）
function tokenBucket(rate, capacity) {
  let tokens = capacity, last = Date.now();
  return () => {
    tokens = Math.min(capacity, tokens + (Date.now() - last) / 1000 * rate);
    last = Date.now();
    if (tokens < 1) return false;
    tokens -= 1;
    return true;
  };
}
```

多实例部署时，单机计数必然失真（每台各限各的），需要分布式限流：**Redis + Lua 脚本保证"检查-扣减"原子性**，滑动窗口用 ZSET 或时间片 key 计数。限流维度要分层设：按 IP/用户（防单点刷爆）、按接口维度（保核心链路）、全局水位（保系统不雪崩）。

三个工程要点：第一，**阈值必须来自压测**——根据历史峰值与系统水位定速率，而不是拍脑袋；第二，**超限反馈要明确**——返回 `429` 与 `Retry-After`，客户端按指数退避重试，服务端静默丢弃会让调用方无限重试、雪上加霜；第三，**限流器自身要可靠**——Redis 抖动时是 fail-open（放行，保可用性）还是 fail-closed（拒绝，保系统），取决于业务容忍度，一般核心链路选 fail-open 并告警。最后补一句：限流只是保命手段，配合**降级**（非核心功能关停）、**熔断**（依赖故障快速失败）才构成完整的过载保护体系。

**延伸考点：** 滑动窗口 vs 令牌桶在"突发流量"下的行为差异是什么？网关限流与应用层限流的阈值如何协同（上游严格、下游宽松）？

### Q16. 老 JS 项目想迁 TypeScript，怎么控风险、怎么用才不算白用？

**问题：** 团队决定把运行多年的 JS 服务迁到 TypeScript，但又怕"类型写了一大堆，线上问题一个没少"。请给出迁移策略与类型落地的正确姿势。

**期望加分项：**
- 能给出渐进迁移路径：`allowJs` 混合编译、按模块边界逐个迁移、每个模块有行为对照
- 能讲清 tsconfig 关键配置：`strict: true`、`esModuleInterop`、`module`/`target` 对产物形态（CJS/ESM）的影响
- 能区分"编译期类型"与"运行时校验"：类型不校验运行时数据，接口入参/DB 读出的数据要配 zod/joi 校验
- 能说明生产部署形态：tsc 编译产物 + sourcemap，而不是用 ts-node 跑生产
- 主动提及坑：`any` 泛滥红线（eslint 禁 no-explicit-any）、装饰器是实验特性、JSON/Redis 序列化后类型失真

**减分项：**
- 只谈"类型安全"的好处，给不出迁移节奏
- 以为类型定义了就校验了运行时数据
- 用 ts-node 直接上生产，或不知道 sourcemap 的作用

**解答：**

核心判断：TS 的价值不在"有类型"，而在**把约束建在模块边界上**——让接口契约、数据模型、三方库封装成为强制检查点；用不好（全 `any`、类型与运行时两层皮）就是纯成本。迁移节奏上建议"自底向上、按边界切"：tsconfig 开 `allowJs` 与旧 JS 共存编译，先给核心数据模型和对外接口定义类型，再逐个模块迁移并补齐测试，每步保持可发布状态。关键配置：

```jsonc
{
  "compilerOptions": {
    "strict": true,            // 必开，否则 any 泛滥
    "esModuleInterop": true,   // 避免 import 兼容性坑
    "module": "commonjs",
    "target": "es2020",
    "sourceMap": true,
    "outDir": "dist"
  }
}
```

两个最容易踩的认知坑。一是**类型不等于校验**：TS 类型编译后消失，接口入参、DB/Redis 读出的数据随时可能是脏数据。正确姿势是边界校验——HTTP body、RPC 入参用 zod 定义 schema，运行时校验后再进业务；类型推导与 zod schema 共用一处定义，避免两套。二是**生产部署形态**：用 `tsc` 编译产物启动（`node dist/main.js`），配 `sourceMap` 让线上堆栈映射回源码；`ts-node` 适合本地开发，上生产启动慢、内存高。装饰器（NestJS 依赖）目前仍是实验特性，升级 Node 版本时要留意行为变化。

迁移中要立红线：eslint 禁止 `no-explicit-any`（确需时 `unknown` + 收窄）、禁止 import 路径绕过类型检查、CI 里 `tsc --noEmit` 当门禁。工程上把"类型、校验、文档"三位一体（schema 即文档），TS 才算真正落地。

**延伸考点：** `JSON.parse` 的结果为什么是 `any`？如何用 zod 让"反序列化 + 校验 + 类型推导"一步到位？ESM 与 CJS 混用时 TS 的 `moduleResolution` 怎么配？

### Q17. 容器里 Node 服务频繁"被杀"，优雅退出怎么做？Docker 部署有哪些坑？

**问题：** 服务容器化后，发布/扩容时频繁出现请求中断、连接被切断，日志显示进程收到 SIGTERM 后立即退出。请说明优雅退出的完整实现与容器部署要点。

**期望加分项：**
- 能讲清 SIGTERM 处理流程：停止接收新请求（`server.close()`）→ 等在途请求完成（带超时兜底）→ 关闭 DB/Redis 连接 → `process.exit(0)`
- 知道 Node 默认收到 SIGTERM 直接退出，以及未处理时在途请求被截断的后果
- 能指出 Docker 里 **PID 1 陷阱**：node 直接作为 PID1 不处理/不转发信号，需 `tini` 或 `--init`，否则 SIGTERM 根本到不了应用
- 能给出健康检查设计：`/healthz`（存活）与 `/readyz`（就绪）分离，readiness 摘流量、liveness 触发重启
- 部署优化：`npm ci --omit=dev`、多阶段构建、`NODE_ENV=production`

**减分项：**
- 只知道监听信号，不知道 PID1 陷阱
- `server.close()` 后无限等，没有超时强杀兜底
- 健康检查只有 liveness，发布时流量照样打进来

**解答：**

先讲根因：Node 收到 `SIGTERM` 的默认行为是**立即退出**，已建立的连接、在途请求全部被截断。容器编排（K8s 滚动更新、Docker stop）都会先发 SIGTERM 再等宽限期强杀，所以优雅退出不是"要不要做"，而是"不做必现问题"。标准流程：

```js
const server = http.createServer(app);
server.listen(3000);

async function shutdown(signal) {
  logger.info(`received ${signal}, draining...`);
  server.close(async () => {              // 停止接收新连接，等在途请求完成
    try {
      await db.pool.end(); await redis.quit();   // 最后关依赖连接
      process.exit(0);
    } catch (e) { process.exit(1); }
  });
  setTimeout(() => {                       // 兜底：宽限期到了还没结束，强杀
    logger.error('graceful shutdown timeout, force exit');
    process.exit(1);
  }, 10_000).unref();                      // unref 避免定时器阻塞退出
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

两个高频坑。一是 **Docker 的 PID1 问题**：容器里 `CMD ["node", "app.js"]` 时 node 是 PID1，而 Docker 只会把信号发给 PID1，Node 对 PID1 场景的信号处理与常规进程不同，很可能收不到 SIGTERM。解决：`CMD ["node", ...]` 配合 `--init`（Docker 自动注入 tini）或镜像里显式装 `tini` 做 init 进程转发信号。二是 **关闭时序**：先关 server 再关连接池/Redis，顺序反了会导致在途请求执行到一半拿不到连接；`server.close()` 在 K8s 下还要配合 `terminationGracePeriodSeconds` 留足宽限。

健康检查建议拆两个端点：`/healthz`（liveness，探活用）与 `/readyz`（readiness，返回 503 摘流量）。readiness 失败时 K8s 把实例从 Service 摘掉，发布流程里"先就绪探测通过再放流量"才能做到零中断。部署层面：`npm ci --omit=dev` 装生产依赖、多阶段构建（builder 装全量 → runtime 只拷产物）、`NODE_ENV=production` 让 Express 等框架关闭开发模式开销。

**延伸考点：** readiness 探针在依赖（DB/Redis）故障时该返回什么？WebSocket 长连接下优雅退出如何处理"等不完的连接"（超时策略）？

### Q18. 下单接口 P99 要压到 50ms，现在 100ms，怎么着手优化？

**问题：** 业务要求下单接口 P99 ≤ 50ms，当前压测 P99 约 100ms。请给出从量化拆解到优化落地、再到验证的完整方法论。

**期望加分项：**
- 先量化拆解再动手：把链路各环节（网关、网络、应用计算、DB、Redis、三方调用）耗时分别测出来，找到占比最大项
- 能给出性能指标意识：P99 与平均延迟的差异（尾部延迟由 GC、抖动、慢请求主导）
- 能列举针对性手段：缓存热点、N+1 改批量、连接池调优、减少序列化、异步化削峰、避免事件循环阻塞
- 每步优化都有"前后对比"证据，而不是凭感觉
- 能说明压测与线上的差异：缓存命中率、数据分布、GC 停顿

**减分项：**
- 上来就"加缓存""加机器"，讲不出证据链
- 只优化平均延迟，P99 没变化（没关注尾部）
- 优化做完不验证，或验证方式与线上差异大

**解答：**

方法论一句话：**先测量、再拆解、单点优化、逐个验证**，P99 是"全程最慢的 1%"，它由链路中最慢的一段决定，必须量化出"慢在哪"。第一步是拆解耗时：压测时在应用里用 `AsyncLocalStorage` 给每个请求打各环节耗时（网关到达→鉴权→库存查询→DB 事务→序列化→返回），统计每个环节的 P50/P99。常见结论：DB 慢查询（缺索引、N+1）、事件循环阻塞（见 Q2）、序列化大对象、依赖连接池排队——**每一类对应完全不同的解法，不测量就动手等于盲人摸象**。

```js
// 环节打点思路：AsyncLocalStorage 贯穿一次请求，出口统一上报
const als = new AsyncLocalStorage();
async function withTrace(fn) { return als.run({ steps: [] }, fn); }
function mark(step) { als.getStore()?.steps.push([step, Date.now()]); }
// 统计时按 step 求 P50/P99，找到最慢环节
```

第二步按"收益/成本"排序优化，常见手段与适用场景：热点数据加 Redis 缓存（Q12）；N+1 查询合并为 `IN` 或 JOIN；连接池 max 调大或复用连接（Q19）；大 JSON 响应裁剪字段、用 `fast-json-stringify`；写操作异步化（MQ 削峰）；同步计算挪 worker；慢 SQL 加索引。第三步是**验证闭环**：同一压测脚本（autocannon/wrk，固定并发与数据分布）在优化前后对比 P50/P99/P999，P99 没动就说明优化点不在尾部链路上，继续拆。

三个容易被忽略的坑：一是**只看平均延迟**——平均可能被 99% 的快请求拉低，P99 不动等于没优化；二是 **GC 停顿**——`--trace-gc` 看下 major GC 是否频繁，堆太大时 P99 会被 GC 拉爆，调 `--max-old-space-size` 或减少瞬时大对象；三是压测数据要模拟线上分布（热点商品集中、缓存命中率接近），否则优化的假热点。

**延伸考点：** P99 优化到 50ms 后，P999 反而变差了，可能是什么原因？如何用"延迟直方图"而非均值来汇报性能优化结果？

### Q19. 高并发下数据库连接报 ETIMEDOUT，连接池怎么配置和治理？

**问题：** 高峰期数据库偶发 `ETIMEDOUT` / `connection timeout`，排查发现应用与 DB 之间连接数被打满、大量请求排队。请说明连接池配置要点与连接泄漏治理。

**期望加分项：**
- 能讲清连接池核心参数及互相制约：`max`（上限，过大打爆 DB）、`min`/`idleTimeout`（空闲回收）、`connectionTimeout`/`acquireTimeout`（排队等待上限）、`queueLimit`
- 能指出泄漏场景与修复：手动 `getConnection` 未 `release`、事务异常未回滚未归还，正确做法是 `pool.query` 自动归还或 `try/finally` 释放
- 能给出连接池与并发限流的配合：限流并发（Q3）要低于池上限，否则排队堆积
- 能讲清超时与重试策略：等待超时要快速失败并告警，别无限排队
- 主动考虑：连接被服务端/网络中断后池中"陈旧连接"的处理（自动重连、主动探测）

**减分项：**
- 只会"调大 max"，不讲副作用
- 不知道 `getConnection`/`release` 与自动归还的区别，泄漏不会排查
- 池满了只会无限等待，没有超时机制

**解答：**

先讲本质：连接池是"复用"与"总量控制"的平衡。`max` 太小 → 高峰期排队；`max` 太大 → 把 DB 连接数打爆（数据库 `max_connections` 是硬上限），两者都会表现为超时，必须先看清是哪种。配置上三个参数联动：`connectionTimeout` 是获取连接的等待上限（超时即报 `ETIMEDOUT` 并快速失败），`idleTimeout` 回收空闲连接，`min` 保持常驻连接数。典型的治理配置（以 mysql2/pg 为例）：

```js
const pool = mysql.createPool({
  host, user, password, database,
  connectionLimit: 20,          // max：结合 DB max_connections 与实例数反推
  queueLimit: 50,               // 排队上限，超过直接报错而不是无限等
  acquireTimeout: 3000,         // 获取连接等待上限
  idleTimeout: 60000,
  enableKeepAlive: true,        // 对抗网络中间件切断空闲连接
});
await pool.query(sql, params);  // 自动取还，最安全
```

泄漏是超时的头号元凶：`pool.getConnection()` 拿到的连接**不会自动归还**，如果漏了 `release()`（尤其是事务中途抛异常、提前 return），连接就被永久占住，池很快被打空。规范：能用 `pool.query` 就不用 `getConnection`（后者必须 `try/finally` 保证 release）；事务里 `connection.beginTransaction()` 后，异常分支先 `rollback` 再 `release`，顺序反了连接带着未提交事务归还，会污染下一个请求。

工程上的三个坑：一是**连接池上限要与并发限流联动**——业务并发 500、池只有 20，排队必然超时，要么调池要么限并发，两者要成组设计；二是**陈旧连接**——DB 重启、网络闪断后池里全是死连接，靠 `enableKeepAlive` + 连接前 `ping`/重连逻辑，或让驱动自动剔除；三是"多个池实例"问题——每个模块各建一个池、或每次请求 new 池，等于没池化，池必须全局单例。最后，超时参数别设太大，`acquireTimeout` 3s 拿不到就该快速失败并告警，让用户快速重试而不是默默挂起。

**延伸考点：** 事务里拿到连接后，为什么不能把 `release` 放在 `finally` 之前？连接池打满时，应该先排查"泄漏"还是先调参数？

### Q20. 大促晚高峰订单服务突然大面积超时，作为负责人怎么排障？（开放题）

**问题：** 晚高峰订单服务报障：错误率、延迟同时飙升，用户开始流失。没有现成的答案，请完整描述你的排障思路、优先级与每一步的证据获取方式。

**期望加分项：**
- 有清晰的优先级：先止血（扩容/限流/降级/回滚）→ 再定位 → 修复验证 → 复盘，且能说明为什么"先止血"
- 能给出分层排查路径：入口（网关/负载均衡）→ 应用（CPU/内存/GC/事件循环延迟/错误日志）→ 依赖（DB 慢查询/连接/Redis/三方接口），层层排除
- 有证据意识：每一步结论来自监控数据、日志、链路追踪，而不是猜测
- 能区分"应用问题"与"依赖问题"：应用 CPU 高 vs DB 慢查询 vs 上游超时，现象不同排查方向完全不同
- 有复盘意识：时间线、根因、改进项（监控补齐、容量规划、预案演练）
- 联系实际：见过或处理过类似事故，能说具体手段（告警面板看哪些指标）

**减分项：**
- 一上来就"看代码"，没有全局判断
- 只会说"查日志"，讲不出分层与优先级
- 没有止血意识，全程"研究式排障"
- 复盘只写根因，没有改进项

**解答：**

开放性题考察的是**方法论与现场判断力**，不是标准答案。完整框架分四步，且顺序有讲究：

**第一步：止血优先于定位。** 大面积超时时第一要务是控制影响面，不是找根因。按影响面从大到小选手段：降级（关停非核心功能）、限流（保核心链路，见 Q15）、扩容（加实例，前提是瓶颈不在 DB/无状态化）、回滚（最近发布是否引入问题）。同时确认发布窗口：**先查最近一次发布/配置变更**——事故里"改了才挂"是最高频根因。

**第二步：用数据定界（是应用问题还是依赖问题）。** 打开监控面板按分层看：入口层看网关错误码、超时分布；应用层看 CPU、内存、GC 次数、事件循环延迟（`monitorEventLoopDelay`）、错误率与日志；依赖层看 DB 慢查询数、连接池水位（Q19）、Redis 延迟、三方接口 P99。**判断依据**：应用 CPU 爆满且事件循环延迟高 → 主线程问题（Q2）；CPU 正常但 DB 慢查询飙升 → 依赖问题（缺索引/大事务/锁等待）；连接池打满 → 连接泄漏或并发超限（Q19）。用链路追踪（如 OpenTelemetry）把单条慢请求的环节耗时拉出来，能直接看到慢在哪一段。

**第三步：定位根因。** 结合时间线（事故起点与发布/流量突变的时间对齐）、错误样本（堆栈、慢 SQL 日志）、复现验证（回滚或灰度验证根因假设）。典型根因：缓存击穿（Q12）、事件循环阻塞、慢 SQL、三方依赖超时拖垮线程池、发布引入回归。

**第四步：复盘与改进。** 产出时间线 + 根因 + 三个层面改进项：监控（补哪些指标、告警阈值）、容量（压测水位、限流预案）、流程（发布灰度、预案演练）。好的复盘不是追责，是让同类事故"第一次发生是事故，第二次是耻辱"。

加分表现：能主动说"我先同步业务方止损口径"（对外沟通）、"我同步测试与运维拉群"（协作）、"每一步结论我都有监控截图作证"（证据链）。答得出这些，说明真的处理过线上事故。

**延伸考点：** 如果瓶颈定位到"单条慢 SQL 拖垮了连接池"，你的止血顺序是什么？如何设计事故复盘的时间线文档，让改进项真正落地？
