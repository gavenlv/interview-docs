# 数据 · ClickHouse（面试题库）

本文件聚焦 ClickHouse 在真实数仓/OLAP 工程中的落地能力，覆盖表引擎选型（MergeTree 家族）、分区与索引设计、写入与批量导入、查询优化（PREWHERE/projection/物化视图）、分布式表与副本、ZooKeeper/Keeper 依赖、内存与磁盘管理（parts、merge、TTL）、数据一致性、实时/离线链路集成（Kafka、S3/HDFS）、字典与低基数优化，以及大集群运维与故障恢复。题目均为场景化问题，重点考察候选人在大数据量、高吞吐写入、低延迟查询约束下的取舍判断、量化依据与排障链路，不考概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

---

### Q1. 报表表和「取最新」表分别选什么引擎？

**问题：** 业务要建两张表：一张「订单按天汇总」的报表表，一张「用户最近一次登录」的查询表。数据都来自同一份明细，你会分别选什么表引擎？为什么？

**期望加分项：**
- 能说出 MergeTree 是唯一支持主键/分区/副本的底座，其余引擎都是在其上做能力叠加
- 能结合写入与查询模式说明选型依据：报表表是「预聚合 + 覆盖写」，登录表是「取最新一条」
- 能指出 ReplacingMergeTree 去重是异步后台 merge 触发的，查询要用 argMax/FINAL 兜底，不能依赖「合并完就干净」
- 能主动问数据规模、写入频率、查询延迟要求，再给结论
- 能联系实践：引擎建表后不能直接 Alter 更换，选错要换表重导

**减分项：**
- 只背「MergeTree 是列式存储」等概念，答不出场景匹配
- 说不清 ReplacingMergeTree 的去重时机、分区限制与 version 语义
- 忽略引擎选型与运维成本（换引擎 = 重建表）的关系
- 把引擎选型和查询性能完全割裂开谈

**解答：**
先明确两张表的查询模式再选型：报表表是「按维度预聚合、数值覆盖累加」，登录表是「按 key 取最新一条」，二者都是典型的写多读少、按 key 收敛的场景，普通 MergeTree 需要在查询时手动 group by 或取 max，代价高。报表表用 SummingMergeTree：以维度列作为 ORDER BY（排序键），数值列在后台 merge 时自动求和，查询仍要写 group by 保证结果正确（合并是异步的，SummingMergeTree 只减少 parts、加速查询，不改变 SQL 语义）。登录表用 ReplacingMergeTree：以 user_id 为排序键前缀，写入时带 version 字段（如登录时间戳），查询用 argMax(value, version) + GROUP BY，或对低并发小表用 FINAL。实践中的坑：ReplacingMergeTree 的替换只发生在 merge 时且只在同一分区内生效，跨分区不替换，所以写入时要保证同一 key 落在同一分区；version 相同或缺失时保留行为不可控；依赖 FINAL 会强制合并语义，大表慎用；引擎建表后无法 Alter 更换，选错只能重建重导。

**延伸考点：** ReplacingMergeTree 里 version 字段没有索引，怎么优化「取最新」的查询性能？为什么说 SummingMergeTree 查出来的结果不保证是最终正确值？

---

### Q2. 分区键怎么选？

**问题：** 一张 10 亿行的事件明细表，业务经常按天过滤（最近 30 天）、偶尔按小时过滤，还会按 user_id 精确查单用户。分区键怎么定？有人建议按 user_id 取模分 16 个分区，你怎么评价？

**期望加分项：**
- 能说出分区的核心目的是分区裁剪（partition pruning），分区粒度要匹配最频繁的时间过滤粒度
- 能指出按 user_id 取模分区的缺陷：每次查询都要扫全部分区，裁剪完全失效，还会放大跨分区聚合成本
- 能给出权衡：默认按天分区 + 排序键带 user_id，兼顾时间范围过滤与 user_id 等值查询
- 能说明分区粒度过细（按小时）导致 parts 过多、merge 压力大、可能触发 Too many parts
- 能分清分区与索引的分工：分区是物理裁剪，排序键/稀疏索引是分区内裁剪

**减分项：**
- 把分区和排序键/索引混为一谈
- 建议按低基数字段或 user_id 取模做分区，却不考虑查询是否命中裁剪
- 不考虑分区数量与写入频率、merge 吞吐的关系
- 只说「按天分区」但答不出何时该按小时、何时该按周

**解答：**
分区键的第一目标是让查询跳过无关数据，所以应选查询里最稳定、最常出现的过滤维度；时间天然有序，是默认选择。建表 `PARTITION BY toYYYYMMDD(event_date)`、`ORDER BY (event_date, user_id)`：按天过滤直接裁剪到目标分区，user_id 等值查询由排序键前缀的稀疏索引解决，不需要靠分区。按 user_id 取模分区的方案是反面教材：查询条件不带取模字段时每个分区都要扫，分区失去意义，且跨分片/跨分区聚合成本翻倍。实践中的坑：分区粒度别细于写入频率——分区数越多，后台 parts 越多，merge 跟不上就报 Too many parts 阻塞写入；按小时分区只适合超高写入频率且查询确实按小时过滤的场景。分区字段要用可裁剪的时间表达式，避免对无索引的字符串字段做隐式转换导致裁剪失效。大分区内没有索引覆盖的查询会全扫该分区，这是最常见的时间维性能杀手。

**延伸考点：** 分区裁剪失效的常见原因有哪些（比如函数套用、隐式类型转换）？按周分区和按月分区各适合什么写入频率？

---

### Q3. 等值查询慢，怎么用排序键/稀疏索引逐层优化？

**问题：** 10 亿行明细表按天分区后，按 user_id 的等值查询仍然要扫大半天数据。你会怎么用排序键、索引逐层优化？改排序键有什么代价？

**期望加分项：**
- 能说出 ClickHouse 主索引是稀疏索引：数据按 ORDER BY 物理有序，索引每 8192 行（granule）记录一行起点
- 能给出量化认知：过滤字段必须是排序键前缀才高效，前缀顺序决定裁剪效果
- 能给出字段排序原则：高频过滤、高基数字段放前，并说明等值查询 vs 范围查询对前缀的要求差异
- 能指出排序键变更 = 全表重写（ALTER TABLE ... MODIFY ORDER BY + materialize 或换表重导），代价极高，需慎重
- 能提到跳数索引/projection 作为补充而非替代

