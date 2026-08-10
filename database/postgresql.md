# 数据库 · PostgreSQL（面试题库）

本文件聚焦 PostgreSQL 在真实后端/数据工程中的落地能力，覆盖 MVCC 与垃圾回收（vacuum、bloat、autovacuum）、事务隔离与快照、索引选型（B-tree/GIN/GiST/BRIN）、慢查询优化、锁与并发排障、性能调优（内存/checkpoint/WAL）、分区、JSONB、全文检索、复制（物理/逻辑）、高可用与备份恢复、连接池、字符集与排序，以及迁移与监控体系。题目均为场景化提问，重点考察候选人基于量化证据定位问题、在多种方案间做取舍并给出可落地上线的实践结论的能力，不考概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

---

### Q1. 表删了大量数据，文件大小却没变小，查询还变慢了，怎么处理？

**问题：** 订单表有 5 亿行，业务上 delete 了很多数据，但表文件大小没变小，查询还变慢了，怎么定位和处理？

**期望加分项：**
- 能说出 MVCC 下 DELETE 不物理删除，而是产生 dead tuple，vacuum 只回收空间供复用、不把文件还给操作系统（高水位）
- 能给出量化定位手段：pg_stat_user_tables 的 n_dead_tup/last_vacuum、pgstattuple 的膨胀率、relpages 对比
- 能区分「统计信息过期导致变慢」和「真实 bloat 导致变慢」两种可能
- 能说清 VACUUM 与 VACUUM FULL/pg_repack 的区别：前者在线但缩不了文件，后者缩文件但要独占锁或做在线重写
- 能联系线上实践：先查长事务/hot_standby_feedback 等导致 vacuum 被阻塞的原因，而不是盲目 FULL
- 能主动考虑边界：大表 VACUUM FULL 会占磁盘双倍空间、锁表，要规划停机窗口

**减分项：**
- 只会背「dead tuple」概念，给不出任何查询和处置命令
- 一上来就建议 VACUUM FULL，不考虑锁和磁盘空间代价
- 把变慢单纯归因于 bloat，忽略统计信息过期、索引膨胀等更常见原因
- 不知道 vacuum 被长事务/hot_standby_feedback 阻塞这回事

**解答：**
先确认是不是真空回收出了问题，再决定处置强度。第一步查证据：`SELECT n_dead_tup, n_live_tup, last_vacuum, last_autovacuum FROM pg_stat_user_tables WHERE relname='orders';` 若 n_dead_tup 长期很大、last_autovacuum 很旧，说明垃圾回收没跟上；再装 pgstattuple 扩展用 `pgstattuple('orders')` 看 dead_tuple_percent 估算膨胀率。第二步找阻塞源：长事务（`SELECT * FROM pg_stat_activity WHERE xact_start 很久之前`）或备库开启 hot_standby_feedback 都会阻止 vacuum 推进，先处理这些根因。第三步按膨胀程度处置：轻度膨胀靠手工 `VACUUM (VERBOSE, ANALYZE) orders;` 在线回收；中度膨胀用 `VACUUM FULL` 或 pg_repack 重写表缩文件；同时 `REINDEX TABLE` 处理索引膨胀。实践中的坑：VACUUM FULL 需要 ACCESS EXCLUSIVE 锁并占用约一倍磁盘空间，线上要在低峰做；查询变慢常常先查 `ANALYZE` 后的执行计划——统计信息过旧导致选错执行计划比 bloat 更常见，先 `ANALYZE` 复测；日常靠 autovacuum 维持，别指望定期手工 FULL。

**延伸考点：** hot_standby_feedback 为什么会导致主库 bloat？怎么用 pg_stat_progress_vacuum 观察在线 vacuum 进度？

---

### Q2. 高并发写入频繁报 serialization failure 和 deadlock，怎么排查？

**问题：** 高并发插入/更新场景下，应用频繁报 "could not serialize access due to concurrent update" 或 "deadlock detected"，怎么排查和解决？

**期望加分项：**
- 能先区分两类错误：40001（serialization failure，快照冲突）与 40P01（deadlock，锁环），处理方式完全不同
- 能说出 repeatable read/serializable 下并发更新同一行会直接抛 40001，而 read committed 下是阻塞后覆盖
- 能给出排查手段：打开 log_lock_waits + deadlock_timeout、看服务端日志的 DETAIL（PID/语句/锁）
- 能给出工程解法：统一更新顺序、缩短事务、应用层重试（退避）、必要时降隔离级别
- 能提到高并发 upsert（ON CONFLICT）并发命中同一行在低版本会报 serialization failure 的已知问题
- 能联系线上实践：ORM 隐式事务导致锁持有时间长这类真实根因

**减分项：**
- 分不清 40001 和 40P01 的区别，混为一谈
- 只会说「加锁/重试」，给不出死锁日志怎么读、pg_locks 怎么查
- 不知道重复读隔离级别下写冲突是「乐观失败」而非「阻塞」
- 不讨论事务长度和锁获取顺序这些根因

**解答：**
先把两类错误分开。40001 常见于两个事务以 repeatable read/serializable 并发更新同一行：后者提交时发现快照之后该行被改过，直接抛错放弃（乐观策略），解法是应用捕获 40001 重试（指数退避），或业务上显式 `SELECT ... FOR UPDATE` 排队，或接受 read committed 语义。高并发 UPSERT 场景：多个连接对同一行执行 INSERT ... ON CONFLICT DO UPDATE，旧版本（PG 11 前）并发同一行也会产生 serialization failure，升级版本或做重试。40P01 死锁则是两个事务以不同顺序拿锁形成锁环：先开 `log_lock_waits = on`、`deadlock_timeout` 调到 1s 左右，服务端日志会打印两个事务的 PID、语句和锁类型，再结合 `pg_locks` + `pg_stat_activity` 还原锁链。根因通常是：更新顺序不一致（先改 order 再改 user，与别人相反）、事务内做过多无关操作（外部调用、大查询）拉长持锁时间、外键更新触发子表锁。预防：按主键排序后统一更新顺序、事务短小、避免事务内网络调用。实践中的坑：别只杀进程解死锁，要根治锁顺序；deadlock_timeout 别设到几百毫秒，会导致频繁死锁检测消耗 CPU。

**延伸考点：** 同一行并发 upsert 为什么会报 40001，而普通 UPDATE 在 read committed 下不报？应用层重试要注意哪些幂等性问题？

---

### Q3. EXPLAIN ANALYZE 显示行数估算严重偏低，全表扫描，怎么优化？

**问题：** 一条 SQL 之前毫秒级现在秒级，EXPLAIN ANALYZE 显示对 500 万行的表估了 1000 行，走了 seq scan，怎么定位和优化？

**期望加分项：**
- 能先抓「估算 vs 实际行数」的偏差，判断是统计信息过期还是成本模型参数问题
- 能给出完整的观测姿势：EXPLAIN (ANALYZE, BUFFERS)，区分 shared buffers 命中与磁盘 IO
- 能说出对应解法：ANALYZE / 提高 default_statistics_target / 扩展统计信息（extended statistics）
- 能讲出成本模型参数的影响：effective_cache_size、random_page_cost（SSD 上约 1.1）导致 seq scan 误判
- 能联系线上实践：统计信息过旧导致执行计划劣化的监控手段（pg_stat_all_tables 的 last_analyze）
- 能主动考虑边界：ANALYZE 后计划漂移、生产库避免随意改参数

**减分项：**
- 不看 actual rows，只看「走了 seq scan」就急着加索引
- 说不出 BUFFERS/计时信息怎么读
- 把问题一刀切归因于「没索引」，答不出成本模型和统计信息的角色
- 不知道 ANALYZE 频率与自动统计收集（autovacuum 触发 analyze）的关系

