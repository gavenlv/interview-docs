# 后端 · Python（面试题库）

本题库聚焦真实工程场景，不考八股文：核心考察点在 GIL 与并发模型选型、asyncio 异步实践与陷阱、内存管理与线上泄漏排查、性能剖析与优化、Web 框架与部署运维、类型注解与工程规范、Python 与大数据生态的结合。题目均为场景化提问，重点看候选人是否踩过坑、做过取舍、能给得出量化依据。

### Q1. 多线程爬虫加了线程并发，QPS 为什么还是上不去？

**问题：** 你用 Python 多线程写了一个并发爬虫，开了 50 个线程，但请求耗时几乎没有下降，QPS 也上不去。怎么排查和解释？

**期望加分项：**
- 能立刻点出「IO 密集 vs CPU 密集」的区别，说明 GIL 只在 CPU 密集时成为瓶颈，网络 IO 等待时会释放 GIL
- 能提出用 `ThreadPoolExecutor` 限流而非裸开线程，并给出量化对比（串行 10 QPS → 并发 200 QPS）
- 能分阶段埋点定位耗时分布（DNS、建连、发送、等待响应），而不是空谈理论
- 能指出 DNS 解析同步阻塞且持 GIL 的坑
- 能主动提连接复用（`requests.Session` 连接池）与线程数拐点压测

**减分项：**
- 只会背「Python 有 GIL，所以多线程没用」一句话
- 把 GIL 说成「Python 完全不能并发」，不知道 IO 场景多线程有效
- 答不出如何量化验证（压测曲线、cProfile）
- 忽略线程数盲目叠加带来的上下文切换开销

**解答：**
先判断瓶颈方向：请求没提速，先看耗时分布，而不是急着怪 GIL。用 `time.perf_counter` 分阶段埋点（DNS、建连、发送、等待、解析），或跑一次 `cProfile -s tottime` 看热点。Python 的 GIL 是解释器级锁，但 IO 等待（read/write、sleep）会释放 GIL，所以纯网络 IO 场景下多线程是有效的；QPS 上不去更常见的原因是：每个请求都新建 TCP 连接、DNS 查询是同步阻塞的（`socket.getaddrinfo` 不释放 GIL）、线程开太多导致切换开销反超收益。量化方法：写脚本对比 1 / 10 / 50 线程的 QPS 曲线，找到拐点。改进：全局复用 `requests.Session`（内部 urllib3 连接池）、缓存 DNS 结果、用 `ThreadPoolExecutor(max_workers=N)` 限制并发，IO 密集可进一步换 aiohttp 协程。实践中容易踩的坑：`requests` 的 DNS 解析发生在 GIL 内；Windows 上线程创建开销大；线程数按「目标 QPS × 单请求耗时」推算更合理，不是越大越好。

**延伸考点：** 同样的爬虫换成 CPU 密集任务（解密、压缩），多线程还有效吗？为什么？

### Q2. IO 密集服务，多进程、多线程、协程怎么选型？

**问题：** 要重写一个网关服务，上游是外部 HTTP 接口，单请求平均耗时 200ms，目标 QPS 5000。你在多进程、多线程、协程之间怎么选型？

**期望加分项：**
- 能先算账：按 Little's law，目标吞吐需要稳态并发 ≈ 5000 × 0.2s = 1000，据此判断规模
- 能说清三者调度开销与 GIL 的关系：协程 < 线程 < 进程
- 能结合 Python 生态给出方案（asyncio + Uvicorn，多 worker 横向扩容）
- 能主动考虑异常隔离、CPU 核数、容器限制
- 能给出 worker/线程数的配置经验及何时打破默认值

**减分项：**
- 直接背「协程最好」不给量化依据
- 分不清三者本质差异（谁来调度、切换成本量级）
- 忽略 GIL 在 CPU 密集场景的作用
- 答不出 IO 密集场景下线程池也能满足需求

**解答：**
先算账再选型。单请求 200ms、目标 5000 QPS，按 Little's law 稳态并发约 1000。协程是单线程内的用户态调度，切换成本微秒级，asyncio 单进程轻松扛住 1000+ 并发连接；线程切换是内核态、开销数十微秒，且有 GIL，但 IO 等待会释放 GIL，5000 QPS 用线程池也扛得住；进程开销最大，但能绕过 GIL 并做故障隔离。工程上的常见组合：asyncio 应用（FastAPI + Uvicorn）按核数开 worker（CPU 密集取核数，IO 密集可适当多开），每个 worker 内部是协程；若业务里有同步阻塞代码，用 `ThreadPoolExecutor` 隔离或直接多 worker 兜底。选型还要看团队：如果代码库全是同步 `requests`，硬上 asyncio 改造成本高，不如先同步 + 多 worker 再逐步迁移。坑点：asyncio 里混入同步阻塞调用会卡死整个事件循环，一个 worker 全挂；多 worker 是进程隔离，共享状态（连接池、内存缓存）会按倍数放大。

**延伸考点：** 协程切换真的无锁吗？asyncio 里两个协程真的能并行执行吗？

### Q3. 服务跑一周内存涨 3 倍，怎么定位泄漏？

**问题：** 一个常驻 FastAPI 服务，内存随运行时间持续增长，重启就好但无法接受。怎么系统化定位内存泄漏？

**期望加分项：**
- 能先区分「真泄漏」与「内存膨胀后到峰值稳定」（缓存未清理）
- 能给出具体工具链：`tracemalloc`、`objgraph`、`pympler`、`gc` 模块
- 能说出快照 diff 的定位手法（`snapshot.compare_to`，按 lineno）
- 能列常见元凶：无上限全局缓存、lru_cache 无 maxsize、闭包/回调持引用、pandas 循环拼接
- 能联系线上：定时 dump、RSS vs 私有内存、容器 OOM 现象

**减分项：**
- 只回答「用 objgraph」不给操作步骤
- 分不清内存泄漏和内存膨胀
- 不知道 tracemalloc 能按行定位分配位置
- 没有排查顺序：先怀疑业务对象还是第三方库