**减分项：**
- 以为主键 = 唯一约束，或以为给 user_id 单独建索引就有用
- 答「加索引」但说不清 ClickHouse 索引是稀疏的、按排序键前缀裁剪
- 不讨论多个过滤字段在 ORDER BY 中的顺序组合
- 不知道改排序键要重写全表，轻描淡写说「改一下就行」

**解答：**
先理解机制：ClickHouse 的 MergeTree 主索引是稀疏的，数据按排序键物理有序，索引只记录每个 granule（默认 8192 行）的起始值，查询先定位 granule 再做块内扫描。因此过滤条件必须是排序键前缀才能高效裁剪。逐层优化：①把高频等值过滤字段前移：`ORDER BY (user_id, event_date)`，user_id 等值 + event_date 范围都能高效命中；②无法放前缀的字段加跳数索引做粗过滤；③固定高频查询用物化视图/projection 预计算。实践中的坑：排序键字段顺序不同，查询性能差异可达数量级，不能拍脑袋要压测；排序键字段不宜过多（每多一个字段，插入排序与索引存储开销都涨）；`ALTER TABLE ... MODIFY ORDER BY` 之后要 `ALTER TABLE ... MATERIALIZE INDEX/ORDER BY` 才会真正重写数据，大表耗时数小时且占磁盘；旧版本只能换表重导。高基数字段（如 user_id）放前面时，时间范围裁剪会变差，两者要实测权衡。

**延伸考点：** ORDER BY 加太多列会带来哪些具体开销？为什么说稀疏索引对范围查询的裁剪不如等值查询直观？

---

### Q4. 低基数字段过滤慢，跳数索引怎么设计？

**问题：** 表按时间排序，但业务经常按 status 字段（几十个枚举值）过滤，全扫很慢。给 status 加跳数索引有用吗？参数怎么定？

**期望加分项：**
- 能说清跳数索引本质是粗粒度剪枝：按 granule 记录块内极值/布隆过滤，跳过不满足条件的块，不是点查索引
- 能针对低基数枚举给方案：status 用 `TYPE set(N)` 或 `bloom_filter(0.01)`，并说明两者差异
- 能说出 GRANULARITY 参数含义与取舍：值越大索引越粗、构建开销越小、裁剪能力越弱
- 能指出跳数索引的代价：写入放大、内存/磁盘占用、构建慢，且新插入数据需后台 merge 后才构建
- 能先用 `EXPLAIN indexes=1` 验证是否命中，再决定是否保留

**减分项：**
- 把跳数索引当普通二级索引理解，以为能直接点查
- 答不出 bloom_filter 误判率与 set 类型的区别
- 忽略跳数索引对写入的拖累
- 不看数据分布，一律推荐加跳数索引

**解答：**
先判断过滤模式：status 基数低（几十个枚举）、查询频繁，且不在排序键前缀里，此时跳数索引能跳过大量不满足条件的 granule，是有效的。但它只做「减负」——把不需要的块剪掉，命中块内仍要扫描。建表时加 `INDEX idx_status status TYPE bloom_filter(0.01) GRANULARITY 1`；对纯枚举型低基数字段，`TYPE set(100)` 记录块内出现过的值更省内存；GRANULARITY 表示每 N 个 granule 建一个索引块，值越大索引越粗。实践中的坑：跳数索引对高基数且分布均匀的字段（如 user_id）几乎无用，因为每个块大概率都包含目标值，剪不掉；索引构建发生在后台 merge，写入后立即查可能不命中，可用 `OPTIMIZE TABLE ... FINAL` 提前构建（大表慎用）；索引本身占内存与磁盘，列很多时叠加明显；`EXPLAIN SELECT ... indexes=1` 能看命中的索引与跳过比例，这是上线前的必要验证。

**延伸考点：** 为什么点查高基数字段时跳数索引几乎失效？GRANULARITY 调大调小分别影响什么？

---

### Q5. 高频小批量写入导致 Too many parts 怎么办？

**问题：** 业务方每次来几条数据就 INSERT 一次，一天上百万次，线上报 Too many parts、写入变慢。你会怎么治理？buffer 表能用吗？

**期望加分项：**
- 能说清 parts 产生机制：每次 INSERT 至少生成一个 part，后台 merge 异步追赶，写入频率过高就爆
- 能给出量化建议：攒批（单批数万行以上、频率降到每秒几次甚至更低）、应用侧批量插入
- 能说出 buffer 表适用场景与坑：进程崩溃丢数据、单线程 flush、不适合核心链路
- 能给出相关参数：parts_to_throw_insert、parts_to_delay_insert、max_partitions_per_insert_block
- 能指出分布式表下实际落盘频率 = 分片数 × 插入频率，评估时要乘分片数

**减分项：**
- 只答「用批量插入」，给不出批量大小/频率的量化依据
- 不知道 Too many parts 的成因与对应参数
- 推荐 buffer 表却不谈丢数据风险
- 不考虑 merge 吞吐与 parts 数监控

**解答：**
机制上，每次 INSERT 生成一个 part，写入频率远高于后台 merge 消化速度时，parts 数量指数级增长，触发 `parts_to_throw_insert`（默认 300）直接拒绝写入。核心是降低落盘频率：应用侧攒批，单批 5 万行以上、频率控制在每秒个位数到每分钟几十次，具体上限取决于机器与 merge 能力，用 `system.parts` 观察 active parts 数量在稳定回落即可。buffer 表可在 ClickHouse 内部做缓冲（`min/max_rows`、`min/max_time` 控制 flush），但它是内存缓冲、进程崩溃可能丢数据，且单线程 flush，吞吐有限，适合低频低价值数据的临时缓冲，不适合核心实时链路——更推荐在应用层或 Kafka + 物化视图链路里攒批。实践中的坑：分布式表写入时每个分片各自落 part，真正的落盘频率是 分片数 × INSERT 频率；并发 INSERT 数（`max_insert_threads`、`max_insert_block_size`）也会放大 part 数；Too many parts 后可临时调大阈值应急，但根治靠降频攒批。