**解答：**
先看执行计划里 estimated rows 和 actual rows 的偏差：偏差几个数量级是统计信息过期，`ANALYZE orders;` 复测即可。若偏差仍大，是统计采样精度问题：提高 `default_statistics_target`（默认 100，可到 1000-5000）或对关联多列建扩展统计信息 `CREATE STATISTICS ... ON (a, b) FROM t;`（PG 10+）解决多列相关导致的选择性误估。若估算合理但仍选 seq scan，是成本模型失真：`effective_cache_size` 应设为可用内存的 75% 左右，SSD 上 `random_page_cost` 从默认 4 调到 1.1，让索引随机读不再被高估。观测姿势用 `EXPLAIN (ANALYZE, BUFFERS)`：对比每个节点耗时占比和实际读盘量，命中 shared buffers 的节点快，读磁盘的节点是瓶颈。实践中的坑：生产上别直接调 random_page_cost 压测对比前后计划；ANALYZE 后执行计划可能变化导致 SQL 性能波动，可用 pg_hint_plan 定向但慎用；排查时用 `EXPLAIN` 之前先确认参数 `track_counts` 已开启，否则统计信息根本不更新。

**延伸考点：** 多列条件相关性强导致估算失真的具体例子是什么？为什么 autovacuum 触发的 analyze 在大表上采样可能不够？

---

### Q4. 等值、范围、数组包含、模糊搜索、低基数列过滤，索引怎么选型？

**问题：** 同一张表有等值查询、范围查询、数组 contains 查询、模糊搜索，还有低基数列（状态字段）过滤，你分别选什么索引？B-tree 之外的 GIN/GiST/BRIN 各适合什么场景？

**期望加分项：**
- 能按查询谓词对号入座：等值/范围 → B-tree，数组/JSONB/全文 → GIN，几何/相似 → GiST，物理有序大表的范围扫描 → BRIN
- 能说清部分索引与表达式索引的适用场景：低基数列用部分索引、大小写不敏感用表达式索引
- 能指出低基数列（如状态只占 5%）直接建 B-tree 通常不如部分索引或不建
- 能讲清多列 B-tree 的前导列规则：等值列放前、选择性高的放前，且不能任意跳过前导列
- 能权衡写放大：GIN 写慢适合读多写少，BRIN 要求数据物理有序
- 能联系线上实践：用 pg_stat_user_indexes 的 idx_scan 找出冗余索引删除

**减分项：**
- 一律回答「加索引」，说不出具体类型和依据
- 不知道部分索引和表达式索引的存在
- 把 GIN/GiST/BRIN 的场景说反或混淆
- 不考虑索引维护成本（写放大、膨胀）和删除冗余索引

**解答：**
按谓词形态选型而非一律 B-tree。等值与范围：B-tree，多列时把等值条件列放前面（前导列规则），选择性高的放前；注意查询不能跳过前导列。数组/JSONB contains/全文检索：GIN，对应 `@>`、`?`、to_tsvector 等操作符，GIN 反范式存储适合读多写少，更新频繁场景写放大明显。几何、范围类型、向量相似度：GiST，支持 KNN 排序（`ORDER BY col <-> point`）。超大数据表、数据按插入时间物理有序、且查询按时间范围过滤：BRIN 是隐藏神器——每 128 个 block 存一行元数据，10 亿行表 BRIN 索引只有几十 MB，而 B-tree 可能要几十 GB。低基数列过滤：`status='pending'` 只命中 1% 行时，建部分索引 `CREATE INDEX ... ON t (status) WHERE status='pending';`，体积小命中率高；选择性太低的列干脆不建。表达式索引：`CREATE INDEX ON t (lower(email));` 支持大小写不敏感等值。实践中的坑：多列索引的列顺序错了等于没建；部分索引谓词与查询条件必须匹配才走；GIN 在大量 UPDATE 场景膨胀快，要配合 vacuum；线上用 `pg_stat_user_indexes` 找 idx_scan 长期为 0 的索引删除，索引不是越多越好。

**延伸考点：** BRIN 索引在什么条件下会退化失效（min/max 范围重叠）？多列 B-tree 的列顺序为什么要用等值优先于范围？

---

### Q5. 两个事务都读同一行，后提交者 UPDATE 报 serialization failure，为什么？

**问题：** 事务 A、B 都 select 了同一行，A 先 update 并提交，B 再 update 时报 "could not serialize access due to concurrent update"（40001），为什么？业务端该怎么办？

**期望加分项：**
- 能讲清 PG 快照隔离的语义：repeatable read 下读是快照，但写-写冲突由「首次更新者获胜」裁决
- 能指出这是乐观失败而非阻塞：B 发现目标行版本比自己快照新且不可见，直接抛错让应用重试
- 能对比 MySQL repeatable read 的行为差异：MySQL 用当前读 + 锁，B 会阻塞而不是失败
- 能给出业务解法：捕获 40001 重试、显式 SELECT ... FOR UPDATE 排队、或降级到 read committed
- 能延伸讲 write skew 与 serializable/SSI 的关系
- 能联系线上实践：库存扣减这类场景怎么用原子 UPDATE 规避

**减分项：**
- 把 40001 归因于死锁或锁等待
- 说不清「快照读 + 首次更新者获胜」的机制，只会背定义
- 建议改成「加悲观锁」但说不清 select for update 的具体用法和代价
- 不知道 read committed 下同样并发更新的行为差异

**解答：**
PostgreSQL 的 repeatable read 是快照隔离：事务内所有读基于事务开始时的快照，但更新走的是「当前读 + 首次更新者获胜」：B 更新某行时发现该行已被并发事务修改（且新版本 B 的快照不可见），直接抛 40001 放弃 B 的整个事务，这是乐观冲突失败，需要应用重试。注意这不是死锁、不是锁等待，B 不会阻塞。与 MySQL 对比：MySQL RR 的 UPDATE 是当前读并加 next-key 锁，B 会阻塞到 A 提交再基于新值执行（可能覆盖 A 的结果）；PG 选择「失败并让应用重试」，语义更干净但要求应用处理 40001。业务侧三种做法：①捕获 40001 指数退避重试（适合冲突概率低的场景）；②预期冲突高的场景（如库存扣减）显式 `SELECT ... FOR UPDATE` 悲观排队，或写成原子语句 `UPDATE stock SET qty = qty - 1 WHERE id = ? AND qty >= 1` 并检查受影响行数；③冲突可接受的业务降级到 read committed（并发更新同一行是阻塞后覆盖，不再抛错，但要注意丢失更新语义）。延伸：serializable 隔离级别下 SSI 还能检测 write skew（两人各读一行再各自改不重叠的行破坏约束），代价是更多 40001。实践中的坑：ORM 默认隐式开事务时，40001 会让整个工作单元回滚，重试要做在完整业务单元层面而非单条 SQL。

**延伸考点：** 什么是 write skew？为什么 read committed 下两个事务并发 UPDATE 同一行是「最后提交者胜」，业务上如何避免丢失更新？

---

### Q6. 线上偶发 deadlock detected，怎么快速定位根因和预防？

**问题：** 业务日志偶发出现 "deadlock detected"，服务端日志能看到两个进程互相等待，怎么快速定位根因并预防再犯？

**期望加分项：**
- 能给出定位配置：log_lock_waits=on、deadlock_timeout 调低，读服务端日志的 DETAIL（PID/语句/锁链）
- 能给出实时的锁链还原手段：pg_locks 关联 pg_stat_activity、pg_blocking_pids()
- 能归纳典型根因：多表更新顺序不一致、事务过长、外键约束引发的隐式锁
- 能给出预防方案：统一锁获取顺序（按主键排序）、缩短事务、避免事务内外部调用
- 能区分死锁和普通锁等待：死锁是「互相等待成环」，会有一方被自动牺牲回滚
- 能联系线上实践：批量任务并发处理、订单和用户表更新顺序这类真实案例

**减分项：**
- 只会说「杀掉一个事务」，答不出锁环的成因和预防
- 不知道 log_lock_waits / deadlock_timeout 这些开关
- 把死锁归因于慢查询或行数多，抓不到「锁顺序」这个本质
- 不考虑外键、索引页锁这些隐式锁来源