**解答：**
先区分「真泄漏」与「缓存膨胀」：把 RSS 随时间画曲线，若持续线性增长且 `gc.collect()` 后不回弹，才是真泄漏。步骤：① 启动时 `tracemalloc.start()`，用 `tracemalloc.get_traced_memory()` 采样；② 取两个时间点快照 diff：`tracemalloc.take_snapshot()` 后 `snapshot.compare_to(previous, 'lineno')`，能直接看到哪行代码持续增长；③ 对疑点对象用 `objgraph.show_growth()` 看哪些对象类型数量在涨，`show_backrefs()` 看谁持有引用——通常会发现全局 dict/list、类级缓存、无 maxsize 的 `lru_cache`、或某库内部保留历史。④ 配合 `gc.get_objects()` 按类型统计。常见元凶：无上限的全局缓存、FastAPI 依赖里把大对象挂在 module 级、循环里 `pd.concat` 拼接 DataFrame、日志 handler 泄漏、`requests.Session` 异常累积。线上可用 `py-spy dump` 或周期性 `tracemalloc` dump 到文件对比。坑：`sys.getsizeof` 只算浅层大小，别用它判断整体内存；容器里看 RSS 要减去页缓存；对象若被 C 扩展持有，`gc` 模块查不到，得靠 RSS 曲线或 ctypes 排查。

**延伸考点：** 如果泄漏对象在 C 扩展层（lxml、numpy），怎么继续定位？

### Q4. 循环引用会泄漏吗？del 之后内存为什么不回收？

**问题：** 两个对象互相引用，`del` 掉外部引用后内存没被释放。这是不是泄漏？Python 的 GC 到底怎么工作？

**期望加分项：**
- 能说清「引用计数为主、分代回收为辅」的两层机制
- 能解释循环引用不会被引用计数回收，需要分代 GC 发现
- 能说明 `__del__` 与循环引用组合的历史坑（Python 3.4+ 的改进与残留问题）
- 能给出验证方法：`gc.collect()` 前后对比，检查 `gc.garbage`
- 能联系真实案例：`gc.disable()` 的后果、回调闭包持引用

**减分项：**
- 把「引用计数」和「GC」混为一谈
- 答不出为什么需要分代回收（性能权衡）
- 不知道 `gc.disable()` 的后果
- 认为循环引用一定泄漏，答不出何时会被回收

**解答：**
Python 内存回收分两层：默认引用计数为主，引用数归零立即释放；循环引用（A↔B）引用计数永远不为零，靠第二层 `gc` 模块分代收集发现并回收。分代回收把对象分 0/1/2 三代，新对象进 0 代，每轮存活升代，代越老收集越不频繁——用少量「延迟回收」换吞吐。判断是不是泄漏，要看它是否还能被回收：手动 `gc.collect()` 后 RSS 明显回落，说明只是回收不及时而非泄漏；若对象进了 `gc.garbage`（含 `__del__` 的循环引用对象）才是真泄漏，Python 3.4 起大多数情况可回收，但 `__del__` + 循环引用仍是坑，应避免在 `__del__` 里做重活，改用 `weakref.finalize`。实践里更常见的是「假泄漏」：对象根本没被 `del` 干净，事件循环/回调闭包仍持引用。排查时先用 `gc.get_objects()` 看对象数量是否随请求增长，再 `objgraph.show_backrefs` 找持有链。坑：`gc.disable()` 能提升吞吐，但会让循环引用对象累积，高并发下反而内存暴涨；`gc.set_threshold` 别调太大，否则内存峰值失控。

**延伸考点：** 既然有分代回收，为什么还经常建议显式 `gc.collect()`？什么场景下该调？

### Q5. asyncio 并发爬虫 QPS 上不去，可能是什么原因？

**问题：** 用 asyncio + aiohttp 写了并发爬虫，开了 500 个并发任务，但 QPS 只有几十，远低于预期。你从哪些层面排查？

**期望加分项：**
- 能按「本地 → 网络 → 目标站」分层排查
- 能指出 aiohttp 默认连接池限制（`TCPConnector` 默认 limit=100）
- 能想到同步阻塞代码混入事件循环（`time.sleep`、requests、重计算）卡住整个循环
- 能提出用 `asyncio.run(debug=True)` / `slow_callback_duration` 抓超时告警
- 能联系 OS 限制：`ulimit -n` 文件描述符上限、DNS 线程池
- 能区分「本地瓶颈」和「目标站限速」（429/连接 reset）

**减分项：**
- 只会说「加并发数」，不知道并发受连接池/句柄限制
- 不知道 aiohttp 有默认连接限制
- 没意识到 `await asyncio.sleep` 与 `time.sleep` 的本质区别
- 答不出如何区分本地瓶颈与目标站瓶颈

**解答：**
按「本地 → 网络 → 目标站」三层排查。本地先看事件循环是否被阻塞：混入 `time.sleep`、`requests`、正则大循环、CPU 重计算都会卡住整个循环，表现为 QPS 骤降，用 `asyncio.run(debug=True)` 或设 `loop.slow_callback_duration` 能打出超时告警；检查 `TCPConnector`——aiohttp 默认每个 host 连接池上限 100，500 个任务实际在排队，需调大 `limit` 与 `limit_per_host`；再看文件描述符，500 并发至少 500+ 个 socket，`ulimit -n` 默认 1024 很紧张。网络层：DNS 解析走线程池（默认 32 线程），可换 `aiohttp.resolver.AsyncResolver`；TLS 握手本身有开销，全局复用一个 `ClientSession` 才能吃到 keep-alive 连接池收益。目标站：目标对单 IP 限速会表现为大量 429 / 连接 reset，此时本地优化无效，需要代理池或限速到对方容忍区间。坑：每请求新建 `ClientSession` 会丢连接池收益；`asyncio.gather` 传几千个任务前先确认任务数远低于连接上限；用 `py-spy dump` 看阻塞时线程栈更直观。

**延伸考点：** 目标站对单 IP 有 QPS 限制，而你要 10 倍量，怎么设计？怎么验证瓶颈在本地还是目标？

### Q6. FastAPI 视图写成 async def 就好吗？什么时候反而该写同步 def？

**问题：** 团队要求 FastAPI 接口全部写成 `async def`，但上线后某些接口反而更慢、甚至卡住整个服务。你怎么看？