**延伸考点：** Kafka 引擎表为什么天然规避了这个高频小 part 问题？buffer 表在什么场景反而会让问题更严重？

---

### Q6. 重复 key 要「最新一条」，表结构怎么设计？去重可靠吗？

**问题：** MergeTree 表每天有大量重复 key，业务要的是每个 key 的最新一条。你怎么设计表结构与查询？这种去重可靠吗？

**期望加分项：**
- 能给出 ReplacingMergeTree + version 字段的设计，并说明 version 决定保留哪行
- 能说清去重是后台 merge 时发生、异步的，查询不能依赖「已去重」
- 能给出查询兜底：argMax(value, version) + GROUP BY，或小表用 FINAL（说明 FINAL 的开销）
- 能指出只在同一分区内去重，跨分区重复需要保证写入分区归属
- 能联系实践：高频更新下 merge 滞后，短期内查到的可能不是最终状态

**减分项：**
- 以为 INSERT 时就去重
- 说「加 version 字段」但答不出查询端怎么取最新
- 不知道跨分区不去重
- 不讨论去重时效与数据新鲜度窗口的权衡

**解答：**
ReplacingMergeTree 的去重发生在后台 merge：同排序键的行只保留 version 最大的，且只在同一分区内生效。工程上要把「存储层去重」和「查询层去重」分开：存储层用 `ORDER BY (key)` + version 列（如 `toUnixTimestamp(updated_at)`）建表；查询层用 `SELECT key, argMax(value, version) FROM t GROUP BY key`，这是生产上推荐的做法——FINAL 会强制按合并语义重算，小表可接受，大表扫描成本高。写入时保证同一 key 落在同一分区（如 `PARTITION BY toYYYYMMDD(updated_at)`），否则跨分区不替换，这是最容易踩的坑。version 相同或缺失时保留哪行不确定，写入要保证 version 唯一且递增。高频更新场景下 merge 滞后，会存在「查出来不是最新」的短暂窗口，对时效敏感的业务要在查询层自己取最新或改用 Kafka 宽表/流式更新链路，不要信任后台合并的时效。去重正确性最终以查询层的 argMax 为准，存储层只是优化。

**延伸考点：** 业务要求「最多保留 N 天内的最新一条」，分区与 TTL 怎么配合？FINAL 与 GROUP BY+argMax 在什么量级下性能差异明显？

---

### Q7. 全表 group by 越来越慢，怎么用聚合引擎或物化视图预聚合？

**问题：** 10 亿行明细表，运营每天跑全表 group by 出报表，越来越慢。你会怎么用 SummingMergeTree / AggregatingMergeTree 或物化视图做预聚合？

**期望加分项：**
- 能区分 SummingMergeTree（数值求和）与 AggregatingMergeTree（任意聚合，-State/-Merge 配合）的适用场景
- 能说清预聚合后查询仍要 group by 的原因（合并是异步的，语义不变）
- 能给出物化视图增量聚合方案：明细表 + 物化视图写入聚合表，历史数据手动回填
- 能指出聚合粒度必须匹配查询维度，不可下钻到未预聚合的维度（维度爆炸问题）
- 能说出 count distinct / quantile 类指标要用 uniqState/quantileState 态函数

**减分项：**
- 混用两种聚合引擎的语义
- 以为预聚合表查出来就是最终结果，不用再 group by
- 不做维度匹配分析直接上物化视图
- 不知道 -State/-Merge 函数的用法与 AggregateFunction 列类型

**解答：**
先盘点报表查询的维度与指标组合，判断预聚合能覆盖多少查询，再决定方案。SummingMergeTree 只对数值列求和，适合「按维度求和」类报表；AggregatingMergeTree 可承载 count distinct、avg、quantile 等任意聚合，但存储的是聚合中间态：写入端用 `uniqState(uid)`、`sumState(amount)`，查询端用 `uniqMerge()`、`sumMerge()` 合并。链路：明细表 + 物化视图 `CREATE MATERIALIZED VIEW mv TO agg_table AS SELECT dim, uniqState(uid), sumState(amount) FROM detail GROUP BY dim`，明细每次插入自动增量写入聚合表；历史存量必须手动 `INSERT INTO agg_table SELECT ... FROM detail` 回填。实践中的坑：物化视图是插入时触发，不随明细表的 TTL/schema 变更联动，需要单独维护；维度组合多时聚合表膨胀（维度爆炸），要按查询频率只预聚合高频组合；时间窗口边界（跨天归因）与视图粒度不匹配会算错；AggregateFunction 列用普通工具看是乱码，必须用 -Merge 查询。

**延伸考点：** 维度爆炸时你会怎么降维？物化视图与 projection 的预聚合方案各自适用什么场景？

---

### Q8. 实时指标被反复更新，怎么用折叠引擎高效计算？

**问题：** 业务要实时统计「当前在线人数」「账户余额」这类会被反复覆盖更新的指标，同一 key 的明细行频繁变更。ReplacingMergeTree 只能取最新，统计还得全量重算，有没有更高效的做法？

**期望加分项：**
- 能引出 CollapsingMergeTree：写入 sign=1/-1 的行对，merge 时相互抵消，每个 key 只保留净态
- 能说清查询必须带 sum(sign) 过滤（如 `HAVING sum(sign) > 0` 或先对 sign 过滤），否则结果是错的
- 能指出乱序/重复插入导致配对失败的问题，以及 VersionedCollapsingMergeTree 用 version 保证配对正确
- 能说清适用边界：适合求和/计数类可折叠指标，对 count distinct 无能为力
- 能联系实践：更新流程是「先插 sign=-1 的旧值，再插 sign=+1 的新值」

**减分项：**
- 不知道 sign 配对机制与查询过滤要求
- 混淆 CollapsingMergeTree 与 ReplacingMergeTree 的语义
- 忽略乱序写入导致折叠失败的问题
- 不讨论该方案对 count distinct 类指标无效