**解答：**
死锁本质是锁环：T1 持有 A 等 B，T2 持有 B 等 A，PG 检测到后随机牺牲一方回滚并记录日志。定位分两步：事前配置 `log_lock_waits = on`（deadlock_timeout 内没拿到锁就记日志），deadlock_timeout 默认 1s 即可，调太低会频繁做检测拖慢性能。事后从日志 DETAIL 段拿两个事务的 PID、正在执行的语句、各自持有的锁类型（RowExclusiveLock、ShareLock 等）。实时查锁链：`SELECT pid, wait_event_type, wait_event, query FROM pg_stat_activity WHERE wait_event_type='Lock';` 再用 `SELECT * FROM pg_blocking_pids(pid);` 逐层还原等待树。根因几乎都是应用层锁顺序：比如任务 A 先改 orders 再改 users，任务 B 反过来；或事务里先插入子表再由外键触发父表行锁，与另一个事务的父表更新互等；还有索引页锁（并发插入相邻 key）和批量大事务持有过多行锁。预防：①所有多行更新按主键排序后统一顺序；②事务尽量短，把查询、外部调用挪出事务；③批量任务错峰、控制每批规模；④必要时用 `SELECT ... FOR UPDATE` 预排序加锁。实践中的坑：死锁解决后要复查是否真的消除了锁环，而不是靠不断重试掩盖；高并发下相同 SQL 反复死锁，多半是锁顺序或外键设计问题，不是 SQL 本身的锅。

**延伸考点：** 外键约束为什么会把父表 UPDATE 变成锁来源？怎么用 lock_timeout 防止死锁检测不及时导致的无限等待？

---

### Q7. 数据库报 "not accepting commands to avoid wraparound data loss"，怎么办？

**问题：** 某天线上 PostgreSQL 报 "database is not accepting commands to avoid wraparound data loss"，应用全挂，这是怎么回事？autovacuum 平时该怎么调优避免？

**期望加分项：**
- 能说出 32 位事务 ID 回卷机制与冻结（freeze）原理，及 autovacuum_freeze_max_age 的强制冻结触发
- 能说出根因通常是长事务/idle in transaction 阻塞 vacuum 推进，导致 age 逼近 2 亿
- 能给出监控手段：age(datfrozenxid)、pg_database 与 pg_stat_user_tables 对照
- 能给出调优参数：scale_factor 调小、autovacuum_max_workers、cost_limit、per-table 配置
- 能说清 hot_standby_feedback 与长查询导致主库无法回收的关联
- 能给出应急处理：尽快杀长事务、手动 vacuum freeze，并区分「危险期」处置

**减分项：**
- 只说「跑一下 vacuum」，说不清为什么会到这一步
- 不知道 freeze 和 xid 回卷的概念
- 不知道长事务会 pin 住最老 xid 阻止 vacuum
- 调优只背参数名，给不出数值和依据

**解答：**
PostgreSQL 用 32 位事务 ID（约 21 亿个），靠「冻结」标记让旧元组对所有事务可见，从而允许 xid 复用；当某个库的 age(datfrozenxid) 逼近 2 亿（autovacuum_freeze_max_age 默认值）时 autovacuum 会强制做冻结 vacuum，一旦继续逼近 21 亿，数据库会进入只读保护拒绝一切新事务——这就是报错来源。根因几乎都是「长事务」：任何最老活跃事务（含 idle in transaction）会 pin 住最老的 xid，让 freeze 无法推进，age 只涨不降；备库开 hot_standby_feedback 且备库有长查询时同理。应急处理：先 `SELECT pid, state, xact_start, now() - xact_start FROM pg_stat_activity WHERE state='idle in transaction' OR (state='active' AND xact_start 很久之前);` 杀掉最老的长事务（`pg_terminate_backend`），再手动 `VACUUM (FREEZE)` 拉回 age。预防调优：把 `autovacuum_vacuum_scale_factor` 从 0.2 调小到 0.01-0.05（或对超大表用 per-table 的 `autovacuum_vacuum_threshold` 兜底），避免表涨到阈值才触发；`autovacuum_max_workers` 可提到 3-6；`autovacuum_vacuum_cost_limit` 调大到 1000-2000 提速。监控：对 age(datfrozenxid) 设置阈值告警（如 1.5 亿告警、1.8 亿紧急），并告警「最长事务时长」和 idle in transaction。实践中的坑：freeze 风暴——多张大表同时到达强制冻结阈值会集中刷盘打满 IO；备库开 hot_standby_feedback 会阻止主库回收被备库查询需要的老版本，让 bloat 和 age 同时恶化，要定期清理备库长查询；大表 vacuum 用 PG 13+ 的并行 vacuum（PARALLEL）提速。

**延伸考点：** 为什么 hot_standby_feedback 开在备库会同时导致主库 bloat 和 age 上升？怎么在不关掉它的前提下缓解？

---

### Q8. 大量会话阻塞在一个简单 INSERT 上，等待链怎么查、怎么解？

**问题：** 某个功能偶发卡死，pg_stat_activity 里看到几十个会话在等待一个锁，等待的语句是普通 INSERT，而源头是一个「idle in transaction」的长连接，怎么定位和处置？

**期望加分项：**
- 能说清 idle in transaction 是锁等待的头号来源：事务已跑完但没 COMMIT/ROLLBACK，持有的行锁/表锁不放
- 能给出锁链还原命令：pg_stat_activity 过滤 wait_event_type='Lock'、pg_blocking_pids() 逐层找根、pg_locks 对照
- 能给出处置：pg_terminate_backend 杀根因会话，并设置 statement_timeout/lock_timeout/idle_in_transaction_session_timeout 兜底
- 能指出 DDL 的 ACCESS EXCLUSIVE 锁会堵住一切读写，且与长查询互斥的典型事故
- 能联系线上实践：ORM 连接池把事务泄漏成 idle in transaction、连接池 max-lifetime 与事务长度的关系

**减分项：**
- 只看到「很多会话在等」，不会用 pg_blocking_pids 找根因
- 不知道 idle in transaction 会持锁
- 只杀等待者不杀根因，反复发作
- 不知道可以用 timeout 参数兜底防复发

**解答：**
这是典型的「锁等待链」问题：等待者只是一批简单 INSERT（或 UPDATE），它们要拿同一行/同一表的锁，但锁被一个长期未提交的事务占着。先看 `SELECT pid, state, wait_event_type, wait_event, now()-xact_start FROM pg_stat_activity WHERE wait_event_type='Lock';` 找到所有等待者；再用 `SELECT pg_blocking_pids(pid) FROM pg_stat_activity WHERE ...;` 逐层向上追，最终停在某个 `state='idle in transaction'` 或长时间 active 的会话上——它持有锁但没提交。处置：`pg_terminate_backend(根因pid)` 让它回滚释放锁；如果它是应用「忘提交」的僵尸连接，同时修应用。防止复发靠参数兜底：`idle_in_transaction_session_timeout=60s`（挂起事务超时自动断）、`lock_timeout`、`statement_timeout` 按业务设置。实践中的坑：①ALTER TABLE 等 DDL 默认拿 ACCESS EXCLUSIVE 锁，会和任何读写互斥——线上做过大表加列的都知道，低峰执行或先用 `lock_timeout` 试探；②连接池（如 HikariCP）归还连接时如果事务没提交，事务会跨连接继续挂着，要确认 ORM 事务边界完整；③杀错进程会误伤业务，先确认 pid 对应的应用连接来源（application_name/client_addr）；④`SELECT pg_terminate_backend` 对「正在等待锁的会话」也有效，但治标要治本。

**延伸考点：** 怎么区分锁等待和磁盘 IO 等待（wait_event 的不同取值）？为什么 `idle_in_transaction_session_timeout` 能治本而 statement_timeout 不行？

---

### Q9. 订单表的扩展字段想存 JSONB，哪些查询适合、GIN 索引怎么建、什么时候别用？

**问题：** 订单表有一批第三方回调的扩展字段，结构经常变，想存成 JSONB。哪些查询场景适合用 JSONB？GIN 索引怎么建？什么情况下不应该用 JSONB 而应该拆列或拆表？