**期望加分项：**
- 能说清 FastAPI 对普通 `def` 会自动丢线程池执行，`async def` 跑在主事件循环
- 能指出「async 视图里调用同步阻塞库（psycopg2、requests）」会卡死事件循环
- 能给出判断标准：你的 IO 是否有异步驱动（asyncpg / aiomysql / aiohttp）
- 能给出混用处理：`run_in_executor` / `anyio.to_thread` 隔离，或换异步驱动
- 能联系线上现象：一个慢接口拖垮所有接口，单请求压测看不出来

**减分项：**
- 一刀切认为 async 更好
- 不知道 FastAPI 对普通 def 的处理机制（自动线程池）
- 意识不到事件循环共享是所有 async 视图的公共瓶颈
- 答不出异步驱动缺失时的替代方案

**解答：**
关键不是「写不写 async」，而是「await 的东西是不是真的异步」。FastAPI 对普通 `def` 视图自动用 `ThreadPoolExecutor` 执行，不占事件循环；`async def` 视图直接跑在主事件循环里，若内部调用同步阻塞代码（`psycopg2`、`requests`、`time.sleep`），会阻塞整个进程的所有请求——表现为一个慢接口拖垮全站。判断标准：数据库驱动有没有 async 版（asyncpg / aiomysql / SQLAlchemy async）、HTTP 是否用 aiohttp/httpx；没有就老老实实写 `def`，或把阻塞调用用 `run_in_executor` / `anyio.to_thread` 包起来，更彻底的是换异步驱动。混用策略：阻塞调用少，用线程池隔离；调用密集，就整体迁异步。另一点：async 视图里做 CPU 密集计算同样卡循环，应丢进程池或任务队列。坑：`time.sleep` 在 async 里不报错，只是悄悄卡住整个服务，要用 `asyncio.sleep`；async DB 连接池打满时所有 `await` 排队等待；这类问题要并发压测（100 并发）才能暴露，单请求测试看不出来。

**延伸考点：** SQLAlchemy 的 async session 底层真的异步吗？连接池满时表现如何？

### Q7. 线上 CPU 100% 但 GC 正常，怎么定位热点？

**问题：** 线上 Python 服务 CPU 100% 持续多分钟，内存和 GC 都正常。如何在不重启、不改代码的情况下定位？

**期望加分项：**
- 能给出免插桩方案：`py-spy dump` / `py-spy record` 实时抓线程栈与火焰图
- 能说清 `cProfile`（侵入式插桩）与 `py-spy`（采样式）的差异与开销
- 能分析 CPU 高的可能原因：死循环、正则回溯、过度序列化、日志刷屏
- 能区分「单次调用耗时长」与「调用次数爆炸」两类热点
- 能结合业务周期（定时任务、缓存重建）找规律

**减分项：**
- 只会说「加日志重启」
- 不知道 py-spy 这类采样工具
- 定位到函数后没有进一步看调用次数 vs 单次耗时
- 没考虑多进程部署下先找到具体哪个 worker

**解答：**
在线定位别用 cProfile——侵入式插桩本身会让服务更慢。首选 `py-spy`：`py-spy dump --pid <pid>` 打印各线程当前栈，多抓几次看重复出现的帧；`py-spy record --pid <pid> -o flame.svg` 出火焰图，宽块就是热点。它基于采样，不改代码、不重启、开销极小，注意容器里需要 `SYS_PTRACE` 权限。拿到热点后分两类：单次调用耗时长的（算法、正则回溯如 `(a+)+` 嵌套量词、`json.dumps` 大对象）还是调用次数爆炸的（循环写日志、重复读文件、高频 `time.time()`）。常见元凶：正则回溯、INFO 级日志刷屏 + 字符串格式化、循环内字符串拼接、pandas 逐行 `apply`。再看是否周期性飙高：与定时任务、缓存重建时间点对齐分析。坑：`ps` 看到的是进程整体 CPU，多 worker 部署先 `top` 按 CPU 排序锁定具体 worker 再 `py-spy`；`GC 正常` 只能排除内存问题，别被带偏去查内存。

**延伸考点：** 火焰图里一个很宽的调用栈，怎么判断该优化「调用次数」还是「单次耗时」？

### Q8. Gunicorn worker 怎么配？sync 和异步 worker 怎么选？

**问题：** 用 Gunicorn 部署 Django/FastAPI，什么时候用 `sync` worker，什么时候用 `gevent` / `uvicorn` worker？并发参数怎么定？

**期望加分项：**
- 能说清 Gunicorn 的 pre-fork 模型：master 管理、worker 处理请求
- 能解释 sync worker「进程内一次一个请求」，适合短请求/CPU 密集，对长连接是灾难
- 能说明 gevent worker 适合同步代码 IO 密集；FastAPI 配 uvicorn worker
- 能给出 worker 数配置逻辑（核数起步、IO 密集上调）而非死记 `2*CPU+1`
- 能提到 `max-requests` 防内存膨胀、`graceful-timeout` 优雅退出、`timeout` 防假死

**减分项：**
- 只会背 `2*CPU+1` 公式不理解含义
- 不知道 sync worker 对长轮询/长连接的致命问题
- 意识不到「worker 数 × 每 worker 并发」才是总并发
- 答不出 uvicorn worker 与 gevent worker 的适用差异

**解答：**
Gunicorn 是 pre-fork 模型：master 进程管 worker，worker 干活。`sync` worker 每个 worker 一次只处理一个请求——最稳，也最容易被长请求拖垮：一个慢请求占住整个 worker，后面的快请求全部排队，所以 IO 密集或长连接场景必须上异步 worker。选型逻辑：代码是同步 IO 密集（Django + requests/psycopg2）→ `gevent` worker（monkey patch 让同步库变协作式，注意 monkey patch 必须最先 import）；代码本身是 asyncio（FastAPI）→ `uvicorn` worker（`--worker-class uvicorn.workers.UvicornWorker`），且不要在里面再叠加 gthread。worker 数：CPU 密集取核数，IO 密集可到核数 2-4 倍，核心公式是「worker 数 × 每 worker 并发能力 ≥ 目标并发」，压测定参。必配项：`--max-requests 10000 --max-requests-jitter 1000` 定期重启防内存膨胀；`--graceful-timeout 30` 配合 `SIGTERM` 优雅退出（先停接新请求、等存量跑完再杀）；`--timeout 30` 防 worker 卡死。坑：sync worker 配 Django 的 `CONN_MAX_AGE` 长连接会造成 worker 间连接数放大；UvicornWorker 下代码里若有同步阻塞照样卡——worker 类型救不了阻塞代码。