**解答：**
折叠方案的核心是用 sign 字段把「历史值」和「新值」做成一对，后台 merge 时相互抵消，让每个 key 只保留净态，之后按 key 聚合即可得到实时结果，无需扫描全部历史变更。建表 `ORDER BY (key)` + sign Int8 列；更新流程：`INSERT (key, old_value, -1)` 和 `INSERT (key, new_value, +1)` 成对写入；查询实时值：`SELECT key, sum(value * sign) ... GROUP BY key HAVING sum(sign) > 0`。乱序或重复写入会导致 sign 配对错乱，此时升级为 VersionedCollapsingMergeTree，加 version 字段，保证同一 version 内 sign 配对正确。实践中的坑：查询漏写 sum(sign) 过滤会把已折叠的历史行也算进去，是线上最高频的事故；该方案对 count distinct 类指标无效（抵消不掉重复计数），这类场景要改用 AggregatingMergeTree 或宽表覆盖更新；高并发写同一 key 时配对顺序可能错乱，需要业务侧串行化或接受短暂误差；折叠依赖后台 merge 收敛，merge 未执行前查询结果可能暂时包含未抵消的行，实时查询要容忍这一点。

**延伸考点：** 为什么折叠查询必须用 sum(sign) 而不是简单过滤 sign=1？什么情况下你会放弃折叠改走宽表覆盖更新？

---

### Q9. ALTER DELETE/UPDATE（mutation）的代价有多大？

**问题：** 业务要按条件删除一批历史数据，或批量改某个字段。在 ClickHouse 里执行 ALTER TABLE ... DELETE/UPDATE 会有什么代价？生产上怎么安全执行？

**期望加分项：**
- 能说出 mutation 不是原地修改，而是异步重写受影响分区的 parts
- 能量化代价：磁盘 IO 放大（可能两倍以上）、CPU 高、期间查询变慢、可能触发 Too many parts
- 能给出安全姿势：低峰执行、限制并发与同步（mutations_sync）、查 system.mutations 盯进度、KILL MUTATION 兜底
- 能建议替代方案：按分区删除用 DROP PARTITION，按时间自动清理用 TTL，改字段优先重建表
- 能指出 mutation 不提供事务语义，失败要人工干预

**减分项：**
- 以为 delete 是立即生效的
- 不知道 mutation 是重写 parts 而非标记删除
- 生产环境随意全表 mutation 且不看 system.mutations
- 不讨论 TTL、DROP PARTITION 等更廉价的替代方案

**解答：**
ClickHouse 的 UPDATE/DELETE 本质是异步 mutation：把受影响分区内的 parts 读出来、改写后再写回，保留新旧两个版本，代价与命中数据量成正比。第一步想清楚能不能不做：按分区删用 `ALTER TABLE t DROP PARTITION '...'`（秒级）；按时间自动清理用 TTL（如 `TTL event_date + INTERVAL 30 DAY`）；只有必须按任意条件删/改时才用 `ALTER TABLE ... DELETE WHERE ...` / `UPDATE ... SET ...`，并在业务低峰执行。执行时用 `mutations_sync = 1/2` 同步等待（2 表示等所有副本），或异步执行后查 `system.mutations` 看 state 与 progress，卡住用 `KILL MUTATION`。实践中的坑：mutation 期间目标分区的查询可能变慢（新旧 part 并存）；大表全量 mutation 可能持续数小时并产生大量临时 part，磁盘峰值接近两倍；mutation 排队堆积会挤占 merge 资源，进而 Too many parts 阻塞写入；KILL MUTATION 后部分 part 可能处于中间状态，需要观察并人工清理。生产原则：能不 mutation 就不 mutation，大范围变更优先「分区级重建 + 重新导入」。

**延伸考点：** 为什么说 ClickHouse 不适合做高频点更新？KILL MUTATION 之后表状态如何确认恢复干净？

---

### Q10. 一条慢查询怎么逐步优化？

**问题：** 「统计每个城市近 7 天的订单量」这条查询跑了十几秒。请描述你从定位瓶颈到优化完成的完整步骤。

**期望加分项：**
- 能先定位瓶颈再动手：EXPLAIN / system.query_log 看 read_rows、read_bytes、扫描分区数，判断是扫太多还是聚合内存/网络问题
- 能给出查询侧优化：PREWHERE 把过滤前置（并说明其生效条件与局限）
- 能给出 schema 侧优化：排序键、跳数索引、物化视图/projection
- 能说清 EXPLAIN 输出怎么读（partitions 数、granules、是否命中索引/投影）
- 能说明优化要按数据流验证，改一次测一次

**减分项：**
- 上来就调参数，不先量化瓶颈
- 不知道 PREWHERE 只对部分列过滤生效、且对选择性差的过滤收益有限
- 不了解 projection 的构建时机与失效条件
- 不会读 EXPLAIN / query_log 的关键指标

**解答：**
先量化再优化。用 `EXPLAIN SELECT ...` 或查 `system.query_log` 的 read_rows/read_bytes/partitions，判断瓶颈是「扫描数据量太大」还是「聚合/网络开销大」。若扫描量大：①确认分区裁剪生效（EXPLAIN 里看 partitions 数，若扫了全部分区先查过滤条件写法）；②把低选择性的过滤条件放 PREWHERE（宽表、过滤列与其他查询列分开存储时收益明显，但要求过滤能走索引裁剪或跳数索引，否则白加）；③对无法走排序键前缀的字段加跳数索引；④高频固定查询用物化视图或 projection 预聚合。若聚合慢：优化 group by 基数、调整 max_threads、考虑预聚合表。实践中的坑：PREWHERE 并非总是更快——过滤列本身是主查询列或过滤选择性差时反而多一次 IO；projection 写入有额外开销，ALTER 后要 `MATERIALIZE PROJECTION` 才生效，且带 LIMIT/特定算子时可能不命中投影；网上抄的参数（如调大 max_memory_usage）只是缓解症状，要回到扫描量与合并路径上治理。

**延伸考点：** EXPLAIN 输出里哪个字段能判断查询是否命中投影？为什么 group by 超高基数 key 要特别小心？

---