**期望加分项：**
- 能给出适用判断：schema 多变、嵌套结构、查询不频繁或路径固定、只做少量路径过滤 → JSONB 合理
- 能给出 GIN 索引建法及支持的操作符：`CREATE INDEX ... USING GIN (data jsonb_path_ops);` 支持 `@>`、`?`、`?|`、`?&`
- 能说清 jsonb_path_ops 与默认 jsonb_ops 的取舍：前者体积小、只支持 @>，后者功能全
- 能说出反例：高频过滤字段、需要 join/强约束/外键、频繁更新子字段（整行重写）、需要范围查询的字段都不适合
- 能给出混合方案：稳定字段拆普通列 + 变动字段留 JSONB + 生成列（generated column）冗余可索引列
- 能联系线上实践：JSONB 无 schema 校验带来的脏数据治理成本

**减分项：**
- 一股脑推荐 JSONB，说不清什么查询能走索引、什么不能
- 不知道 GIN 索引支持哪些 JSONB 操作符，或以为范围查询能走 GIN
- 忽略 JSONB 的写放大：更新子字段是整行重写
- 不讨论与关系建模的取舍（何时该拆表）

**解答：**
先判断读写模式再决定。适合 JSONB 的场景：schema 确实多变（三方回调字段）、查询只按固定路径过滤（`WHERE data @> '{"status":"paid"}'`）、嵌套结构难以拆平。建索引：`CREATE INDEX idx ON orders USING GIN (data jsonb_path_ops);` ——支持 `@>`、`?`、`?|`、`?&` 四类操作符走索引；jsonb_path_ops 体积约为默认 jsonb_ops 的一半且更快，但只能支持 `@>` 一种，功能与体积二选一。不适合的场景要果断说不：①高频等值/范围过滤的字段（如订单金额、创建时间）应建普通列 + B-tree；②需要 join 的关联键、外键约束、非空约束——JSONB 全都没有；③频繁 UPDATE 单个子字段——JSONB 更新是整行重写（写放大 + 膨胀），高频更新的子字段拆列；④需要范围查询的数值字段。工程上的混合方案：稳定高频字段拆列，变动字段留 JSONB，再用生成列把 JSONB 里高频查询的路径冗余成普通列并建索引：`CREATE TABLE t (... paid_at timestamptz GENERATED ALWAYS AS ((data->>'paidAt')::timestamptz) STORED);`。实践中的坑：JSONB 不校验 schema，脏字段会在下游查询里爆雷，写入前要做数据校验；`->>` 返回 text、`->` 返回 jsonb，比较类型要对；GIN 索引在频繁更新的表上膨胀快，配合 autovacuum；pg13 起支持 `jsonb_path_query` 等 SQL/JSON 路径函数，但底层仍是整行存储。

**延伸考点：** JSONB 的 `?` 操作符和 `@>` 有什么区别，各自怎么走 GIN 索引？什么情况下应该把 JSONB 拆成子表（一对多）？

---

### Q10. LIKE '%关键词%' 很慢，还要做全文检索（高亮、相关度排序），方案怎么选？

**问题：** 商品名要支持模糊搜索：LIKE '%关键词%'，B-tree 索引完全用不上，查询慢；同时还要做全文检索（关键词高亮、按相关度排序）。PG 内怎么做？方案怎么取舍？

**期望加分项：**
- 能区分场景：前缀匹配（LIKE 'abc%'）走 B-tree；中缀模糊（'%abc%'）用 pg_trgm + GIN；全文检索用 tsvector/tsquery + GIN
- 能给出 pg_trgm 的建索引方式：`CREATE INDEX ... USING GIN (name gin_trgm_ops);` 及 ILIKE 大小写不敏感
- 能说出 pg_trgm 的原理（trigram 拆词）和边界：小于 3 字符的查询性能差、索引体积膨胀
- 能说清 FTS 的配置：zhparser/pg_jieba 做中文分词、setweight 权重、ts_rank 排序
- 能给出取舍：数据量级大、要求实时或高相关度时引入外部搜索引擎（ES）的边界
- 能联系线上实践：查询规范化（大小写、全半角）对命中率的影响

**减分项：**
- 不知道 pg_trgm 这个扩展，只会说「加索引」
- 以为 tsvector 能直接处理中文，不提分词扩展
- 混淆 trigram 和 FTS 的适用边界
- 不考虑索引体积、写放大和数据量级，一律推 PG 内方案

**解答：**
按查询形态分三层。①前缀匹配 `LIKE 'abc%'`：普通 B-tree 走 range scan 即可，无需特殊处理。②中缀模糊 `LIKE '%abc%'` / ILIKE：装 pg_trgm 扩展，`CREATE EXTENSION pg_trgm; CREATE INDEX ON products USING GIN (name gin_trgm_ops);` 原理是把文本拆成 3 字符的 trigram 集合建倒排索引，查询时同样拆词求交集。注意：长度小于 3 的搜索词基本退化为全扫（可用 `pg_trgm.word_similarity_threshold` 等参数缓解），索引体积约为数据量的 1.5-2 倍且维护成本高，只适合读多写少的表。③全文检索：`CREATE INDEX ON posts USING GIN (to_tsvector('simple', body));` 查询用 `to_tsquery`，排序用 `ts_rank`，高亮用 `ts_headline`；中文必须装 zhparser/pg_jieba 等分词扩展，`'simple'` 配置对中文等于整句一词，索引形同虚设。权重：`setweight(to_tsvector(title),'A') || setweight(to_tsvector(body),'B')` 让标题命中排前面。方案取舍：PG 内置 FTS 够用于中小规模、同义词需求简单的场景；一旦要求近实时索引、高召回（拼音/错别字/同义词）、亿级文档，就该引入 Elasticsearch，PG 只做源库。实践中的坑：查询词要规范化（转小写、去全角）；pg_trgm 的 GIN 索引在频繁 UPDATE 的表上膨胀快，需配合 autovacuum；`ILIKE '%x%'` 与 `LIKE` 的索引匹配要求操作符与索引 operator class 一致。

**延伸考点：** pg_trgm 的 GIN 和 GiST 两种索引怎么选？tsvector 对中文无效时，zhparser 的分词效果如何验证（token 覆盖率）？

---

### Q11. 订单流水一年 20 亿行，查询越来越慢，分区怎么设计、partman 要注意什么？

**问题：** 订单流水表一年 20 亿行，按时间过滤越来越慢，打算做声明式分区（declarative partitioning）。怎么设计分区键和分区粒度？pg_partman 自动化建分区时要注意什么坑？

**期望加分项：**
- 能说清分区键必须匹配最频繁的时间过滤条件，且分区裁剪（partition pruning）依赖 WHERE 带分区键
- 能给出设计：RANGE 按月/按周 + 在分区键上建索引，主键必须包含分区键列（PG 11 起的要求）
- 能说出 pg_partman 的作用：自动创建未来分区、按保留期删除历史分区
- 能指出常见坑：分区数过多导致 planner 开销与 DDL 锁、default 分区陷阱、跨分区更新（移动行）的代价
- 能给出监控与运维手段：pg_partition_tree、分区裁剪失效的排查
- 能联系线上实践：分区 vs 分库分表（PG 无内置 sharding）的边界判断

**减分项：**
- 把分区和分库分表混为一谈，或以为分区能解决所有性能问题
- 不知道主键必须含分区键的限制
- 不知道 default 分区、ATTACH/DETACH 的锁行为
- 忽略查询不带分区键时退化为全分区扫描