**延伸考点：** 优雅退出期间请求被 kill 导致数据不一致，怎么处理（信号时序、幂等重试）？

### Q9. requests 和 aiohttp 混用，连接超时与 TIME_WAIT 堆积怎么治理？

**问题：** 服务里既有 requests 又有 aiohttp 发 HTTP 请求，生产环境频繁出现连接超时和大量 TIME_WAIT 堆积。怎么治理？

**期望加分项：**
- 能说明混用本身不是问题，问题在连接复用与连接池管理
- 能给出 requests 全局复用 Session + urllib3 连接池参数（pool_connections / pool_maxsize、超时、重试）
- 能解释 aiohttp 连接池与事件循环绑定的特性（ClientSession 生命周期）
- 能分析 TIME_WAIT 堆积与端口耗尽的关系（四元组、ip_local_port_range、2MSL）
- 能给出监控指标：fd 数量、TIME_WAIT 计数、连接池命中率
- 能提出统一 http client 抽象层

**减分项：**
- 答不出每请求新建 Session / `requests.get` 的后果
- 不知道 urllib3 连接池默认大小与 keep-alive 行为
- 对 TIME_WAIT 只知概念不知危害
- 没考虑下游服务能承受的连接数上限

**解答：**
先分清问题归属：TIME_WAIT 堆积说明「短连接大量建立」，根因往往是每请求新建连接而非复用。requests 的规范用法是全局复用一个 `Session`（内部 urllib3 连接池，默认 `pool_connections=10`、`pool_maxsize=10`，高并发要调大）；不要每请求 `requests.get`——那是每次新建 TCP 连接，QPS 一高就端口耗尽（默认约 2.8 万可用端口、TIME_WAIT 在 2MSL≈60s 内不复用，客户端单机高并发必炸），排查时先 `netstat -ant | Select-String TIME_WAIT | Measure-Object`（Linux 用 `ss -s`）数一下数量。aiohttp 连接池绑定事件循环：`ClientSession` 与 loop 同生命周期，跨 loop 使用会告警；`TCPConnector(limit=100, limit_per_host=20, ttl_dns_cache=300)` 控制并发与 DNS 缓存。治理方案：两侧都加超时——requests `timeout=(connect, read)`、aiohttp `aiohttp.ClientTimeout(total=10)`，超时是必填参数；开 keep-alive，幂等请求配置重试；同一目标服务尽量统一走一个 client，避免两套连接互相放大。监控：`/proc/self/fd` 数量、TIME_WAIT 计数、连接池 hit/miss。坑：Docker 容器内 `ip_local_port_range` 是宿主机共享的；`httpx` 同步/异步 API 一致，可作为迁移中间层。

**延伸考点：** 客户端连接池设置多大合适？如何根据下游 QPS、平均耗时和超时时间推算？

### Q10. 处理 10GB 日志文件 OOM 了，怎么改？

**问题：** 用 Python 写了个脚本分析 10GB 日志，`open().read()` 一次性读入后 OOM。怎么重写？

**期望加分项：**
- 能点出逐行迭代与一次性读入的本质区别（文件对象迭代自带缓冲）
- 能给出流式 + 增量聚合的设计（generator 流水线、Counter/defaultdict 聚合）
- 能提到并行方案（multiprocessing 分片 seek 找行边界）与 mmap
- 能给出真实工具链：polars 惰性执行 / duckdb / pandas chunksize，或 awk/rg 粗筛
- 能注意编码、换行符（\r\n）、窗口对齐等边界问题

**减分项：**
- 只会说「用 readline」不给理由
- 不知道文件对象迭代底层有缓冲，内存占用与文件大小无关
- 把所有数据堆内存，不做流式聚合
- 没考虑排序/去重类需求的内存方案（heap Top N）

**解答：**
原则是「流式 + 增量聚合」。第一步把 `data = f.read()` 换成 `for line in f:`——文件对象迭代自带缓冲（默认按块读取），内存占用与文件大小无关，这是最直接的修复。但 10GB 且要统计多维度时，逐行 Python 可能慢，优化分三层：① 流式单机：逐行解析，聚合用 `defaultdict` / `Counter`，只要中间态不大即可；② 并行：`multiprocessing` 多 worker 各负责一段（`f.seek()` 到近似位置后找行边界），或用 `mmap` + 正则扫描避免逐行开销；③ 换引擎：纯文本聚合用 `awk` / `rg`，结构化大文件用 polars（惰性执行、列式、可外存）或 pandas `read_csv(chunksize=100_000)`。聚合类的坑：按时间窗聚合要做「窗口对齐」处理行边界；编码声明错误会让按行迭代错乱（用 `errors='replace'` 兜底）；Top N 用 `heapq` 而不是全量排序；Windows 下换行符 \r\n 要统一处理。10GB 的最终方案通常是：polars/awk 先粗筛，Python 只做业务聚合，避免逐行硬扛。

**延伸考点：** 换成 Kafka 实时流后，增量聚合的状态怎么管理？乱序数据窗口怎么处理？

### Q11. pandas 处理千万行数据太慢，怎么一步步优化？

**问题：** 一个 pandas 脚本处理 2000 万行数据要跑 1 小时，`apply` 里还有自定义逻辑。怎么一步步优化？

**期望加分项：**
- 能给出性能层次认知：向量化 > 内建聚合 > apply/iterrows（慢 50-500 倍）
- 能给出具体手段：dtype 优化（category / float32）、`np.select` / `pd.cut` / `Series.map` 替代 apply
- 能提到 `groupby.transform` 避免逐组循环、merge 前统一 key dtype
- 能给出量化验证方法（`%time` 分段、前后对比）
- 能考虑换引擎（polars / duckdb）与并行分片的前提
- 能意识到「关系操作是否该下沉到 SQL」的问题

**减分项：**
- 只会说「用多进程」不分析瓶颈在哪
- 不知道 apply 逐行开销的来源（解释器循环）
- 没检查过列 dtype 是否 object、是否该降精度
- 没考虑数据是否真的需要 pandas（该进数据库/duckdb）