### Q11. 物化视图的正确用法与典型误用

**问题：** 有同事把物化视图当「加速所有查询的万能工具」往生产上堆，你怎么评估这个做法？物化视图的典型误用有哪些？

**期望加分项：**
- 能说清物化视图本质：插入时触发的 SQL + 一张存储表，只对命中预设粒度的查询生效
- 能指出典型误用：维度不匹配、历史数据没回填、视图与明细 schema 不同步导致写失败
- 能给出正确流程：先盘点查询集 → 定聚合粒度 → 建视图 + 聚合表 → 回填历史 → 结果对比验证
- 能区分物化视图与 projection：前者增量维护、粒度自定义；后者是查询侧自动匹配的冗余存储
- 能提到视图数据异步写入、可能有滞后，以及需要单独配置 TTL

**减分项：**
- 把物化视图当成通用查询加速器
- 不知道视图是插入时触发、历史数据不会自动进入
- 不看查询维度与视图粒度的匹配性
- 说不出视图与 target 表、明细表三者之间的关系

**解答：**
物化视图 = `CREATE MATERIALIZED VIEW mv TO target AS SELECT ... FROM source`：source 每插入一批数据，视图 SQL 对新数据执行一遍并写入 target，本质是「插入时的增量预计算」，不是查询加速器。正确流程：①盘点高频查询的维度与指标组合，选覆盖度最高的几组做视图；②target 用 Summing/AggregatingMergeTree 承接；③视图建好后必须手动 `INSERT INTO target SELECT ...` 回填历史存量；④对比视图结果与明细 group by 结果验证正确性，并看 `system.parts` 确认视图表数据在增长。典型误用：视图粒度（如按天）与业务查询（按小时）不匹配，白建；视图挂多个导致写入放大；视图 SQL 引用的列被删改后视图写失败（明细写入会卡住或丢数据）；视图表不继承明细表 TTL，要单独配。与 projection 的取舍：物化视图维护成本高但粒度可定制、可加过滤；projection 对查询透明、自动匹配，但只适合「同一表多种排序/简单聚合」的场景，粒度不灵活。

**延伸考点：** 如果物化视图的 target 表写失败，明细表本身的 INSERT 会怎样？视图 SQL 里出现 JOIN 为什么通常不建议？

---

### Q12. 分布式表查询慢，网络放大怎么治？

**问题：** 6 节点集群，业务表建成了 Distributed 表，查询越来越慢。排查发现查询把各分片数据拉到发起节点再聚合。怎么优化？分布式表有什么使用原则？

**期望加分项：**
- 能说清 Distributed 表是逻辑路由层，数据实际存本地表；分布式聚合是「本地预聚合 + 汇总」两阶段
- 能给出避免全量拉取的写法：GLOBAL IN / GLOBAL JOIN 让数据先分发到各分片本地执行
- 能说出优先用本地表 + 应用层路由，查询只触达相关分片
- 能指出 sharding key 选型影响数据分布均衡与本地 JOIN 命中率
- 能提到 prefer_localhost_replica 等路由参数与读写分离实践

**减分项：**
- 以为 Distributed 表自己存了数据
- 不知道分布式聚合的两阶段机制
- 在分布式表上做高频大结果集 JOIN 而不考虑网络开销
- 不讨论分片键选错导致的数据倾斜

**解答：**
Distributed 表只是路由视图：INSERT 时按 sharding key 分发到本地表，SELECT 时把 SQL 下发各分片、结果汇总回发起节点。慢的根源通常是网络放大和汇总节点单点压力。优化方向：①能用本地表就用本地表，固定查询直接路由到具体分片（如按 user_id 分片后单用户查询打到对应分片）；②需要全局去重/关联时用 `GLOBAL IN` / `GLOBAL JOIN`，子查询结果先分发到各分片、本地完成关联，网络传输远小于非 GLOBAL 版本的全量拉取；③sharding key 选高基数且业务常关联的字段（如 user_id），让 JOIN 尽量在本地命中；④聚合查询依赖本地预聚合，分片间数据分布均匀时汇总量可控。实践中的坑：非 GLOBAL 的 IN/JOIN 子查询会把右表全量广播到每个分片，跨分片数据量大时是性能灾难；sharding key 分布不均会导致某分片磁盘/CPU 打满（数据倾斜）；Distributed 表写入要关注 async_insert 缓冲语义与 flush 时机；集群扩容后旧数据不会自动重分布，会出现「新分片空、旧分片满」的热点问题。

**延伸考点：** GLOBAL JOIN 与普通 JOIN 在分布式下的执行差异是什么？扩容后数据倾斜你怎么处理？

---

### Q13. ReplicatedMergeTree 对 ZooKeeper/Keeper 的依赖怎么治理？

**问题：** 集群用 ReplicatedMergeTree 做双副本，某天 ZooKeeper 抖动，写入变慢甚至报错。你怎么排查和治理对 ZK 的依赖？

**期望加分项：**
- 能说清 ReplicatedMergeTree 依赖 ZK 做元数据协调（part 日志、副本队列），数据本身存在本地
- 能说出常见故障链路：ZK 慢 → insert 阻塞 → 副本队列积压 → 副本滞后
- 能给出排查命令：system.replicas、system.replication_queue、system.zookeeper
- 能给出治理手段：迁 ClickHouse Keeper（减少一跳网络）、控制表/分区数量减少 ZK 节点、限制并发 insert、设合理超时
- 能提到跨地域部署副本会放大 ZK 交互延迟，要谨慎

**减分项：**
- 以为副本数据是存在 ZK 里的
- 不知道 replication queue 积压的排查入口
- 建议「换掉 ZK」但说不出 ClickHouse Keeper 的迁移成本与步骤
- 忽略 ZK 抖动对写入链路的直接阻塞影响