**解答：**
先确认收益来源：分区让查询能裁剪掉无关子表（partition pruning），让旧的冷数据可独立删除（DROP PARTITION 比 DELETE 快几个数量级），让 vacuum/索引维护按分区并行。设计要点：分区键选查询里最稳定的时间过滤字段，`PARTITION BY RANGE (created_at)`，粒度按写入与查询频率平衡（月分区通常够，超高吞吐可按周/按天）；主键/唯一约束必须包含分区键列（PG 11+），例如 `PRIMARY KEY (id, created_at)`；查询必须带分区键且不做函数包裹（如 `WHERE date(created_at)='2024-01-01'` 在旧版本不裁剪，PG 11 起部分支持分区键函数，PG 16 支持更多形式），否则退化为扫所有分区。pg_partman：`create_parent(... 'p_interval', '1 month')` 自动建未来 N 个分区、定期 `run_maintenance()` 建新区、按 `retention` 策略 drop/merge 旧分区。实践中的坑：①分区数别失控（几千个分区会让 planner 和每次 ATTACH 的锁变重）；②default 分区是隐形杀手——路由不到具体分区的数据落进 default 后永远不再自动迁移，查询扫 default 全表，要监控 default 分区是否有数据；③更新分区键会把行搬到另一个分区，锁开销大（PG 17 前），设计上避免更新分区键；④每个子分区都要单独建索引（在父表上建会自动分发）和单独 vacuum；⑤分区表上的 ON CONFLICT、外键引用在旧版本受限，先确认版本能力。最后想清楚：分区解决的是「单表体积与维护」问题，不是并发和容量扩展问题，真要水平扩展再考虑 citus 或分库分表。

**延伸考点：** 分区裁剪失效的常见原因有哪些？为什么默认分区里有数据会导致查询性能断崖？

---

### Q12. 换了 64G 内存的机器，参数怎么调？checkpoint 风暴是怎么回事？

**问题：** 给 PostgreSQL 换了台 64G 内存、SSD 磁盘的机器，除了默认参数什么都没动。要调哪些参数、调完怎么验证收益？同事说改完 max_wal_size 后还会出现 checkpoint 风暴，这又是什么？

**期望加分项：**
- 能给出内存类参数的量化建议：shared_buffers 约 16G（25%）、effective_cache_size 约 48G（75%）、work_mem 按并发计算总账、maintenance_work_mem
- 能说清 work_mem 的分配模型：每个 sort/hash join 操作每个进程一份，100 并发 × 100MB 会爆内存，这是最容易被改炸的参数
- 能讲清 checkpoint 机制：max_wal_size 到顶触发全量脏页刷盘，IO 尖峰（checkpoint 风暴），调大 max_wal_size + checkpoint_completion_target 摊平
- 能给出验证手段：pgbench、pg_stat_statements 前后对比、pg_stat_bgwriter/checkpointer 视图
- 能区分哪些参数要重启（shared_buffers、max_connections）哪些 SIGHUP 即可
- 能联系线上实践：synchronous_commit=off、random_page_cost 等 SSD 时代参数的调整

**减分项：**
- 只会背「shared_buffers 25%」，讲不清为什么和验证方式
- 盲目把 work_mem 调到 1G，不算并发总账
- 不知道 checkpoint 风暴的成因和 max_wal_size/completion_target 的关系
- 改完不做压测对比，说不清收益

**解答：**
按「内存、checkpoint、并发」三条线调。内存：`shared_buffers` 设 16G（物理内存 25% 是经验上限，再大收益递减且管理开销上升；超过 32G 建议开 huge_pages）；`effective_cache_size` 设约 48G（让规划器认为大部分数据可缓存，影响索引扫描的选择）；`work_mem` 默认 4MB 对小表 sort 偏小，但它是「每操作 × 每进程」分配——100 并发 × 10 个操作 × 100MB = 100G 直接 OOM，正确做法是保持适中（如 16-64MB），对单条大查询用 `SET LOCAL work_mem` 临时放大；`maintenance_work_mem` 提到 1-2G 加速 vacuum 和建索引。checkpoint：`max_wal_size` 默认 1GB 偏小，改大（如 8-16GB）降低 checkpoint 频率；`checkpoint_completion_target=0.9` 把脏页刷盘摊到两个 checkpoint 之间，避免刷盘尖峰——不调它，WAL 写满就会触发「全量刷盘风暴」，表现为 IO 打满、写放大、查询长尾延迟。并发：`max_connections` 按应用估算（配合 PgBouncer 可调小），`wal_buffers` 默认即可。验证：pgbench 压测对比 TPS/延迟，`pg_stat_statements` 对比慢查询均值，`pg_stat_bgwriter`/`pg_stat_checkpointer` 看 checkpoint 次数和耗时；改完用 `SHOW` 和压测数据说话，别凭感觉。实践中的坑：`fsync` 别关（SSD 上收益有限但数据风险极大）；`synchronous_commit=off` 只适合能容忍丢最后 1 个事务的批处理场景；改 shared_buffers/max_connections 要重启，规划好维护窗口。

**延伸考点：** 为什么说 work_mem 是线上 OOM 的头号嫌疑参数？SSD 时代 random_page_cost 该调成多少、为什么？

---

### Q13. 报表从库和「部分表同步到别的库」，分别用物理复制还是逻辑复制？

**问题：** 需求有两个：①给报表搭一个 PG 只读从库；②把订单表和用户表实时同步到另一个业务域的库（需要改 schema 或只同步部分表）。两个场景分别选什么复制方案？各有什么坑？

**期望加分项：**
- 能说清物理复制（流复制）的定位：WAL 字节级、整个实例一致副本、只读备库、延迟低、DDL 自动同步
- 能说清逻辑复制（发布订阅）的定位：按表按行复制、可选择性、订阅端可写、支持跨版本/跨平台
- 能给出同步/异步的选择：synchronous_standby_names + synchronous_commit 控制 RPO
- 能说出物理复制的坑：hot_standby_feedback 与主库 bloat、备库 promote、slot 管理
- 能说出逻辑复制的坑：初始同步要 slot（WAL 堆积风险）、不复制 DDL、序列不同步、主键/副本标识要求、冲突处理
- 能给出监控手段：pg_stat_replication 的 sent_lsn/replay_lsn 计算延迟

**减分项：**
- 分不清两种复制的边界，或以为逻辑复制能替代物理复制做高可用
- 不知道逻辑复制不复制 DDL、序列不同步这些关键限制
- 不知道复制槽（replication slot）不消费会撑爆 pg_wal
- 说不清同步复制怎么配置、RPO 怎么量化

**解答：**
场景①报表从库用物理流复制：`pg_basebackup` 建备库，主库 `wal_level=replica`，备库回放 WAL 字节级，整个实例一致，DDL 也会复制，延迟毫秒级。监控延迟：`SELECT now() - pg_last_xact_replay_timestamp(), replay_lsn FROM pg_stat_replication;` 对比 sent_lsn 与 replay_lsn。要严格 RPO 就开同步复制：`synchronous_standby_names='first 1 (*)'` + `synchronous_commit=remote_apply`（或 remote_write），代价是备库故障时主库写入阻塞。场景②用逻辑复制（PG 10+）：主库 `CREATE PUBLICATION pub FOR TABLE orders, users;` 目标库 `CREATE SUBSCRIPTION sub CONNECTION '...' PUBLICATION pub;` 支持只同步指定表、订阅端可写、可跨大版本（甚至 PG→PG 之外的数据平台）。逻辑复制的坑要背熟：①初始同步需要消费复制槽，slot 长期无人消费会让 pg_wal 无限增长撑爆磁盘（监控 `pg_replication_slots` 的 active）；②DDL 不同步——改表结构要两边手工执行；③序列不同步——insert 到订阅端用序列做主键会撞键，需手动设置序列起点；④表必须有主键或 `REPLICA IDENTITY FULL`（无主键表旧版本整行复制，慢且膨胀）；⑤行冲突——订阅端同一主键被外部修改会导致 apply 报错停摆，要设计冲突处理或保证订阅端只读。实践中的坑：逻辑复制不能替代物理复制做高可用（只复制数据不复制所有对象）；切换主从后 slot 归属要重新规划；延迟与冲突监控（`pg_stat_subscription`）必须纳入告警。

**延伸考点：** 逻辑复制在订阅端有主键冲突时 apply 会怎样、怎么恢复？物理备库 promote 后原来的复制槽和备份链怎么处理？

---

### Q14. 要求 RPO 5 分钟内、能恢复到任意时间点，备份方案怎么做？pg_dump 和 pg_basebackup 怎么分工？

**问题：** 公司要求数据库 RPO 控制在 5 分钟以内，并且出事故后能恢复到任意时间点（PITR）。备份方案怎么设计？pg_dump 和 pg_basebackup 分别什么时候用？恢复演练怎么验证？