**解答：**
先定位再动手：`df.info()` 看每列 dtype，object 列和 float64 是内存大户；用 `%time` 分段测各段耗时，通常 apply 最慢。优化顺序：① dtype：数值列降 float32/int32、类别列用 `category`，内存立省一半且 cache 友好；② 干掉 apply：逐行 apply 本质是 Python 解释器循环，比向量化慢 50-500 倍——if-elif 链用 `np.select`、范围分箱用 `pd.cut`、映射用 `Series.map` 或 `merge`、多列条件用布尔掩码组合；③ 聚合用 `groupby` 内建方法，`groupby.transform` 避免逐组拼接；④ 大 join 前统一 key dtype（object 与 category 混用会静默转 object 拖慢）；⑤ 数据量真到千万级，考虑换引擎：polars 惰性执行（向量化 + 并行 + 外存）、duckdb 直接 SQL 查询 Parquet，pandas 只做最终输出。坑：循环里 `pd.concat` 是 O(n²)，先收集 list 再一次性 concat；apply 里的第三方库调用先确认能否向量化（`str.extract` 替代 `re.search` 循环）；并行 map 在分片不均时收益有限，先把单进程优化到极致再谈并行。

**延伸考点：** apply 换成向量化后结果不一致（NaN、时区、边界），怎么做等价性验证？

### Q12. 代码库里 type hints 形同虚设，怎么落地？

**问题：** 接手一个 5 万行 Python 项目，函数签名与文档注释写满了「:param x:」，但没有类型标注。如何推进类型化改造且不破坏现有逻辑？

**期望加分项：**
- 能给出改造路线：新增代码强制、存量按调用链优先级渐进、先核心模块
- 能说明工具链：mypy strict 模式、pyright，及二者差异与分工
- 能处理动态特性（**kwargs 用 TypedDict、Any 蔓延、三方库 stub）
- 能区分静态检查（编译期契约）与运行时校验（pydantic 边界防御）的职责
- 能给出 CI 门槛与验收标准（diff 文件必须通过、禁止裸 `# type: ignore`）

**减分项：**
- 只喊口号「提高代码质量」没有执行方案
- 不知道 mypy 与 pyright 差异、`# type: ignore` 滥用
- 没考虑存量代码的渐进策略
- 答不出类型标注与运行时类型检查的关系

**解答：**
改造是工程问题不是技术问题。策略上：5 万行一次改完不现实，定红线——新增代码和改动函数必须有类型标注，CI 对 diff 文件强制 mypy；存量按调用链优先级（底层库 → 中间服务 → 接口层）分批推进，每批配回归。工具：mypy 生态成熟、`strict` 模式能抓 `Any` 蔓延；pyright 快、对 `TypedDict` / `Protocol` 支持好、IDE 体验佳——可 mypy 做 CI 门槛、pyright 做 IDE 实时检查。动态代码处理：`**kwargs` 用 `TypedDict` 约束；三方库无 stub 用 typeshed 或局部 `# type: ignore` 并注明原因（CI 里 `warn_unused_ignores` 抓冗余 ignore）；运行时校验只放在边界（请求、配置）用 pydantic，内部不做 runtime check 以免性能损耗——静态检查是「编译期契约」，pydantic 是「运行时防御」，分工不同。收益量化：类型化后重构时 mypy 能拦截字段名/类型变更漏改，接口契约清晰后联调错误率下降。坑：`Optional[str] = None` 默认值陷阱、泛型逆变协变误用、装饰器后签名丢失（`functools.wraps` + 显式泛型）、pandas 的 DataFrame 行级无类型导致「一切皆 Any」——用 `pandas-stubs` 或自定义 Protocol。

**延伸考点：** `Optional[str]` 和 `Union[str, None]` 有区别吗？`list[str]` 在 `isinstance` 里能用吗？为什么？

### Q13. 生产日志全打一个文件，排查问题无从下手？

**问题：** 线上服务日志混乱：有的模块不打日志、有的刷屏、关键请求链路没有 trace_id。请设计一套生产级日志方案。

**期望加分项：**
- 能给出 JSON 结构化日志规范（time / level / logger / trace_id / 业务字段）
- 能说明 logging 组件组合（Handler / Formatter / Filter）与 RotatingFileHandler 策略
- 能提出 trace_id 方案：中间件生成 + `contextvars.ContextVar` + logging Filter 注入
- 能说明同步日志 IO 对性能的影响与 QueueHandler 异步落盘
- 能考虑采集链路（stdout + 容器平台，或 Filebeat → Kafka → ES）与敏感字段脱敏
- 能给出热路径不打 debug、惰性格式化等纪律

**减分项：**
- 只会说「多打日志」不解决结构化与分级
- 不知道 logging 的 handler/formatter/filter 组合机制
- 没考虑多进程多 worker 写同一文件的轮转竞态
- 答不出 trace_id 在异步/多线程下怎么传递（contextvars）

**解答：**
设计要点：① 结构化：统一 JSON 格式，字段含 `time`、`level`、`logger`、`trace_id`、业务 key（user_id、order_id），这样 ES 才能聚合、告警才能按字段过滤；② trace_id：入口中间件生成或透传 `X-Request-Id`，存进 `contextvars.ContextVar`（asyncio 与 threading 都能自动继承，比 threading.local 安全），再写一个 logging `Filter` 把 trace_id 注入每条 record——同一请求的所有日志（含第三方库打的）都能串起来；③ handler：开发用 ConsoleHandler，生产用 `RotatingFileHandler` 按大小切（`maxBytes` / `backupCount`），高吞吐单进程用 `QueueHandler` + 后台线程落盘，避免日志 IO 拖慢业务；多 worker 写同一文件有切分竞态，给文件名带 pid 或走集中采集；④ 采集：现代做法是应用只打 stdout/stderr，容器平台（Docker json-file / K8s）统一收集，或 Filebeat 读文件入 Kafka/ES，应用层不碰文件权限；⑤ 纪律：热路径禁止 debug、循环内禁止 info，用 `logger.debug("...%s", obj)` 惰性格式化（不触发 % 运算）。坑：库代码里调 `logging.basicConfig` 会污染宿主应用；重复添加 handler 导致日志翻倍；`asctime` 默认不带毫秒，排查耗时问题要显式格式。

**延伸考点：** 异步队列日志怎么保证进程退出/崩溃时不丢日志？日志含手机号、token 等敏感字段怎么脱敏？

### Q14. 数据量上了 TB 级，Python 怎么和大数据生态协作？