**解答：**
ReplicatedMergeTree 的每个写操作都要在 ZK 上登记 part 日志、同步副本状态，ZK 的延迟会直接卡在写入链路上，所以「ZK 抖动 → 写入变慢」是必然的。排查三步：`system.replicas` 看 is_session_expired、queue 状态；`system.replication_queue` 看积压与卡住的条目；`system.zookeeper` 看会话与延迟。治理：①优先迁到 ClickHouse Keeper（3 节点起步），它是 ClickHouse 内嵌的独立组件，少一跳网络、会话管理更优；②压减 ZK 节点数：少建表、分区粒度别过细、及时 merge/TTL 清理——每个 part 在 ZK 都对应日志条目，part 越多 ZK 越重；③限制并发 insert、调整 queue 相关超时参数；④紧急时临时切单副本写入再恢复。实践中的坑：ZK 全挂时写入不会立即失败而是阻塞排队，恢复后积压队列可能长时间追平，期间副本数据滞后；随意 KILL 会话会导致副本反复重新同步；Keeper 迁移期间要盯 replication queue 确保不丢数据；核心表不要为了省事去掉副本。

**延伸考点：** ClickHouse Keeper 相比 ZooKeeper 解决了哪些具体痛点？副本队列长期卡住你会怎么处理？

---

### Q14. 双副本集群查询结果不一致，怎么验证和规避？

**问题：** 双副本集群里，同一查询打到不同节点偶尔返回结果不一致（少几行或多几行）。你怀疑是副本滞后，怎么验证？怎么规避？

**期望加分项：**
- 能说清 ReplicatedMergeTree 是最终一致：insert 写主副本即返回，复制异步进行，查询默认不等待
- 能给出验证方法：system.replicas 对比 absolute_delay 与本地 parts 集合，确认是滞后还是数据损坏
- 能给出规避手段：select_sequential_consistency=1（查询等副本追平）、insert_quorum 写多副本
- 能说清这些设置的代价：牺牲可用性/写入吞吐，只能按查询开不能全局开
- 能提到 Distributed 表路由到滞后副本的问题与 max_replica_delay_for_distributed_queries

**减分项：**
- 以为 ClickHouse 默认强一致
- 不知道 insert_quorum 与 select_sequential_consistency 的作用和代价
- 一上来就在全集群开强一致
- 分不清「副本滞后」与「数据本身不一致」两类问题

**解答：**
先定性：ReplicatedMergeTree 是最终一致性，insert 只需写主副本（或 quorum 个副本）即返回，复制是异步的，查询打到滞后副本就会看到旧数据——这是「副本滞后」而非数据损坏，先验证归属再决定手段。验证：查 `system.replicas` 的 absolute_delay、is_leader，对比两个节点的 part 集合；若数据本身也对不上，再看 `system.replication_queue` 是否有执行失败的条目。规避：业务侧对强一致要求的查询设置 `select_sequential_consistency = 1`（查询前等副本追平，副本故障时该查询会失败）；写入侧用 `insert_quorum = 2`（配合 insert_quorum_parallel），保证 insert 返回时至少两个副本已落盘，把「读旧」窗口压到极短。实践中的坑：全局开 select_sequential_consistency 会让每个查询都依赖复制进度，副本故障时查询大面积失败，生产上只对少数强一致查询开；insert_quorum 降低写入吞吐并依赖 ZK/Keeper 协调，副本故障时写入可能一直等 quorum 而超时；「读到旧数据」还有一种来源是 Distributed 表路由到滞后副本，可用 load_balancing 策略或 max_replica_delay_for_distributed_queries 过滤滞后副本。

**延伸考点：** insert_quorum 开启后副本故障，写入表现是怎样的？读一致性设置对查询延迟的影响你如何量化评估？

---

### Q15. 凌晨批量写入报 Too many parts、CPU 打满，是 merge 风暴吗？

**问题：** 每天 0 点批量任务大量写入，凌晨频繁报 Too many parts 且 CPU 打满，白天恢复正常。这是 merge 风暴吗？怎么治理？

**期望加分项：**
- 能说清 parts 生成与 merge 吞吐的平衡机制、merge 风暴的触发条件
- 能给出治理手段：写入错峰、控制单批大小与并发、分区粒度匹配写入节奏、调整 merge 参数
- 能提到 merge 自动调度是按 parts 数量/大小分级的，不要高峰期手动 optimize 全表
- 能给出监控：system.parts 的 active parts 数、system.merges 队列、parts 趋势
- 能说明 Too many parts 时先查 parts 分布与分区设计，再动参数

**减分项：**
- 只会说「加机器」
- 把 parts 数量与分区数混为一谈
- 高峰期手动 OPTIMIZE ... FINAL 加剧风暴
- 不监控 merge 队列与 parts 数量的变化趋势

**解答：**
机制上，每次 INSERT 生成一个 part，后台 merge 按大小分级把多个小 part 合并成大 part；0 点批量写入爆发时 parts 数量快速增长，merge 线程吃满仍追不上，触发 `parts_to_throw_insert` 抛错，CPU 同时被 merge 占满——这就是典型的 merge 风暴。治理前先看 `system.parts` 的 active parts 数量与 `system.merges` 队列，确认是 merge 跟不上而非分区设计问题。治理手段：①写入错峰，把 0 点大任务拆成多个时间窗、降低单批并发；②分区粒度与写入节奏匹配（如按天分区但一次写几十亿行，考虑按小时分区或调大单批大小），避免同一分区瞬间产生大量小 part；③调大 merge 并行：`background_pool_size`、`max_merge_threads`，并确认 `max_bytes_to_merge` 允许更大 part 参与合并；④禁用手动 `OPTIMIZE TABLE ... FINAL`，尤其高峰期。实践中的坑：盲目调大 merge 线程会让 CPU 与磁盘 IO 打架、拖慢查询；小 part 过多会放大每次查询的索引读取开销；风暴结束后 merge 收敛需要时间，parts 数回落前查询性能都受影响；不要在白天高峰做全表 optimize。

**延伸考点：** parts 数量与查询性能的关系曲线怎么理解？为什么「分区下钻到小时」在批量写入场景可能适得其反？

---

### Q16. Kafka 实时链路怎么接 ClickHouse？

**问题：** 要把 Kafka 里的用户行为日志实时落到 ClickHouse，要求延迟秒级、尽量不丢。你会怎么设计链路？Kafka 引擎表有哪些坑？