**期望加分项：**
- 能给出 PITR 的标准组合：pg_basebackup 全量基线 + archive_command 持续归档 WAL + recovery_target_time 恢复
- 能说清两者分工：pg_dump 是逻辑备份（单库/表级、跨版本迁移），pg_basebackup 是物理备份（整个实例、支持 PITR）
- 能给出归档配置片段：`archive_command='cp %p /wal_archive/%f'`，及失败重试的重要性
- 能说明 RPO 达标逻辑：RPO=5min 意味着备份频率 + WAL 归档间隔都须小于 5 分钟，RPO=0 需要同步复制
- 能提到生产级工具：barman/wal-g/pgBackRest 管理备份生命周期与自动恢复
- 能强调恢复演练（restore 到临时实例验证）是备份方案的一部分而非可选

**减分项：**
- 分不清逻辑备份和物理备份，或以为 pg_dump 能做 PITR
- 不知道 PITR 需要「基线 + WAL 归档 + 恢复目标」三要素
- archive_command 写死但没考虑失败重试，或不知道 archive_status 怎么查
- 只谈备份不谈恢复演练和数据校验

**解答：**
RPO≤5min + 任意时间点恢复 = 物理备份 + WAL 归档 + PITR，三要素缺一不可。搭建：①基线：`pg_basebackup -D /backup/base -Ft -z -Xs`（-Xs 把备份期间 WAL 一并收走，避免 WAL 间隙）；②归档：`archive_mode=on`、`archive_command='test ! -f /wal_archive/%f && cp %p /wal_archive/%f'`，脚本必须能处理失败重试，否则归档断档时 pg_wal 会无限堆积且无法 PITR；验证归档用 `SELECT pg_switch_wal();` 后检查 archive_status 目录出现 .done。③恢复：停库 → 解包基线 → 建 recovery.signal（PG 12+，旧版为 recovery.conf）→ 配置 `restore_command`、`recovery_target_time='2026-08-01 10:00:00'` 启动即回放到目标时刻，`recovery_target_inclusive` 控制含不含目标点。分工：pg_dump 适合备份单库/单表、跨大版本迁移、导出 schema；它是逻辑备份，重放慢、不能 PITR；pg_basebackup 是全实例物理一致性快照。生产建议直接用 barman/pgBackRest/wal-g 管理「定期基线 + 持续归档 + 自动恢复测试」，别手写脚本。实践中的坑：①备份不等于恢复——必须周期性在临时实例演练 `pg_restore`/恢复基线到目标时间，检查数据量与行数校验（checksum）；②RPO=0 需要同步复制 + 每事务等待，5 分钟 RPO 意味着归档中断告警阈值要小于 5 分钟；③恢复出来的实例要尽快接入监控，避免「备份存在但恢复不了」；④pg_dump 的一致性靠事务快照，大库 -Fc + --jobs 并行可提速，但并行恢复要按依赖排序。

**延伸考点：** WAL 归档断档时怎么发现和补救？恢复到「目标时间点之前最后一个事务」和「目标时间点」有什么区别？

---

### Q15. 用 Patroni 搭高可用集群，自动故障转移时怎么避免脑裂和数据丢失？

**问题：** 要搭一套 PostgreSQL 高可用：自动故障转移、尽量不丢数据。用 Patroni + etcd 怎么理解它的工作机制？故障切换时怎么避免「脑裂」（旧主还在写）和丢事务？

**期望加分项：**
- 能讲清 Patroni 的原理：以 etcd 等 DCS 为共识存储 leader 租约，Patroni agent 竞争/续约 leader key，失联后多数派选举新主
- 能说清避免脑裂的关键：leader 租约 + DCS quorum（多数派），旧主在失去租约后拒绝写入；配合 fencing（pg_rewind 回追）
- 能说清丢数据的控制：异步复制下 failover 必然丢最后未同步事务，要 RPO≈0 必须同步复制（synchronous_standby_names + synchronous_commit）
- 能提到 Patroni 的 REST API（/health、/primary、/replica）与配套的 VIP/HAProxy 切换
- 能指出坑：etcd 本身的高可用、网络分区时 quorum 丢失导致集群拒写、切换后应用连接切换
- 能联系线上实践：pg_rewind 需要 wal_log_hints=on 或 data checksums；备份/监控要独立于 Patroni

**减分项：**
- 说不清 Patroni 和 etcd 各自的角色，或以为 etcd 存数据
- 不知道异步复制 failover 会丢事务，没讨论同步复制
- 不了解租约/多数派机制，答不出怎么防脑裂
- 不知道 pg_rewind、时间线（timeline）这些切换后的关键步骤

**解答：**
Patroni 的思路是「让数据库具备自愈能力」：每个 PG 实例上跑一个 patroni agent，向 DCS（etcd/consul/zookeeper）注册并竞争一个 leader 租约；只有拿到租约的实例才允许以主库模式启动，其余以备库模式跟随。健康检查通过 REST API（/health）暴露，Patroni 周期续约；主库失联（网络分区或宕机）时，其余节点必须满足 DCS 多数派（quorum）才能选举新主，从这点就切断了脑裂的主要来源——旧主在租约过期后不再接受写入，且 Patroni 会尝试给旧主发 SIGSTOP/切换只读。防丢数据：异步复制下主库崩溃瞬间，最后几个未同步事务必然丢失，RPO>0；要 RPO≈0 必须开同步复制：`synchronous_standby_names='first 1 (*)'` + `synchronous_commit=remote_apply`，代价是备库故障时写入被阻塞——所以生产常见「同步一个 + 异步多个」的 quorum 配置。切换细节：新主 promoted 后时间线（timeline）+1，旧主恢复后通过 pg_rewind 基于新主的时间线回追（要求 `wal_log_hints=on` 或 data checksums，否则报错）。实践中的坑：①etcd 至少 3 节点且独立于数据库服务器，否则失去 quorum 时集群直接拒写；②Patroni 不负责应用连接切换——需要 HAProxy/VIP/DNS 配合；③备份和监控不要挂在 Patroni 管理的节点角色上（角色会变）；④故障切换不是零风险，要定期演练「杀主库」看 RTO/RPO 实测值。

**延伸考点：** 网络分区（旧主和新主各自可见部分节点）时，Patroni 靠什么机制保证不会出现两个主？synchronous_commit=remote_apply 与 remote_write 的差异和取舍？

---

### Q16. 高峰期报 too many clients，怎么用 PgBouncer 解决？事务级池有哪些坑？

**问题：** 应用是 Java 连接池（HikariCP）直连 PostgreSQL，高峰期报 "sorry, too many clients already"，数据库 max_connections 只有 100，但前端可能要几千个连接。怎么解决？PgBouncer 的事务级池（transaction pooling）有哪些必须知道的坑？

**期望加分项：**
- 能说出 PG 每连接一个进程的模型：连接数受内存和 CPU 限制，池化是必选项而非可选项
- 能说清 PgBouncer 三种模式：session/transaction/statement pooling，生产常用 transaction pooling
- 能说出 transaction pooling 的坑：prepared statement、临时表、SET 会话级状态、LISTEN/NOTIFY、pg_backend_pid() 等会话相关特性
- 能给出配置要点：max_client_conn、pool_mode、server_idle_timeout、auth_type（scram-sha-256）、连接数估算
- 能说清应用侧也要收敛：HikariCP maximumPoolSize 别开几百，DB 连接池与 PgBouncer 是两层
- 能联系线上实践：PgBouncer 本身要高可用（多实例/负载均衡），及备选方案 pgpool-II/pgagroal

**减分项：**
- 只想到把 max_connections 调大，不说连接池化
- 不知道 PgBouncer 有事务级/会话级模式之分
- 不知道 transaction pooling 下 prepared statement 和临时表会炸
- 以为有了 PgBouncer 应用侧连接池可以随意开大