**问题：** 现有 Python 批处理每天处理 TB 级数据，单机 pandas 已扛不住。你如何设计方案让 Python 与大数据生态协作？

**期望加分项：**
- 能给出分层：计算引擎下沉（Spark / Flink / DuckDB / polars），Python 做编排与精细分析
- 能说清 PySpark DataFrame API 优先，及 Python UDF 的序列化/逐行开销陷阱（pandas_udf 缓解）
- 能提到列式存储（Parquet）+ 分区裁剪 + zstd 压缩对扫描量的影响
- 能说明 Python 在生态中的角色：Airflow 编排、UDF、特征/模型、报表
- 能给出迁移时机判断：pandas 内存 < 数据量数倍或单任务小时级 → 先 duckdb/polars → 再 Spark

**减分项：**
- 认为「上 Spark 就行」却解释不了为什么
- 不知道 PySpark UDF 慢在 pickle/反序列化与逐行处理
- 忽略存储格式与分区设计（小文件问题）
- 没考虑集群上的依赖分发与调度（Airflow、--py-files）

**解答：**
核心判断：计算下沉、Python 做胶水与精细活。TB 级数据先用引擎算：离线批处理用 PySpark，DataFrame API 优先，能内置函数就绝不用 `udf`——PySpark UDF 每个分区逐行过 Python 解释器 + pickle 序列化，慢 10-100 倍，必须时用 `pandas_udf` 向量化 UDF 缓解；单机但内存吃紧时先试 DuckDB/polars——很多「TB 级」其实是亿行 × 几十列，SSD + 列式存储下 DuckDB 秒级出结果，根本不用上集群。存储层：统一 Parquet + 按日期/业务分区 + zstd 压缩，下游按分区裁剪，扫描量降几个数量级。Python 的位置：Airflow/Prefect 做 DAG 编排（调度、重试、依赖），Spark 预处理后 Python 做业务聚合、特征、模型、最终报表。迁移判断：pandas 内存 < 数据量 3 倍、或单任务小时级 → 先 DuckDB/polars 向量化；仍不够或需跨集群并发 → Spark。坑：Spark 里 Python 包要随任务分发（`--py-files` / conda env），否则集群上 ModuleNotFoundError；小文件问题（每个分区几 KB）比计算本身更拖垮 Spark；Python 3.11+ 与 Spark 版本兼容矩阵要先查；Pandas 2.0 的 Arrow 后端可直接读 Parquet，是轻量场景的好中间层。

**延伸考点：** PySpark 的 broadcast join 什么时候比 shuffle join 好？Python UDF 慢，除改写还有哪些工程手段（谓词下推、预计算列）？

### Q15. CPython 不够快，PyPy、Cython、numba、C 扩展怎么选？

**问题：** 某核心模块是 CPU 密集计算，CPython 跑 5 秒，业务要求 1 秒内。你评估 PyPy、Cython、C 扩展、numba 等方案时怎么权衡？

**期望加分项：**
- 能按场景给方案：数值计算用 numpy/numba，纯 Python 热循环用 Cython/PyPy
- 能说明 PyPy JIT 适合长运行热循环，但 C 扩展兼容性、内存模型（GC 不即时回收）是迁移风险
- 能给出 Cython 渐进路线（cythonize → typed → nogil 释放 GIL）与发布产物问题
- 能提到 numba `@njit` 的加速原理与限制（nopython 模式、首调编译开销）
- 能说先 profiler 确认热点占比再动手，量化收益预期
- 能考虑 multiprocessing + 共享内存、任务队列等「绕过 GIL」的思路

**减分项：**
- 不了解 PyPy 与 CPython 的生态兼容差异就推荐
- 不知道 Cython 释放 GIL 需要 `nogil` 与纯 C 数据
- 没有 profiler 直接「优化」
- 把「多进程」当万能答案，答不出进程间通信与序列化成本

**解答：**
先 profile 再选——很多「CPU 密集」其实是 numpy 运算、正则、字符串处理，方案完全不同。分三层：① 不换运行时：numpy 向量化（数组运算走 C 底层）、`functools.lru_cache` 缓存结果、`multiprocessing` 拆任务到多核（进程各自持 GIL，可线性加速到核数，注意 IPC 序列化成本）；② 数值计算：`numba @njit` 把 Python 循环编译成 LLVM 机器码，对 numpy 友好，但要求代码能在 nopython 模式编译，不能随意用 Python 对象；③ 通用热循环：Cython 渐进式——先 `cythonize` 编译原样 Python 已能提速，再给热点加 typed 变量、`cdef`，最后对纯 C 数据段 `with nogil:` 释放 GIL 实现真并行；④ PyPy：JIT 对解释执行的热循环收益大（可达 5-10 倍），但依赖 C API 的扩展（numpy/pandas/scipy）在 PyPy 上要走 CFFI 或兼容层，部署和生态是大问题，适合「纯 Python 逻辑、无 C 扩展依赖、长驻进程」的场景。工程决策：先花 2 小时 profile，热点占比 > 60% 才值得动；如果热点在第三方 C 库内部，换库比换运行时便宜。坑：numba 首调有 1-2 秒 JIT 编译开销；PyPy 对象由 GC 管理、内存曲线与 CPython 不同，要重新压测；Cython 发布要带 `.so` 产物，跨平台构建复杂。

**延伸考点：** numba 编译失败回退到对象模式时为什么更慢？释放 GIL 的 Cython 代码如何保证线程安全？

### Q16. pip install 装出来的环境跑几天就坏，怎么治？

**问题：** 项目用 `pip install -r requirements.txt` 部署，但 requirements 里都是无版本号的包，生产环境经常装出新版本直接崩。怎么建立可靠的依赖管理？

**期望加分项：**
- 能说清「直接依赖」与「传递依赖」的区分，以及 requirements.txt 不做完整依赖树锁定的缺点
- 能给出工具链：poetry / pip-tools（pip-compile）/ uv 生成 lock 文件
- 能说明 lock 文件的作用：固定完整依赖树保证可复现
- 能提到 Python 版本锁定（.python-version / requires-python）与虚拟环境/镜像隔离
- 能给出升级流程：改约束 → 重新编译 lock → CI 测试 → 灰度
- 能考虑私有源与哈希校验（--hash / wheels 校验）