**期望加分项：**
- 能给出 Kafka 引擎表 + 物化视图落本地表的经典链路，能说清 kafka_format、kafka_num_consumers 等关键设置
- 能说清 Kafka 引擎表是「管道」：消费的数据放临时 part，物化视图再搬进明细表
- 能说清 at-least-once 语义与重复消费的去重兜底（ReplacingMergeTree + version）
- 能提到消费 lag 与失败监控（system.kafka_consumers / system.kafka）
- 能讨论 JSON 解析失败、字段类型不匹配时的处理与背压

**减分项：**
- 以为 Kafka 引擎表存了完整数据
- 不知道解析失败时消息卡在临时表、消费停滞的机制
- 不提重复消费与幂等兜底
- 不做消费 lag 监控

**解答：**
标准链路是三层：Kafka 引擎表做「管道」——`CREATE TABLE kafka_queue(...) ENGINE=Kafka SETTINGS kafka_broker_list=..., kafka_topic_list=..., kafka_format='JSONEachRow'`，再建物化视图把 kafka_queue 的数据搬进真正的明细本地表（MergeTree 或 ReplacingMergeTree）。Kafka 引擎表里的数据是临时的，只做中转，所以「不丢」要依赖物化视图落库成功。关键设置：`kafka_num_consumers` 控制并行消费度（不超过 topic 分区数）、`kafka_thread_per_consumer`；落库天然按 block 攒批，注意 max_insert_block_size 与 part 大小。实践中的坑：消息字段缺失或类型不匹配会导致该 block 消费失败，数据卡在 kafka_queue 临时表里反复重试，消费 lag 持续上涨，所以要盯 `system.kafka_consumers` 的 lag 与错误；at-least-once 语义下进程重启可能重复消费，下游明细表用 ReplacingMergeTree + version 或查询层去重兜底；物化视图 SELECT 里别做重活（heavy JOIN、复杂函数），会拖死消费吞吐；schema 变更要同步改 Kafka 引擎表与明细表，不一致时解析失败堆积。

**延伸考点：** 消费 lag 持续上涨，你先调什么？Kafka 引擎表数据解析失败时消息会丢吗，怎么确认？

---

### Q17. 历史 Parquet/CSV 文件从 S3/HDFS 导入

**问题：** 数仓有一批历史 Parquet/CSV 文件在 S3/HDFS 上，要一次性导入 ClickHouse，并保持分区与压缩合理。你的方案？直接用 INSERT ... SELECT FROM s3()/hdfs() 有什么要注意的？

**期望加分项：**
- 能给出 `INSERT INTO ... SELECT * FROM s3('path', 'Parquet')` / `hdfs()` 的并行导入思路（按文件分片并发）
- 能说出控制块大小与并发避免 part 碎片：max_insert_block_size、min_insert_block_size_rows
- 能提到字段映射、类型推断、时区与 NULL 处理，先导样本验证再全量
- 能给出导入后处理：OPTIMIZE 收敛 parts、分区与文件语义对齐、幂等重导策略
- 能讨论大文件切分与失败重试（断点续导）

**减分项：**
- 单线程导大文件
- 不控制块大小导致 part 碎片
- 不验证数据质量直接全量导入
- 不知道 s3()/hdfs() 表函数与文件格式的字段映射机制

**解答：**
导入的核心是「并行 + 控制 part 生成」。方案：按文件分片并发执行 `INSERT INTO local_table SELECT * FROM s3('s3://bucket/*.parquet', 'Parquet')`，多文件用通配符由引擎并行读取；对超大文件先切分或按前缀分批，避免单任务扫描太久。关键控制点：`max_insert_block_size` 与每批行数决定生成的 part 大小——批太小会碎成几万个小 part，导入完要 `OPTIMIZE TABLE ... FINAL` 收敛；并发数与后台 merge 能力要匹配，避免导入期间 Too many parts。先导 1 个文件验证：Parquet 嵌套结构（struct/array/map）要显式映射到 ClickHouse 的 Nested/Tuple/Map，日期数字格式与时区要确认，NULL 语义要核对；再全量。实践中的坑：CSV 解析失败默认整批跳过或失败取决于设置，要显式处理坏行；S3 文件更新策略（overwrite vs append）决定要不要先 `DROP PARTITION` 再重导（幂等重导）；导入期与实时写入并发会互相抢 merge 资源，需错峰；S3 与本地间的网络带宽要提前评估，大文件建议走对象存储直读而非中转本地。

**延伸考点：** 怎么判断导入后的 part 是否过碎（合理 part 大小是多少量级）？S3 批量导入与 Kafka 实时写入同时落在同一张表时有什么风险？

---

### Q18. 大查询 OOM，怎么从查询与参数两层治理？

**问题：** 一条大 group by 查询把节点内存打满，节点 OOM 重启。重启后同样的查询还会复发。你怎么从查询与参数两层治理？

**期望加分项：**
- 能说清 ClickHouse 内存大头：group by 聚合状态、JOIN 右表、大 ORDER BY/结果集
- 能给出参数：max_memory_usage（单查询）、max_bytes_before_external_group_by/sort（落盘）、max_threads 与内存的联动
- 能给出查询治理：降低 group by 基数、分批查询、预聚合、避免大结果集排序
- 能指出内存上限是 per-query 的，并发叠加会突破节点内存，要用 max_server_memory_usage 兜底
- 能通过 MemoryTracker / system.processes 定位具体查询

**减分项：**
- 只会加 max_memory_usage，答不出外部聚合/落盘参数
- 不分析是哪个查询引起的就乱调参
- 以为 OOM 是 ClickHouse 进程本身的 bug
- 不讨论 per-query 上限与并发查询叠加的关系