**解答：**
PostgreSQL 是「每连接一个后端进程」的架构，连接成本高、内存占用大（几 MB 起步），max_connections 不可能无限调大，正确解法是连接池。PgBouncer 三种模式：session pooling（一个客户端连接独占一个 server 连接，省的是连接建立开销）、transaction pooling（事务结束即归还连接，几百个应用连接只占几十个 DB 连接，生产主流）、statement pooling（慎用，事务中途归还会破坏语义）。transaction pooling 的坑必须背：①prepared statement 绑在具体后端进程上，归还后再取出的连接可能是另一条——PG 14 前的 PgBouncer 基本不支持，应用要么禁用 prepare（JDBC prepareThreshold=0），要么升级到支持 PREPARE 的新版本；②临时表是会话级的，事务结束连接归还后临时表就没了，应用不能再依赖；③SET 语句只在该事务内生效，想全局生效要写进配置（server_reset_query 或 startup 参数）；④LISTEN/NOTIFY、`select pg_backend_pid()`、咨询锁这类会话特性在池化后会串号。配置要点：`pool_mode=transaction`、`max_client_conn` 按前端估算、`server_idle_timeout` 控制空闲回收、`auth_type=scram-sha-256`（PgBouncer 1.17+ 才支持，之前只能 md5）。应用侧 HikariCP 的 maximumPoolSize 也要收敛（单实例 10-30 足够，靠 PgBouncer 吃并发），两层都要合理，否则连接数还是会被打爆。实践中的坑：PgBouncer 是单点，要部署两节点 + TCP 负载均衡；慢查询不会被池化解决，池只是「复用连接」，治标要治本。

**延伸考点：** transaction pooling 下 REPEATABLE READ 隔离级别的事务语义会被破坏吗，怎么规避？为什么 PgBouncer 官方不建议 statement pooling？

---

### Q17. 从 MySQL 迁过来后，字符串排序结果和原来不一样，等值查询还变慢，为什么？

**问题：** 从 MySQL 迁到 PostgreSQL 后，某个按字符串排序的接口结果顺序和原来不一样，而且 `WHERE name = 'ABC'` 查不到 'abc' 的数据，查询还很慢。可能是 collation 的问题，怎么排查和解决？

**期望加分项：**
- 能点破差异本质：MySQL 默认 utf8mb4_general_ci 大小写不敏感、重音不敏感；PG 默认按系统 LC_COLLATE（如 en_US.UTF-8/zh_CN.UTF-8）区分大小写且按语言规则排序
- 能给出排查命令：SHOW lc_collate / pg_database 的 datcollate、`ORDER BY name` 与 `name COLLATE "C"` 对比
- 能给出解法：建库时选对 collation（如 ICU）、列级 COLLATE、citext 扩展做大小写不敏感、或应用层处理
- 能说清 collation 影响索引：索引使用列的 collation，查询 collation 不一致会走不了索引（等值查询变慢的直接原因）
- 能指出硬约束：数据库级 collation 建库后不能改，只能新库 dump/restore
- 能联系线上实践：非确定性 collation（ICU NONDETERMINISTIC）的取舍

**减分项：**
- 只想到「加索引」，答不出 collation 是根因
- 不知道数据库级 collation 事后不可改
- 不知道 citext / 表达式索引（lower()）这些大小写不敏感的工程手段
- 不知道非 ASCII、Emoji、全半角排序差异这类边界

**解答：**
先确认根因：MySQL 的 utf8mb4_general_ci 是不区分大小写、不区分重音的排序规则，而 PostgreSQL 默认继承操作系统 locale（如 en_US.UTF-8 或 zh_CN.UTF-8），按语言规则排序且区分大小写。三个症状对应三个机制：①排序顺序不同——不同 collation 的排序规则本身不同；②`name='ABC'` 查不到 'abc'——PG 的默认 collation 大小写敏感，等值比较不相等；③变慢——这个查询在 MySQL 走 utf8mb4 默认不区分大小写的索引，而 PG 里 `name` 的 B-tree 索引是大小写敏感的，应用想查任意大小写就需要 `lower(name)` 表达式，表达式没建索引就全表扫。排查：`SHOW lc_collate;`（或 `pg_database` 的 datcollate），再对比 `SELECT 'abc' = 'ABC';`。解决分几层：建库前——选对模板与 collation（如 `LC_COLLATE='en_US.UTF-8'`，或用 ICU：`CREATE COLLATION und_ci (provider=icu, locale='und-u-ks-level2')` 做大小写不敏感）；已上线——列级 `ALTER COLUMN name TYPE text COLLATE "und_ci"` 重建索引，或对大小写不敏感等值查询建表达式索引 `CREATE INDEX ON t (lower(name));` 并把查询写成 `WHERE lower(name) = 'abc'`；或装 citext 扩展直接把列类型改为 citext。硬约束：数据库级 collation 建库后不可更改，改只能新建库 + dump/restore，所以建库前就要想清楚。实践中的坑：非确定性 collation（ICU）不能用于模式匹配类索引的某些场景、有性能开销；`ILIKE` 可以大小写不敏感匹配但一般不走 B-tree（可用 pg_trgm）；全半角、重音、Emoji 的排序差异是迁移后「看起来随机」的经典来源，回归用例要覆盖。

**延伸考点：** 为什么说 collation 选择会永久锁定在库里、迁移代价极大？ICU collation 和系统 locale 的排序结果在中文场景有什么差异？

---

### Q18. 「附近的门店」「某区域内的订单」这类空间查询，怎么在 PG 里做？为什么空间数据必须建空间索引？

**问题：** 业务要做「附近 3 公里的门店」「某多边形区域内的订单」，数据量是几千家门店、百万级订单带经纬度。PostGIS 怎么用？为什么这类查询必须依赖空间索引，普通 B-tree 不行？

**期望加分项：**
- 能说出 PostGIS 使用流程：CREATE EXTENSION postgis、GEOMETRY/GEOGRAPHY 列、SRID 概念、GiST 索引
- 能说出空间索引原理：GiST 基于 bounding box（MBR）先粗筛、再精确计算，避免全表逐点计算距离
- 能给出典型 SQL：ST_DWithin（先粗筛后精算、能走索引）vs ST_Distance（容易全表计算）
- 能讲清 GEOMETRY（平面）与 GEOGRAPHY（球面）的取舍：经纬度距离用 GEOGRAPHY 更准但慢，小范围用 GEOMETRY 快
- 能指出坑：SRID 混用导致距离错乱、对列包 ST_Transform 导致索引失效、索引膨胀
- 能联系线上实践：坐标是 POINT 还是复杂多边形、数据更新频率对 GiST 维护的影响、H3 等替代方案

**减分项：**
- 不会写空间 SQL（ST_DWithin/ST_Intersects），只会说「加索引」
- 不知道 GiST 空间索引和普通 B-tree 的本质区别
- 不知道 SRID 和投影坐标系的概念，或 GEOMETRY/GEOGRAPHY 混用
- 用 ST_Distance 做过滤导致索引失效还不知道为什么

**解答：**
PostGIS 在 PG 里加一层空间类型和函数。基本流程：`CREATE EXTENSION postgis;` 建表加空间列 `geom geometry(Point, 4326)`（4326 是 WGS84 经纬度坐标系，SRID 必须统一），建空间索引 `CREATE INDEX ON orders USING GIST (geom);`。为什么必须空间索引：距离/包含判断是二维问题，B-tree 只能处理一维有序，GiST 的做法是对每个几何对象维护一个 bounding box（MBR），索引先按 MBR 快速排除明显不相交的对象，剩下的再精确计算——没有它，几百万行要逐行做球面距离计算。典型查询：附近门店 `SELECT * FROM stores WHERE ST_DWithin(geom, ST_MakePoint(lng,lat)::geography, 3000);`——ST_DWithin 语义是「距离小于阈值」且能走 GiST 索引；而 `ST_Distance(...) < 3000` 这类写法通常要全表算距离，是经典的索引失效写法。GEOMETRY vs GEOGRAPHY：GEOGRAPHY 用球面几何，经纬度距离/面积更符合直觉（公里制），但计算开销大；GEOMETRY 是平面计算、快，适合投影坐标系或小范围近似。实践中的坑：①SRID 不一致（比如有人存 3857 Web 墨卡托、有人存 4326）距离会错乱到离谱，写入和查询都要校验；②查询时对索引列做转换（`ST_Transform(geom, ...)`）会让索引失效，转换应提前落到列上；③空间数据频繁 UPDATE 时 GiST 索引膨胀快，靠 autovacuum 和定期 REINDEX；④数据量特别大且只做「附近点」查询时，可对比 Uber H3 六边形网格方案，用字符串前缀索引代替空间索引。