**减分项：**
- 只回答「加版本号」，不知道传递依赖也会变
- 分不清直接依赖与完整依赖树
- 没有升级与审计流程（pip-audit）
- 不知道 lock 文件与 requirements.in 的关系

**解答：**
问题本质：requirements.txt 只约束了「直接依赖的顶层版本」，`pip install` 每次都重新解析传递依赖的最新版——昨天 `a` 依赖 `b==1.0`，今天 `b` 升 2.0 破坏接口，明天就崩；而且生产崩、本地不崩，因为本地缓存了旧版。方案：① 引入 lock 文件锁定完整依赖树：poetry 用 `poetry.lock`（同时管虚拟环境与构建），pip-tools 用 `requirements.in`（只写直接依赖）+ `pip-compile` 生成完整锁定的 `requirements.txt`（可加 `--generate-hashes`），uv 是更快的新选择（`uv lock` 生成 `uv.lock`，兼容 pip 生态）；② 变更流程：改依赖 → 重新生成 lock（升级才跑 `--upgrade`）→ CI 按 lock 全量安装测试 → 生产按 lock 安装；禁止生产手装散包；③ 环境隔离：每项目虚拟环境或容器镜像，镜像构建时按 lock 安装并打进镜像；④ Python 版本也锁：`.python-version` / `requires-python`，3.11 → 3.12 的 C 扩展二进制不兼容是隐性坑；⑤ 定期 `pip-audit` 扫 CVE，升级走完整测试而非裸升。坑：Windows 与 Linux 的 wheel 二进制不一致，lock 应在生产同环境（Linux 容器）生成；poetry 与 pip 混用会让 lock 失效，二选一；私有 index 配 `PIP_INDEX_URL`，lock 加哈希校验防中间人。

**延伸考点：** 公司内部私有包怎么进入 lock 管理？依赖升级如何验证兼容性（契约测试、金丝雀）？

### Q17. asyncio 里滥用 `asyncio.to_thread`，服务假死问题出在哪？

**问题：** 有人在 asyncio 服务里用 `await asyncio.to_thread(sync_db_query)` 处理所有数据库操作，并发一高，数据库连接数和线程数双双飙升，服务假死。问题出在哪？

**期望加分项：**
- 能指出 to_thread 底层是 `run_in_executor` + 默认 ThreadPoolExecutor（min(32, cpu+4)），并发不受限
- 能解释「线程池满 → 任务排队 → 请求等待」造成假死的现象
- 能说明数据库连接数被放大、撞上 `max_connections` 的连锁反应
- 能给出正确姿势：真异步驱动（asyncpg/aiomysql）或 Semaphore 限流 + 超时
- 能说明 `asyncio.cancel` 无法真正取消线程内任务、`wait_for` 超时后线程仍执行的坑

**减分项：**
- 只说「应该用异步驱动」而答不出为什么
- 不知道 to_thread 用默认线程池且不限制并发
- 意识不到连接数放大与下游 max_connections 的关系
- 没提超时与信号量控制

**解答：**
`asyncio.to_thread` 本质是 `loop.run_in_executor(None, func)`，用默认 `ThreadPoolExecutor`（`min(32, os.cpu_count()+4)` 线程）。问题一：并发不受限——请求量超过线程数后任务在队列排队，表现为线程池满、CPU 不高但请求全卡；问题二：数据库连接数被放大——每个线程持一个连接，`max_connections=100` 的库很快被打满，新连接报 `too many connections`，连锁假死。正确做法分三层：① 能真异步就用真异步：asyncpg / aiomysql / SQLAlchemy async，连接池 `pool_size` 设小（并发与连接是 1:N 复用，几十并发几个连接就够）；② 必须用 to_thread 的场景（同步 SDK 无法换）：用 `asyncio.Semaphore(n)` 包住调用限流，配合 `asyncio.wait_for` 加超时，自定义线程池大小并做满时告警；③ 治理：监控线程池活跃度与数据库连接数，打点告警而非等假死再救。更深一层：GIL 下线程内 CPU 段仍互斥，纯 IO 等待会释放 GIL，线程池主要开销在切换。坑：`run_in_executor` 的任务无法被 `asyncio.cancel` 真正中断——取消只丢引用，线程照跑；`wait_for` 超时后底层线程还在执行，可能造成请求重复执行/数据重复写入，需要幂等兜底。

**延伸考点：** `await` 一个永远不完成的协程（第三方 SDK 内部卡死）怎么兜底？取消任务和杀线程的能力边界是什么？

### Q18. 接口返回 50MB JSON 又慢又频繁 GC，怎么优化？

**问题：** 接口返回 50MB JSON（列表嵌套），压测发现 CPU 高、GC 次数多、响应慢。序列化是主要瓶颈，怎么优化？

**期望加分项：**
- 能先质疑需求：50MB 单响应本身就该分页或改协议（架构问题优先）
- 能给出 orjson / ujson / rapidjson 与标准库的量化对比（3-10 倍差距）
- 能说明 GC 频繁源于海量临时对象分配，换 C 层序列化库显著缓解
- 能提到 gzip/brotli 传输压缩与 StreamingResponse 流式响应
- 能提到字段裁剪、结果缓存（缓存 gzip 后的 bytes，连压缩 CPU 都省）
- 能注意 orjson 与标准库在浮点、datetime、bytes 上的行为差异

**减分项：**
- 只说「加 gzip」不分析序列化本身
- 不知道 orjson 与 ujson 的差异（orjson 支持 dataclass/numpy、输出 bytes）
- 没考虑数据结构设计（深嵌套 vs 扁平化）
- 忽略了 50MB 本身就是设计问题

**解答：**
先质疑需求：50MB 单响应本身就该分页或改协议，这是架构问题；确实要传再逐层优化。第一层换库：`json.dumps` 是纯 Python 逐对象实现，换 `orjson.dumps`（Rust 实现）典型提速 3-10 倍，自动处理 dataclass/numpy/datetime，输出 bytes 可直接写响应；`ujson` 快但对部分类型兼容性差，`rapidjson` 是 C++ 实现。对比方法：`timeit` 同一份数据跑三个库，量化后上线。第二层减对象：GC 频繁源于序列化过程海量临时 dict/list/str 分配，换 C 层库后显著缓解；注意 `orjson` 选项（`OPT_NON_STR_KEYS` 等）与标准库行为差异。第三层传输：加 `Content-Encoding: gzip`（或 brotli/zstd），50MB 文本可压到几 MB，带宽省 90%；FastAPI 用 `StreamingResponse` 分块流式，避免整个响应驻留内存；结果可缓存时在 Redis 存 gzip 后的 bytes，连压缩 CPU 都省。第四层业务：字段白名单裁剪、深嵌套结构拍平、查询层提前聚合。坑：`json.dumps(..., default=str)` 会把 datetime 转字符串且每对象查一次 default 函数，比 orjson 慢一个数量级；orjson 浮点序列化与标准库有差异，测试要覆盖；gzip 在 CPU 密集时是另一笔开销，要权衡带宽 vs CPU。