**解答：**
ClickHouse 是「查询内存大户」，每个查询的内存峰值由聚合状态、排序缓冲、JOIN 数据决定，单节点内存耗尽前先被 per-query 上限拦住的是单个查询，多个并发查询的峰值叠加才会打爆节点。第一步定位：从 `system.processes` / MemoryTracker 日志找到峰值内存的 query_id，分析其 group by 基数、扫描量与排序需求。治理分三层：①查询层：拆小结果集、按时间/分区分批、用 `max_bytes_before_external_group_by`（建议设为可用内存的一半左右）把聚合状态落盘、避免 group by 超高基数 key 再 ORDER BY 的组合；②参数层：`max_memory_usage` 按节点内存 60-70% 设，`max_threads` 别盲目调大（线程 × 每线程缓冲就是内存），再设 `max_server_memory_usage` 限制节点总内存使用；③架构层：对固定的大聚合用预聚合表/物化视图打散。实践中的坑：外部聚合用临时目录、磁盘 IO 放大，别当默认方案；OOM 重启后 page cache 被清空，所有查询冷读变慢，要提前预热或错峰；KILL 大查询用 `KILL QUERY WHERE query_id=...`，不是杀进程；落盘参数设得超过剩余内存同样会 OOM，要结合机器实际规格。

**延伸考点：** max_bytes_before_external_group_by 与 max_memory_usage 怎么搭配才不冲突？为什么 ORDER BY 大结果集比聚合更容易 OOM？

---

### Q19. 高基数枚举字符串列，列级优化怎么做？

**问题：** 表里有个约 500 万 distinct 值的字符串枚举列，按它 group by 的查询偏慢，而且业务还拿它和维表 JOIN。除了建索引，有没有列级优化手段？

**期望加分项：**
- 能说出 LowCardinality：低基数字符串本地字典编码，显著降低内存与磁盘占用、加速 group by
- 能判断 500 万基数是否适合 LowCardinality，给出基数量级与内存收益的权衡
- 能给出 Dictionary（外置字典）：维表载入内存用 dictGet 取值替代 JOIN
- 能讨论字符串数值化（如 user_id 用 UInt64）的收益与代价
- 能说明 LowCardinality 不适合的场景：高基数、频繁排序/范围过滤

**减分项：**
- 500 万基数还硬上 LowCardinality，不分析字典开销
- 不知道 LowCardinality 的内存收益来自字典编码
- 有 JOIN 需求时只会想加索引，想不到字典化/维表方案
- 不考虑基数动态增长对字典重建的影响

**解答：**
列级优化分两层：存储编码层和查询层。LowCardinality 把字符串映射成本地字典序号存储，group by 在序号上做，内存与速度收益明显——但只对「低基数」划算（通常万级以内、且值稳定），500 万 distinct 的列用 LowCardinality 反而可能更慢：字典查找开销、全局字典维护成本高，且基数增长会触发字典重建。这类高基数字段优先考虑「数值化」：把 user_id/城市码这类业务编码映射为 UInt64/数值列，内存与压缩全面受益。查询层方案：JOIN 场景用 Dictionary——`CREATE DICTIONARY dim_dict (...) PRIMARY KEY id SOURCE(CLICKHOUSE(...)) LIFETIME(3600)`，查询里用 `dictGet('dim_dict', 'name', id)` 取维度属性，替代分布式大 JOIN，维表常驻内存、单次查询无网络往返。实践中的坑：Dictionary 的 LIFETIME 更新策略不当会读到过期维度；字典源表变更要评估 reload 时机；LowCardinality 列做 range 过滤/排序有时退化成慢路径；不要照搬「基数 < 1 万」的教条，要结合内存收益与查询频率实测。

**延伸考点：** LowCardinality 的字典是本地还是全局的，跨节点查询会怎样？dictGet 与 JOIN 在分布式场景各自的开销差异？

---

### Q20. 30 节点集群的应急与根因治理（开放性）

**问题：** 你是 30 节点 ClickHouse 集群负责人。某天告警：某分片磁盘使用率 95%、副本队列积压、部分查询超时。请描述你的完整应急流程与根因治理方案。

**期望加分项：**
- 能给有顺序的应急动作：先止血（限流/停重查询/释放磁盘）再排查，避免二次事故
- 能给出排查清单：磁盘、parts 数、merge 队列、Keeper/ZK 状态、replication queue、query_log 慢查询
- 能给出容量治理：扩容/重分布、TTL 清理、冷热分层（本地盘 vs 对象存储）
- 能给出监控与预案：磁盘水位、副本滞后、查询延迟告警阈值，容量模型（单节点数据量/写入吞吐/内存）
- 能体现复盘沉淀：从事故中沉淀 CheckList、参数基线、恢复演练

**减分项：**
- 没有先后顺序地乱查
- 只报故障现象、不治理根因
- 不知道磁盘满会连锁引发 merge 失败 → 副本队列积压 → 写入阻塞的链路
- 不提监控与容量规划，只做一次性救火

**解答：**
应急讲究顺序：先保证集群「能写能查」，再定位根因，最后做长期治理。磁盘打满会连锁触发 merge 失败 → 副本队列积压 → 写入阻塞，所以第一步是止血而不是追着副本队列查。止血：①停掉非核心大查询与批量任务（按 query_id KILL 或降并发限流）；②释放磁盘：按 TTL 清过期分区、`DROP` 无用旧 partition、清理临时 part；③磁盘腾出后 merge 恢复，副本队列自然消化，盯 `system.replicas`。根因：查 `system.disks`/`system.parts` 确认是数据增长还是 parts 碎片膨胀，查 `system.query_log` 找放大 IO 的慢查询，检查 Keeper/ZK 是否也吃紧。治理：建立容量模型（单节点数据量 × 压缩比 + 副本冗余，水位预留 30%），配置 TTL 与冷热分层（`TTL ... TO DISK 'cold'` 到对象存储）、必要时扩容并配合 resharding 工具重分布数据，加告警（磁盘 80% 预警 / 90% 紧急、副本滞后秒数、P99 查询延迟）。实践中的坑：磁盘 95% 时先删 parts 而非先跑 optimize（optimize 需要临时空间）；删除要留出 merge 余量；副本队列积压大概率是写放大而非网络问题；扩容后数据不会自动均衡，要规划迁移窗口；每个故障都要沉淀成 CheckList 与容量基线，否则同样的坑会换张表再踩一次。

**延伸考点：** 磁盘满了但业务不能停写，你有哪些临时腾挪手段？如何用系统表（system.parts/merges/replicas/query_log）做一次集群健康体检？