**延伸考点：** ST_DWithin 为什么能走索引而 ST_Distance 不能？坐标用 GEOGRAPHY 存经纬度时，建索引和查索引需要注意什么（如 geometry_columns 视图）？

---

### Q19. 把 MySQL 线上库迁到 PostgreSQL，最容易踩哪些坑？迁移方案怎么设计？

**问题：** 公司决定把一个 MySQL 线上库迁移到 PostgreSQL。除了 SQL 方言，还有哪些差异是迁移后最容易在线上爆雷的？整体迁移方案（全量 + 增量 + 切换）怎么设计？

**期望加分项：**
- 能列出高价值差异清单：默认隔离级别语义、自增列、大小写与 collation、类型映射、隐式提交、GROUP BY 严格性、ON DUPLICATE KEY 语法
- 能点出行为差异而非语法差异：MySQL RR 用 next-key 锁易死锁 vs PG RC 快照读；PG 的 RR 下写冲突直接 40001
- 能给出迁移链路：pgloader/自研全量 + 增量回放（binlog 消费）双轨校验 + 灰度切换
- 能给出校验手段：行数、checksum、抽样对比、双跑对比
- 能提醒回滚方案：保留 MySQL 只读一段观察期、灰度切换
- 能联系线上实践：时区、float 精度、字符串长度（varchar 语义差异）这些容易被忽略的坑

**减分项：**
- 只提语法差异，答不出隔离级别、锁行为这类语义差异
- 不知道 MySQL 双引号字符串与 PG 标识符的语义冲突
- 迁移方案只有「dump 导入」，没有增量同步、校验、回滚
- 忽略时区、精度、collation 这些隐性数据差异

**解答：**
迁移的坑分「语法层」和「行为层」，行为层最致命。行为层：①隔离级别——MySQL 默认 RR（next-key 锁、间隙锁，并发下易死锁），PG 默认 RC；两边 RR 语义也不同：PG 的 RR 下并发写同一行直接抛 40001 让应用重试，业务代码要做好兼容；②大小写——MySQL 表/列名大小写规则宽松，PG 未加引号的标识符折叠成小写，双引号包住的名字区分大小写，迁移时名字最好统一小写；③collation——MySQL 默认大小写不敏感，PG 默认敏感（见上题），等值查询语义变了、索引也要重设计；④隐式提交——MySQL DDL 隐式提交、PG 的 DDL 是事务性的（可以回滚，但也意味着长事务风险）。语法层：AUTO_INCREMENT → `GENERATED ALWAYS AS IDENTITY`；`ON DUPLICATE KEY UPDATE` → `ON CONFLICT ... DO UPDATE`；`INSERT IGNORE` → `ON CONFLICT DO NOTHING`；`GROUP BY` 在 PG 严格（SELECT 的非聚合列必须出现在 GROUP BY 里，MySQL 默认宽松）；时间类型——MySQL datetime 无时区，PG 用 `timestamptz` 并在迁移时统一时区换算，否则数据「错 8 小时」；浮点/字符串语义（varchar 的尾随空格处理不同）。迁移方案：全量（pgloader 或双写导出）→ 增量（消费 MySQL binlog 通过 Canal/Debezium 回放到 PG，或短停窗全量 + 增量窗口）→ 数据校验（行数 + 字段级 checksum + 抽样 + 业务双跑）→ 灰度切换（先读流量切 PG，观察期保留 MySQL 只读兜底，确认后切写流量）。实践中的坑：大表迁移要在停窗/限流下做，避免锁和 IO 打爆源库；先跑「SQL 兼容性扫描」把报错和语义偏差提前暴露；迁移后压测（PG 和 MySQL 的 join 策略、索引行为都不同，原 MySQL 慢查询优化手段要重新做一遍）。

**延伸考点：** MySQL 的 REPEATABLE READ 和 PG 的 REPEATABLE READ 在幻读/写冲突上行为差异的具体场景？迁移后怎么用工具（如 pgloader 的 compare 或自研脚本）做数据一致性校验？

---

### Q20. 接手一个线上 PG 实例，从零搭一套能定位问题的监控，要采哪些指标？事后怎么用监控复盘？

**问题：** 接手一个线上 PostgreSQL 实例（无任何监控），要求搭一套「能定位问题」的监控体系：数据库侧要采哪些指标？慢 SQL 怎么统计？某次线上事故后怎么用监控数据复盘根因？

**期望加分项：**
- 能按层给出指标清单：可用性/复制、资源负载、连接与锁、垃圾回收与膨胀、WAL 与 checkpoint、慢 SQL
- 能点出 PG 特有指标：age(datfrozenxid)、n_dead_tup、replication lag、idle in transaction、等待事件（wait_event）
- 能给出慢 SQL 方案：pg_stat_statements（shared_preload_libraries 预加载）+ log_min_duration_statement + auto_explain 抓计划
- 能给出工具选型：Prometheus + postgres_exporter / pgwatch2，阈值与告警分级
- 能描述复盘方法：时间线对齐（应用侧 + DB 侧）→ 等待事件分布 → top SQL → 参数/变更对照
- 能指出常见监控盲区：只采系统指标不采等待事件、pg_stat_statements 不 reset 导致统计失真、备份与复制状态无人盯

**减分项：**
- 只列 CPU/内存/磁盘，一个 PG 特有的指标都说不出
- 不知道 pg_stat_statements 要预加载、需要重启才能生效
- 复盘中不会用等待事件和 pg_stat_activity 快照还原现场
- 阈值全凭感觉，给不出量化告警线

**解答：**
监控分层设计。①可用性与复制：实例 up、主从状态、复制延迟（pg_stat_replication 的 sent_lsn - replay_lsn）、备份任务状态。②资源与负载：连接数使用率（max_connections 的百分比）、idle in transaction 会话数、长事务时长（> 阈值告警）、等待事件聚合（pg_stat_activity 的 wait_event_type/wait_event，按 IO/Lock/Client 分类——这是 PG 排障的第一入口，比 CPU 曲线更能定位根因）、缓存命中率（pg_stat_database 的 blks_hit 比例，低于 95% 关注）。③存储与垃圾：表/索引大小、n_dead_tup 与 last_autovacuum（dead tuple 堆积告警）、age(datfrozenxid)（1.5 亿告警、1.8 亿紧急）、pg_wal 目录大小。④慢 SQL：pg_stat_statements 需要写进 `shared_preload_libraries` 并重启实例才能生效（这是常见部署遗漏），用归一化统计 top SQL 的 mean_time/max_time/calls，定期 reset 避免统计失真（pg_stat_statements_info 看溢出）；配合 `log_min_duration_statement` 落明细日志 + `auto_explain`（log_min_duration 阈值内自动打印执行计划）复盘时直接还原慢 SQL 计划。⑤checkpoint：pg_stat_bgwriter/checkpointer 的 checkpoints_timed 与 req 比例、每次耗时。工具选型：Prometheus + postgres_exporter（或 pgwatch2）采集，告警分级（P1：实例挂、复制断、age 紧急；P2：连接数 >80%、复制延迟超阈值、dead tuple 比例高）。复盘方法：对齐时间线（应用发布/流量峰值与 DB 指标同轴）→ 看等待事件分布锁定类别（锁等待？IO？）→ 用 pg_stat_statements 找 top SQL → 对照参数和 schema 变更记录。实践中的坑：只采主机指标不采等待事件等于没监控；慢查询只记日志不关联执行计划，复盘全靠猜；备份/复制告警缺失是「有监控仍出事」的高频原因。

**延伸考点：** auto_explain 的 log_min_duration 参数怎么设置才能既抓到慢 SQL 又不刷爆日志？pg_stat_statements 里 mean_time 正常但 max_time 异常大，可能是什么问题、怎么进一步定位？