**延伸考点：** 如果 50MB 是「列表套对象套列表」的多层嵌套，数据结构怎么设计能让客户端免解析直接消费？（列式 / 扁平 / 增量协议）

### Q19. 秒杀场景用 Python + Redis 扣库存，怎么保证不超卖？

**问题：** 用 Python + Redis 实现秒杀扣库存接口，`get` 判断库存 → `decr` 扣减，压测 1000 并发后超卖严重。怎么修复？还有哪些隐患？

**期望加分项：**
- 能指出 get→decr 是「检查再修改」竞态，必须用原子操作（Lua / DECR 先扣再验）
- 能给出 Redis Lua 脚本原子扣减方案（`register_script`、KEYS/ARGV 用法）
- 能说清 Redis 单线程命令原子性，及 MULTI/EXEC 与 Lua 的差异
- 能考虑扣减与订单落库的一致性：异步补偿、幂等键、对账
- 能考虑热点 key（分片库存）、超时重试重复扣、集群下 Lua 的 slot 限制
- 能给出降级方案（本地预扣 + 异步同步、限流）

**减分项：**
- 只会说「加锁」，说不出分布式锁的坑（锁过期、重入、主从切换）
- 不知道 Redis 单命令 / Lua 原子性的原理
- 没考虑「扣减成功但下单失败」的补偿
- 答不出为什么 get-decr 在 1000 并发下必超卖

**解答：**
先讲根因：`stock = get(); if stock > 0: decr()` 两个命令之间有竞态窗口，1000 并发下 N 个请求同时读到 stock=1，全部执行 decr → 超卖。修复用 Redis 单线程命令的原子性：最简方案先扣再验 `if r.decr(key) < 0: 回滚`；更稳的是 Lua 脚本一次 `eval` 完成「检查 + 扣减」：`local s = tonumber(redis.call('GET', KEYS[1])); if s >= tonumber(ARGV[1]) then return redis.call('DECRBY', KEYS[1], ARGV[1]) else return -1 end`，Lua 在 Redis 内原子执行，`redis-py` 用 `register_script` 注册复用。注意 MULTI/EXEC 只是「打包执行」不保证读改写逻辑正确，WATCH/MULTI 乐观锁在高竞争下重试率高，Lua 最实用。工程上还差几层：① 一致性：先扣减后异步创建订单，失败要补偿回滚（消息队列 + 定时对账）；② 幂等：user_id+sku 幂等键去重防重复点击；③ 热点：单 key 每秒几十万 QPS 会成热点，做分片库存（stock:1..N 子片独立扣，总数兜底）；④ 超时重试：`DECR` 超时重试可能重复扣，重试带请求幂等；⑤ 降级：本地进程内预扣 + 异步同步 Redis，接受极端情况误差。坑：`redis-py` 注意连接池 max_connections；Lua 里参数用 `ARGV` 传值而非拼进 KEYS；Redis Cluster 下 Lua 访问的 key 必须在同一 slot，用 hash tag 兜底。

**延伸考点：** Redis 挂了扣库存服务怎么降级？「先扣 Redis 再写 DB，DB 失败回滚 Redis」如何保证最终一致？

### Q20. 接手一个「能跑但没人敢动」的 Python 微服务，三个月怎么治理？

**问题：** 你加入新团队，要把一个「能跑但没人敢动」的 Python 微服务改造成可维护、可扩展、可观测的系统。给你三个月，你的治理路线是什么？

**期望加分项：**
- 能给出分阶段路线：先可观测（日志/指标/trace）→ 再稳定性（超时/重试/熔断）→ 后工程化（类型/测试/CI）
- 能给出工具选型与理由：结构化日志、Prometheus 指标、OpenTelemetry、Sentry
- 能坚持「治理不是推倒重来」：防腐层、增量改造，重写只针对高频改动模块
- 能提到测试金字塔与契约测试、OpenAPI 文档
- 能给出可度量标准：P99、错误率、MTTR、部署失败率，并说明如何落地
- 能平衡技术债与业务交付节奏，定义完成的验收标准

**减分项：**
- 上来就说「重构重写」不问成本
- 只列技术名词没有优先级与时间表
- 忽视可观测性应最先投入（否则后续优化无数据）
- 没有团队协作层面的推进（规范文档、评审、onboarding）

**解答：**
三个月按「止血 → 建标准 → 提质量」推进。第一个月可观测性优先：统一结构化日志（trace_id 串链路）、Prometheus 埋点（QPS、P99、错误率、慢查询）、Sentry 收异常、OpenTelemetry 做链路追踪——没有数据，后续优化都是盲人摸象；同时把无超时、无重试的调用点列清单，先加超时与熔断止血。第二个月工程基线：代码规范（Black + Ruff + mypy strict，CI 强制）、错误处理规范（自定义异常层次 + 统一响应格式）、测试金字塔（核心单测 + 关键链路集成测试）、存量代码用防腐层隔离脏逻辑，不做大爆炸重写；补 OpenAPI 文档与 pydantic-settings 配置化。第三个月稳定性与演进：部署改造（优雅退出、健康检查、滚动发布）、压测定容量基线、灰度发布；对高频改动模块做类型化与重构；最后沉淀团队规范与 onboarding 材料。成效度量：部署失败率、P99、MTTR、线上事故数。坑：治理最容易死在「标准先行、落地滞后」——每条规范必须配 CI 检查和一次实际落地；可观测性不能拖到第三个月；重写只针对改动最频繁的模块。

**延伸考点：** 如果业务方要求下季度上线 3 个新功能，你的治理计划怎么让路？技术债怎么和业务方谈优先级？
