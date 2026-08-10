# 数据 · BigQuery（面试题库）

本文件面向数据工程师，聚焦 BigQuery 的真实工程场景而非概念背诵。考察重点包括：存储与执行架构（Dremel、Colossus、slot 与按需/容量定价）、分区与聚簇设计及成本治理（dry run、扫描字节数、缓存与物化视图）、查询调优（EXPLAIN、JOIN、MERGE、UDF、半结构化数据展开）、数据管道与安全治理（Storage Write API、BigLake 外部表、Airflow 集成、权限与快照），以及跨数仓选型与迁移实践。题目均为线上可复现的场景化问题，重点考察排查链路、量化依据与取舍判断；难度自 Q1 至 Q20 循序渐进，从实践基础过渡到中阶调优与架构级开放性思考题。

### Q1. 月账单显示某报表查询扫描了 10TB 数据，但源表只有 1TB，怎么定位并优化？

**问题：** 月末账单显示某张报表的一条查询扫描了 10TB 数据，可对应的明细表全表只有 1TB，扫描量怎么会大于表大小？怎么定位并优化？

**期望加分项：**
- 能先分清"扫描字节数 > 表大小"的典型成因：多次扫描同一张大表（JOIN 了多份）、SELECT 了多余列、按需计费下重复执行未命中缓存、隐式笛卡尔积等
- 能给出定位 SQL：查 `INFORMATION_SCHEMA.JOBS_BY_PROJECT`，按 `total_bytes_processed` 排序，再按 `user_email` / 查询文本聚合出 Top 查询
- 有量化意识：能把 `total_bytes_processed` 换算成账单上的 TB/元，并区分"扫描量"与"实际计费量"（按需模式按字节计费、缓存命中不计费）
- 能落地方案：去 `SELECT *`、拆分/改写 JOIN、下沉预聚合、用物化视图
- 主动提到排查时要结合 `creation_time` 时间窗口过滤，避免把 180 天历史全扫一遍
- 能联系线上实践：先看是不是 BI 工具反复触发同一条查询

**减分项：**
- 只说"加个 WHERE 就好了"，给不出定位是哪条查询的方法
- 混淆扫描字节与计费金额、混淆按需与容量定价下的计费口径
- 不知道 `INFORMATION_SCHEMA.JOBS_BY_PROJECT` 或 dry run 这类基础排查手段
- 忽略缓存命中、并发触发、BI 工具直连等非 SQL 因素
- 只会背"SELECT * 不好"，说不出具体的查询级证据链

**解答：**
先别急着改 SQL，第一步是定位：BigQuery 按需定价下账单金额正比于 `total_bytes_processed`，用 `INFORMATION_SCHEMA.JOBS_BY_PROJECT` 按扫描量排序找出 Top 作业，再把 `query`、`user_email`、`creation_time` 列出来，就能确认是不是这一条。扫描量大于表大小在工程上极常见：一是"同表被扫多次"——比如一条查询里 `JOIN` 了同一张大表两份（或子查询重复引用），或临时结果没下沉、每层子查询都重扫原始大表；二是"取列过多"——明细表几百列，`SELECT *` 把不用的宽列全带上，按字节计费下成本被放大数倍；三是 BI 工具直连反复触发同一查询且未命中结果缓存；四是 JOIN 键类型不匹配或缺失条件退化成大爆炸（笛卡尔积），shuffle 与扫描同时放大。定位之后按性价比排序处理：先砍列（只 SELECT 用到的列，通常立竿见影）、再拆 JOIN 或把公共子查询物化成中间表/临时表、然后给高频报表加物化视图或把查询下沉到预聚合层。实践中的坑：`INFORMATION_SCHEMA` 本身也是表，查询时要带 `creation_time` 时间窗过滤，别为了排查 10TB 又扫一遍 180 天历史；参数化查询在日志里 `query` 字段各不相同，聚合时建议先对 `query` 归一化（去掉参数值）再统计；另外 BigQuery 的"查询缓存"只在 24 小时内且查询文本完全一致时命中，BI 工具每次带不同时间参数就永远命中不了。

**延伸考点：** 这条查询改到容量（slot）定价模式下，`total_bytes_processed` 还是计费依据吗？为什么？`SELECT *` 在列存 + 按字节计费的模型下，成本放大的机理具体是什么？

---

### Q2. 明细表每天全量重刷，月度成本涨了 10 倍，分区/聚类怎么调整？

**问题：** 某明细表每天凌晨"先删全表、再全量插入"重刷一遍，这个月账单涨了 10 倍。从分区和聚簇角度怎么调整？

**期望加分项：**
- 能一针见血指出根因：全表重删重插 = 每天把全部历史分区扫描 + 重写一遍，成本随时间线性上涨
- 能给出改法：按天分区 + 只重写当天分区（`WRITE_TRUNCATE` 到指定分区或 `DELETE 分区 + INSERT`），让历史数据不动
- 能说明"重刷"语义变了要怎么办：历史数据修正走补刷脚本（按分区定向重跑），而不是全表重来
- 能结合聚类：查询如果常按 `customer_id`/`order_id` 过滤，在分区内加 `CLUSTER BY`，减少单分区内的扫描块数
- 主动考虑边界：分区数量上限（官方对每表分区数有限制）与保留期规划，历史分区定期过期/归档
- 有量化意识：改造后扫描量应降到"当天数据量 + 少量元数据"，可以用 dry run 验证

**减分项：**
- 把"重刷成本高"归因到表结构之外，答不出全表扫描/重写这个根因
- 只会说"分区"两个字，说不出分区后重刷脚本具体怎么写（装饰器、WRITE_TRUNCATE、MERGE）
- 不考虑历史数据修正与补刷场景，一改就丢能力
- 不知道按天分区 10 年后分区数超上限的问题
- 把分区和聚簇混为一谈

**解答：**
先定性：成本涨 10 倍与 SQL 写法直接相关——"先 `DELETE FROM 表` 再全量 `INSERT`"会让每次运行扫描并重写全部历史分区，表越大、跑的月数越多，每次重刷的代价就越高，成本近似随数据总量线性增长，这恰好解释了"10 倍"。正确做法是给表加时间分区（`PARTITION BY DATE(ts)` 或 `_PARTITIONTIME` 伪列），把重刷范围钉死在目标日期分区上：用 `WRITE_TRUNCATE` 写 `表$20260101` 这种分区装饰器，或先 `DELETE WHERE 分区列 = 当天` 再 INSERT，扫描量就从"全表"降到"一天"。聚簇的作用在分区之后：如果业务查询常带 `customer_id`、`order_id` 等高基数列过滤或做 `GROUP BY`，加 `CLUSTER BY` 可以压缩单分区内的扫描块数，进一步降字节。实践中的坑有三个：一是老的"全量重刷"如果被改成只刷当天，必须同步补一个"定向补刷"机制（按日期区间重跑），否则历史修数没处下手；二是分区数有官方上限（历史上每表约 4000 个分区，官方已放开到更高），按天分区要规划保留期（分区过期自动清理或归档到 GCS），否则十年后建表都建不出来；三是 `CLUSTER BY` 列选择要跟实际过滤条件对齐，选错列收益为零，且聚簇列对顺序敏感、以最左列优先。

**延伸考点：** "重刷当天分区"和"全表重刷"相比，除了扫描量，对并发 DML、查询一致性还有哪些影响？如果上游每天推的是全量快照而非增量，分区方案还成立吗？

---

### Q3. 写查询前怎么预估扫描量？线上怎么按查询统计扫描字节数？

**问题：** 开发同学在上生产前想确认一条新查询会扫多少数据，线上想按查询/按人统计扫描量，分别怎么做？

**期望加分项：**
- 能给出 dry run 的三种用法：控制台 Query editor 勾选 Dry run、`bq query --dry_run`、REST `jobs.insert` 带 `dryRun=true`，且说明 dry run 不执行、不收费、返回预估字节数
- 能给出统计 SQL：`INFORMATION_SCHEMA.JOBS_BY_PROJECT`（或 `_BY_ORGANIZATION`）查 `total_bytes_processed`，按 `user_email`/`query` 聚合排序
- 能说清 dry run 的局限：对 UDF 内部、运行时生成的查询估计不准，是"输入扫描"估计而非最终计费的全貌
- 主动提到把 dry run 固化到 CI：PR 里对每段 SQL 跑 dry run，超阈值（如 >100GB）自动拦截
- 有量化意识：给出按 TB 换算与阈值设定，而不是笼统说"看下成本"

**减分项：**
- 不知道 dry run 是什么，或以为 dry run 要收费
- 只会说"去 console 看 query history"，给不出可复用的 SQL
- 混用 `total_bytes_processed`（计费口径）与 `total_bytes_billed`（按需定价下取整到 10MB 的计费值）
- 不考虑 dry run 与真实执行的偏差来源

**解答：**
写查询前估量用 dry run，BigQuery 提供三个入口：控制台 Query editor 里勾选 "Dry run"、CLI 的 `bq query --dry_run 'SELECT ...'`、以及 API 层 `jobs.insert` 带 `dryRun=true`（各语言 SDK 都支持）。dry run 会返回 `totalBytesProcessed` 等估算字段，不真正执行、不产生任何费用，适合在开发阶段就拦住"上来就扫 10TB"的写法。线上统计则靠 `INFORMATION_SCHEMA.JOBS_BY_PROJECT`，一条常用 SQL 长这样：

```sql
SELECT user_email,
       query,
       total_bytes_processed / POW(1024, 4) AS tb_scanned,
       total_slot_ms / 1000.0 AS slot_seconds
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = 'QUERY'
  AND state = 'DONE'
ORDER BY total_bytes_processed DESC
LIMIT 20;
```

实践中的坑：一是注意计费口径——按需模式下账单按 `total_bytes_billed` 计费，它把每次查询按 10MB 向上取整，量级小的一次次查询积少成多；二是 dry run 对"查询内 UDF 的额外开销、JOIN 引发的 shuffle 放大"估计不足，它主要反映输入扫描，所以 CI 拦截阈值要留出余量；三是 `INFORMATION_SCHEMA` 作业历史只保留约 180 天，且查询它本身也要小心别扫全量历史；四是把 dry run 接进 CI 是成本治理里性价比最高的一招——PR 合并前对每条 SQL 跑 dry run，超过阈值（比如 50~100GB）就拒绝合并，很多团队靠这个把线上扫描量直接压下来一个数量级。

**延伸考点：** `total_bytes_billed` 和 `total_bytes_processed` 在按需/容量两种模式下分别怎么用？为什么说"扫描量不等于计费量"？

---

### Q4. 分区和聚簇什么场景各用哪个？两者能共存吗？

**问题：** 建表时分区和聚簇都是"减少扫描"的手段，面试官问：什么情况下选分区、什么情况下选聚簇、能不能同时用、怎么取舍？

**期望加分项：**
- 能说清机制差异：分区按列把存储切成物理独立片段（裁剪整段）；聚簇在同一分区内按列排序、细化到"块"级裁剪
- 能给出选择规则：查询总是带固定范围的时间条件 → 时间分区；按多列组合过滤/`GROUP BY` 高频 → 聚簇；两者可共存（`PARTITION BY date + CLUSTER BY customer_id`）
- 能说出代价：分区有数量上限、每个分区有元数据开销、按分区覆盖写会留下大量碎片小文件；聚簇对频繁 UPDATE 的表维护收益下降
- 能指出分区列必须能映射成时间/整数范围，聚簇列最多 4 列且顺序敏感（前缀匹配）
- 有边界意识：表小于 1GB、过滤条件能命中主键索引般的精确值时，两者收益都可能趋近于零
- 能联系实践：先看线上慢查询的 `WHERE` 分布再定，而不是拍脑袋

**减分项：**
- 把分区和聚簇当成同一种东西，说不出机制差异
- 只会背"时间分区 + 按 XX 聚类"的模板，给不出取舍依据
- 不知道聚簇列的顺序敏感性与 4 列上限
- 不考虑分区上限、小表收益、UPDATE 场景对聚簇的破坏
- 答不出两者共存的典型组合与各自解决什么问题

**解答：**
先讲机制再讲选择。分区（partitioning）按分区列把数据切成物理上独立的存储段，查询 `WHERE` 带分区条件时优化器做"分区裁剪"，整段跳过，收益是"数量级"的；但它只支持一个分区列，且每表有分区数量上限、每分区都有元数据开销，分区粒度太小（比如小时级但表又不大的场景）反而得不偿失。聚簇（clustering）不改变存储切分，而是在同一分区内按 1~4 列排序组织，让过滤和聚合命中更少的"块"，收益是"百分点级到十倍的扫描压缩"；它对顺序敏感（`CLUSTER BY a, b` 时 `WHERE b=...` 单独用享受不到裁剪），且表发生频繁 UPDATE/DELETE 后排序被破坏，聚簇收益会打折扣。选择规则：查询模式里"必有且稳定"的过滤条件是时间 → 时间分区（这是 95% 的 OLAP 场景）；过滤条件是多列组合、基数高但每列都不适合做分区 → 聚簇；两者完全可以共存，典型组合是 `PARTITION BY DATE(ts) + CLUSTER BY customer_id, region`——分区管时间裁剪，聚簇管业务键裁剪，这是大多数明细大表的标配。实践中的坑：分区列想改（比如从 `_PARTITIONTIME` 换到业务时间列）要重建表，成本不小；聚簇列选错（过滤里根本不出现）收益为零，所以建表前应该先统计线上 TOP 查询的 WHERE 分布；表很小或查询本来就命中极少量数据时，分区/聚簇都不必加，纯属增加维护成本。

**延伸考点：** `CLUSTER BY` 生效依赖数据在写入时的排序，如果每天用流式写入追加乱序数据，聚簇收益还在吗？怎么维持？

---

### Q5. 明细表分区越来越多，存储涨、查询慢，怎么自动清理历史分区？

**问题：** 一张按天分区的明细表跑了两年，分区数 700+，存储费越来越高、查询也变慢。怎么让"过期数据"自动清理又不误删合规要保留的数据？

**期望加分项：**
- 能给出分区过期机制：建表时设 `PARTITION_EXPIRATION_MS` / 表选项 `partition_expiration_days`，过期分区自动删除，无需调度任务
- 能给出脚本化清理：`bq rm --table proj.ds.table$20260101` 或 `DROP TABLE ... $partition` / `DELETE` 按分区条件删除
- 能谈冷热分层：历史分区导出到 GCS（Parquet/Avro）后删分区，查询走 BigLake 外部表，兼顾合规与成本
- 能区分"删除分区"与"删除表内数据"的语义差异与风险（误删不可逆、物化视图依赖会失败）
- 有合规意识：清理窗口要与业务保留期对齐，先列清单再动刀
- 主动提到清理后及时做 `OPTIMIZE` 或重建以消除碎片（若适用）

**减分项：**
- 只会说"写个脚本删数据"，不知道 `partition_expiration_days` 这种原生能力
- 分不清"自动过期"与"手动清理"的边界，把过期机制用在不需要保留的表上
- 不考虑合规保留期与误删恢复（7 天内可恢复的窗口）
- 不知道删除被物化视图/视图引用的分区会报错
- 只说删除，不说归档与冷热分层方案

**解答：**
清理历史分区有两条路线，按"数据要不要留"分。若确认过期数据可以丢弃，直接用分区过期机制：建表时指定 `partition_expiration_days`（对应表选项 `PARTITION_EXPIRATION_MS`），BigQuery 会在过期时间后自动删除分区，不需要任何调度任务，这是成本治理里最省心的做法；若表已存在，用 `ALTER TABLE SET OPTIONS(partition_expiration_days=...)` 补上。若部分数据要留档（合规、审计、下游对账），就走冷热分层：把历史分区导出到 GCS（`EXPORT DATA ... TO 'gs://...'` 成 Parquet），确认导出成功且校验行数一致后，按分区定向删除，查询侧用 BigLake 外部表（或 GCS 外部表）兜底历史，热数据留在原生表——这样"存储费"从 BigQuery 热存储降到 GCS 冷存储，"查询费"仍可按需触发。手动删除分区的写法：`bq rm --table proj.ds.table$20260101`，或 SQL 里 `DELETE FROM table WHERE partition_col < '2025-01-01'`（注意 DELETE 会扫描被删分区并产生 DML 费用）。实践中的坑：一是过期删除是"真删"，只有 7 天 time travel 窗口可救，设过期天数前务必与业务方确认保留期；二是被物化视图/视图引用的分区删除会失败，需先处理依赖；三是 `partition_expiration_days` 设太短、下游某天没消费历史数据就白丢了，建议设"业务保留期 + 冗余缓冲"；四是清理不等于瘦身，如果历史数据是写入时留下的小文件碎片，必要时对热分区做 `OPTIMIZE` 合并。

**延伸考点：** 把历史分区导出到 GCS 后，下游原先按天 `SELECT` 历史数据的查询要改吗？外部表查询延迟和扫描量与原表比有什么差异？

---

### Q6. 生产环境经常出现"全表扫描"告警，怎么系统地避免？

**问题：** 团队在 BigQuery 上接到大量"查询扫描了整张表"的告警，除了加 WHERE，还能从哪些层面系统性地避免全表扫描？

**期望加分项：**
- 能分层回答：表结构层（分区 + 聚簇 + 预聚合表）、SQL 层（WHERE 带分区/聚簇列、只取所需列、避免函数包裹分区列）、治理层（BI 工具强制日期过滤、dry run 拦截、权限上的数据访问策略）
- 能指出"函数包裹分区列导致裁剪失效"这个经典坑：`WHERE DATE(ts) = '2026-01-01'` 无法裁剪，应写成 `ts >= '...' AND ts < '...'` 或对分区列直接用日期比较
- 能谈 BigQuery 的特性手段：`WHERE _PARTITIONTIME` / 伪列约束、查询参数化模板强制传时间参数
- 能提治理手段：对 BI 数据集强制 "date range" 默认过滤、对 ad-hoc 查询用 `--maximum_bytes_billed` 限流（超过直接报错，最硬的手段）
- 主动区分"偶发全扫可接受"与"核心报表全扫不可接受"的边界
- 有量化意识：给出 `maximum_bytes_billed` 具体配置方式与收益

**减分项：**
- 只答"加 WHERE"，没有系统性、分层的答案
- 不知道 `--maximum_bytes_billed` 这种硬限流手段
- 不知道函数包裹分区列会破坏分区裁剪
- 完全不提 BI 工具/权限/治理层面的控制
- 说"全表扫描应该完全禁止"，没有工程上的边界判断

**解答：**
全表扫描要分层治理，单靠"提醒开发加 WHERE"一定漏。第一层是表结构：时间维度给分区、业务键给聚簇，让"该扫的才扫"成为默认；高频指标再下沉到预聚合表/物化视图，报表查询只碰小表。第二层是 SQL 写法，最常见的隐性问题是"函数包裹分区列"——`WHERE DATE(ts) = '2026-01-01'` 这种写法优化器无法裁剪，必须改成范围比较 `ts >= '2026-01-01' AND ts < '2026-01-02'`；另外 `SELECT *` 也会把整表所有列都带上，只取所需列能同时压扫描字节。第三层是治理兜底，三个手段按硬度排序：一是在作业层设 `--maximum_bytes_billed`（SQL 里对应 `SET @@max_bytes_billed`），把单查询扫描上限钉死，超了就报错而不是默默烧钱，这是对 ad-hoc 最有效的硬约束；二是对 BI 工具接的数据源强制默认日期过滤（Looker Studio/BQ 的 dashboard 参数化模板），让"没带时间条件的查询"根本发不出来；三是在 CI 里给每条 PR 的 SQL 跑 dry run，扫描超阈值的直接拒绝合并。实践中的坑：全表扫描不是绝对禁止——对 1GB 级的小维表、或者明确的批量全量分析，全扫是合理的，治理对象应是"无意识的、反复发生的全扫"；`max_bytes_billed` 的单位是字节且按需模式才生效，设之前先看表大小；BI 工具直连 BigQuery 时往往不带分区条件（尤其连接器生成的 SQL 有别名包裹），治理层必须覆盖到 BI 层而不是只约束 SQL 开发者。

**延伸考点：** `max_bytes_billed` 在容量（reservation）模式下还有效吗？为什么很多团队反而在容量模式下放松这个限制？

---

### Q7. 一条 JOIN 查询跑了 40 分钟，用 EXPLAIN 怎么定位瓶颈？

**问题：** 生产上一条两张大表 JOIN 的查询跑了 40 分钟，你打开执行计划（EXPLAIN），重点看哪些字段、怎么一步步定位瓶颈？

**期望加分项：**
- 能说清 BigQuery 执行计划是"树形 Stage"结构，每个 Stage 有 input/output、elapsed、waits、parallelInputs、runs、slot_ms 等关键字段
- 能给出定位套路：先找 elapsed 最大的 Stage，再看该 Stage 的 input rows vs output rows（数据放大比）、shuffle 量、waits 是否偏高
- 能区分两类瓶颈：waits 高 = 并发/slot 不足（资源型），compute/IO 高 = 查询本身重（算法型），应对手段完全不同
- 能结合 `INFORMATION_SCHEMA.JOBS` 的 `total_slot_ms`、`timeline` 交叉验证
- 能落到优化动作：JOIN 键类型对齐、提前过滤/预聚合、加盐处理数据倾斜
- 有实践佐证：用具体 job id 在控制台执行计划图 + Timeline 排障的真实经历

**减分项：**
- 只会背"看执行计划"，说不出看哪些字段、怎么定位
- 分不清 waits 与 compute 的语义，把资源瓶颈误判成查询问题（或反过来）
- 不会从 Stage 树里定位"瓶颈 Stage"，在整体耗时上打转
- 不知道 `INFORMATION_SCHEMA.JOBS` 里有 job 级统计可交叉验证
- 优化建议停留在理论（如"加索引"），不落地到 BigQuery 的实际手段

**解答：**
BigQuery 的执行计划不是传统数据库的"树"而是一棵 Stage 树：查询被拆成多个 Stage（scan、join、aggregate、sort、shuffle 等），每个 Stage 展开后有 `input/output rows`、`elapsed`、`waits`（平均/最大等待）、`parallelInputs`、`runs`、`slot_ms`。定位套路分三步：第一步在 Stage 树里找 `elapsed` 最大的那个 Stage，它就是主瓶颈；第二步看它的特征——若 `waits` 显著偏高（比如平均 wait 占 elapsed 大头），说明查询在等资源，多半是并发查询挤占了 slot（按需模式靠天吃饭、容量模式 reservation 已满），这时候优化 SQL 收效甚微，要么错峰、要么买 slot；若 `waits` 低而 compute/input 量大，说明是查询本身重，进入第三步算法分析：对比该 Stage 的 input rows 与 output rows，放大比巨大通常意味着 JOIN 做成了大 shuffle 或存在数据倾斜（某个 join key 集中了大量行，`runs`/`parallelInputs` 分布不均可以看出），再检查 JOIN 键类型是否一致——类型不匹配（INT64 vs STRING）会强制转换且无法高效 hash join。优化动作按收益排序：先"减少输入"（WHERE 提前过滤、先聚合再 JOIN、按分区钉住时间窗），再"修 JOIN"（键类型对齐、小表内联、对倾斜键加盐拆表合并），最后才考虑重写逻辑。实践里一定要把执行计划图和 `INFORMATION_SCHEMA.JOBS_BY_PROJECT` 里的 `total_slot_ms`、`timeline` 对照看：一个 40 分钟的查询如果只消耗了几十 slot 秒，那纯粹是资源排队问题，改 SQL 是白费劲；反之 slot 消耗巨大，则要优化查询本身。

**延伸考点：** 同一查询在"按需模式"和"容量模式"下执行计划里 waits 的表现有什么不同？怎么用这个差异判断"该买 slot 还是该改 SQL"？

---

### Q8. 大表 JOIN 小表、两张大表 JOIN，分别怎么优化？数据倾斜怎么办？

**问题：** 生产查询里既有"亿级事实表 JOIN 千行维表"，也有"两张十亿级大表 JOIN"。这两类场景在 BigQuery 里的优化思路分别是什么？遇到数据倾斜又怎么办？

**期望加分项：**
- 能指出 BigQuery 对"小表 JOIN 大表"自动做 broadcast（小表广播到各 worker），小表侧尽量先过滤/预聚合，让 broadcast 生效
- 能指出两张大表 JOIN 的关键：JOIN 键类型必须一致（类型不匹配会强制转换、无法走高效路径）、先 WHERE 裁剪再 JOIN、必要时先聚合再 JOIN
- 能讲数据倾斜的识别：执行计划里某 Stage input/output rows 分布不均、单 run 耗时异常；倾斜源常是 NULL、默认值、热点业务键
- 能给出加盐（salt）方案：对热点 key 加随机后缀拆开再合并，并说明加盐后结果的归并写法
- 能说清"能用聚合替代的 JOIN 就聚合"：先按粒度汇总再关联维度，数据量直接掉几个量级
- 有边界意识：NULL 键的语义要单独处理（过滤还是保留），别把倾斜当成查询 bug

**减分项：**
- 两类场景用同一套话术回答，说不清 broadcast 的适用边界
- 不知道 JOIN 键类型不一致对执行计划的影响
- 面对倾斜只说"加盐"，说不出怎么识别倾斜、加盐后怎么归并
- 不先做裁剪/聚合就谈优化
- 把 BigQuery 当传统数据库，提"加索引/加 hint 走嵌套循环"这类不适用的方案

**解答：**
先分场景。小表 JOIN 大表：BigQuery 优化器会把足够小的一侧自动广播（broadcast join），也就是把维表复制到每个 worker 的内存里做 hash join，没有网络 shuffle；要让它生效，注意三件事——把维表侧的过滤和聚合前置（先缩小再 JOIN，能广播的表越小越好）、别在维表列上做函数包裹导致优化器拿不到小表的真实大小、必要时显式用子查询内联（`JOIN (SELECT ... FROM dim WHERE ...) d`）帮优化器缩表。两张大表 JOIN：没有广播的余地，拼的是"减少参与 shuffle 的数据量"——第一，JOIN 键类型必须一致，INT64 JOIN STRING 会触发隐式转换、破坏高效的 hash join 路径（这是线上最常见的隐形杀手）；第二，JOIN 前把两边都裁剪到最小（时间分区、状态过滤、提前聚合）；第三，如果 JOIN 后只是算汇总，干脆改成"先各自 GROUP BY 再 JOIN"，行数往往从十亿级掉到万级，成本差两个量级。数据倾斜：先识别——执行计划里某个 Stage 的 `runs` 不均衡、单 run 的 rows/耗时是平均值的几十倍，多半是热点 key（NULL、空串、默认值如 '0'、头部商户 id）；处理三板斧：先决定 NULL/默认值的语义（该剔除剔除、该单列保留单列），对热点 key 加盐——把热点行拆成 N 份（`CONCAT(key, '_', MOD(x, N))`），JOIN 后再把结果 UNNEST 归并回去；更工程化的做法是"热点行单独处理"：先统计出 top 热点 key，把它们从主 JOIN 里摘出来单独 JOIN，最后 UNION ALL，避免为了少数行拖慢全表。实践中的坑：加盐后聚合结果要记得按业务键去重归并，否则计数会翻 N 倍；BigQuery 没有传统数据库的 join hint 体系，别试图用 hint 解决一切，优化方向永远是"先减数据量，再谈 join 方式"。

**延伸考点：** BigQuery 的 broadcast join 对小表的"大小"判断依据是什么（行数还是字节数）？为什么"看起来很小"的维表有时却走了大 shuffle？

---

### Q9. 上游每天推全量快照，明细表要做增量 upsert，MERGE 怎么用？有什么坑？

**问题：** 上游每天推送全量快照（几亿行），业务表需要按业务键做增量 upsert（更新变化行、插入新行）。用 MERGE 实现，成本和正确性上有哪些坑？有没有替代方案？

**期望加分项：**
- 能写出 MERGE 主干：`MERGE target USING source ON t.key = s.key WHEN MATCHED THEN UPDATE WHEN NOT MATCHED THEN INSERT`
- 能说清 MERGE 的成本结构：要扫描 target + source 全量（做 join），全量快照场景下 MERGE 未必便宜，甚至贵过"每天写新分区 + 读时取最新"
- 能指出正确性坑：同一 target 行被 source 里多条命中时报错、并发对同一行 UPDATE/DELETE 冲突、不能改分区键列
- 能对比替代方案：快照分区 + `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY dt DESC) = 1` 读时去重，或"增量 + 全量分区"混合
- 有边界意识：更新频率 vs 查询频率决定选型——高更新低查询选 MERGE，低更新高查询选快照分区
- 能说 MERGE 的并发限制：对同一表的并发 DML 会因锁冲突重试/失败，需要控制管道并发

**减分项：**
- 只背 MERGE 语法，说不出成本量级与全量快照场景下的性价比
- 不知道 source 重复 key 会报错、不知道分区键不可 UPDATE
- 答不出"读时去重"这个更便宜的同义替代
- 不考虑并发 DML 冲突与失败重试
- 把 MERGE 当银弹，任何增量场景都推 MERGE

**解答：**
先明确：MERGE 是 `target JOIN source` 的语义，BigQuery 实现上要扫描 target 全量 + source 全量做匹配，所以"全量快照 upsert"场景下 MERGE 的成本 ≈ 两张全量表各扫一遍，并不便宜——很多团队用完后发现账单不降反升，就是因为"每天全量 MERGE 10 亿行"本身就很贵。MERGE 的标准写法是 `MERGE ds.target t USING ds.source s ON t.biz_key = s.biz_key WHEN MATCHED THEN UPDATE SET ... WHEN NOT MATCHED THEN INSERT ROW`，几个正确性坑必须知道：一是 source 里同一 key 出现多条会直接报错（不是取最后一条），上游快照必须先去重；二是并发 DML 冲突——同一时间对同一表的多条 UPDATE/DELETE/MERGE 会因行级锁冲突失败，调度要控制管道并发；三是不能 UPDATE 分区列，改分区键会报错。工程上更常见的取舍是"用快照分区替代 MERGE"：每天把全量快照写进独立分区（`dt` 分区），查询用 `QUALIFY ROW_NUMBER() OVER (PARTITION BY biz_key ORDER BY dt DESC) = 1` 取最新行，或建一个"latest"视图——写入成本就是写一遍快照、没有 update 写放大，读取时扫描量大一点。选型标准看读写比例：更新频繁且查询要"当前态"（如订单状态表）→ MERGE 值得；更新低频、查询高频且能容忍扫描历史分区 → 快照分区 + 读时去重更稳更便宜。实践中的坑：MERGE 的 target 表建议同时按时间分区并配合分区裁剪（只 MERGE 近期分区），能把成本压一个量级；快照分区方案要处理好"某 key 在某天没出现"的语义（补全 vs 忽略），否则 `ROW_NUMBER` 会取到过期快照。

**延伸考点：** MERGE 里 `WHEN NOT MATCHED BY SOURCE THEN DELETE` 是做什么的？在"全量快照对齐"场景它怎么帮你把 target 里已消失的行清掉，代价是什么？

---

### Q10. 清洗逻辑想封装成 UDF，JS UDF 和 SQL UDF 性能差多少？什么时候不该用 UDF？

**问题：** 开发想把一段复杂的字符串/JSON 清洗逻辑封装成 UDF 复用到多条管道，问：JS UDF 和 SQL UDF 的取舍？哪些场景根本不该用 UDF？

**期望加分项：**
- 能说清性能机理：SQL UDF 会被优化器内联/下推、可并行向量化执行；JS UDF 跑在 V8 解释层、单行处理、无法参与过滤下推，性能可能差一个数量级以上
- 能说清取舍：能用内置函数（`JSON_EXTRACT`、`REGEXP_*`、`STRING` 系列）表达的就不写 UDF；SQL UDF 能表达的优先 SQL UDF；JS UDF 只留给 SQL 表达不了又必须复用的逻辑
- 能指出不该用 UDF 的场景：在 UDF 里做 GROUP BY/聚合（不支持且违背优化器职责）、对超大表逐行调用 JS、本可用内置正则函数解决
- 能提工程约束：JS UDF 有超时/内存限制、临时 UDF 不能建视图、UDF 版本化与测试成本
- 有边界意识：优先"改数据模型/清洗在写入侧完成"，而不是查询时到处 UDF

**减分项：**
- 不知道 JS UDF 与 SQL UDF 的性能差距及原因
- 一律"封装成 UDF"，没有"内置函数优先"的默认判断
- 不知道 UDF 不能做聚合、不知道临时 UDF 的持久化限制
- 把查询时 UDF 当万能解，不考虑写入侧清洗
- 说不出 JS UDF 的内存/超时限制

**解答：**
默认顺序是：内置函数 > SQL UDF > JS UDF。原因在 BigQuery 的执行模型：SQL UDF（函数体是标准 SQL）会被优化器识别、内联展开、参与列裁剪与过滤下推，可以像普通表达式一样向量化并行执行；而 JS UDF 是逐行把值序列化进 V8 沙箱跑 JavaScript，单行处理、无法下推、无法并行向量化，同一逻辑用 JS 写的执行开销比 SQL 版高一个数量级很常见——所以能用 SQL 表达的逻辑（字符串处理、正则、条件分支、大部分 JSON 解析）一律用 SQL UDF 或干脆用内置函数，`JSON_EXTRACT`、`REGEXP_EXTRACT`、`REGEXP_REPLACE`、`SPLIT` 这些内置函数覆盖了 90% 的清洗需求，连 UDF 都不用写。JS UDF 的正确用途是"SQL 表达不了且必须复用"的复杂逻辑，比如自定义加密算法、依赖第三方 JS 库的解析；同时要知道它的硬约束：JS UDF 有执行超时和内存上限，超大规模表上逐行跑 JS 容易直接超时失败。不该用 UDF 的场景：一是在 UDF 里试图做聚合/排序/JOIN——BigQuery 的标量 UDF 是逐行语义，做不了这些，硬写只会得到一个巨慢的实现；二是对亿级明细表做"查询时清洗"——正确做法是在写入侧（load 前或 ETL 里）把数据清洗好，查询时零 UDF；三是临时 UDF 挂在 `WITH` 里只能本次查询用，要跨管道复用必须建持久化 UDF，持久化 UDF 有版本管理和权限问题，误改一个版本会影响所有引用它的查询，线上改 UDF 要当"发版"处理而不是改代码。

**延伸考点：** 为什么说"在 UDF 里做字符串解析"通常不如在加载阶段就把 JSON 展开成 STRUCT/列？这两条路线的查询性能差异根源在哪？

---

### Q11. 日志以 JSON 存进 BigQuery，怎么高效展开分析？嵌套 STRUCT/ARRAY 的坑有哪些？

**问题：** 埋点日志是嵌套 JSON（一层对象 + 数组），已以 `STRING` 列存进 BigQuery。现在要按嵌套字段过滤、按数组元素聚合，怎么展开？如果当初存成 STRUCT/ARRAY 呢？

**期望加分项：**
- 能明确"查询时用 JSON_EXTRACT/JSON_QUERY 解析 STRING 列的代价"：每行运行时解析、无法利用列存，应在加载时就指定 schema 展开成 STRUCT/ARRAY
- 能给出 STRUCT 展开：`SELECT t.a.b, ...` 点路径取值；数组展开用 `UNNEST(arr)`（隐式 CROSS JOIN），多数组并列要 `LEFT JOIN UNNEST`
- 能指出 UNNEST 的坑：空数组会被过滤掉行（要用 `LEFT JOIN UNNEST` 保行）、多列 UNNEST 交叉展开导致行数爆炸、嵌套层级过深（官方建议 <=15 层）影响性能
- 能谈加载方式：load job 的 `autodetect` + JSON 格式，或显式 schema；schema 变更要 `ALLOW_FIELD_ADDITION`
- 有性能意识：先过滤再 UNNEST、避免 `UNNEST` 后再对大结果聚合；重复查询同一嵌套路径考虑展开成列存到中间表

**减分项：**
- 不知道 `JSON_EXTRACT`（标量/字符串）与 `JSON_QUERY`（对象/数组）的区别，或不知道 `UNNEST` 语义
- 不知道空数组会让行消失的坑
- 一味"查询时解析"，没有"加载时结构化"的结构化思维
- 不了解嵌套层级限制与 schema 变更策略
- 不知道点路径访问 STRUCT 的语法（`record.field`、`record.field[0]`）

**解答：**
原则一句话：能加载时结构化就别查询时解析。如果把日志存成 `STRING` 列，查询里用 `JSON_EXTRACT(json, '$.a.b')` 是逐行运行时解析，既不能利用列存裁剪也绕不开全量扫描，量大之后代价很高；正确做法是 load 时让 BigQuery 按 schema（显式声明或 `autodetect`）把 JSON 解析成 `STRUCT`/`ARRAY` 列存储，查询直接走原生类型。展开语法分两类：STRUCT 用点路径 `SELECT payload.user.id, payload.action`；ARRAY 用 `UNNEST`，`SELECT e.* FROM t, UNNEST(t.items) e` 是隐式 CROSS JOIN，语义是"每行 × 数组每个元素"展开。三个高频坑：一是空数组——`CROSS JOIN UNNEST` 会丢掉原行，要保行必须写 `LEFT JOIN UNNEST(t.items) e ON TRUE`；二是多个数组同时展开（`FROM t, UNNEST(a), UNNEST(b)`）会产生笛卡尔积式的行爆炸，聚合结果翻倍还难排查，通常应先分别聚合再 JOIN；三是嵌套深度——官方建议数据嵌套不要超过约 15 层，太深会显著拖慢加载与查询，宁可拍平。性能细节：先过滤再展开（`WHERE` 带分区/顶层字段条件，把行数降下来再 UNNEST）；如果某个嵌套路径是核心报表的固定查询，把展开结果物化成一个拍平的表/物化视图，而不是每次都 UNNEST 大表；加载时 `autodetect` 对复杂嵌套 schema 可能猜错类型（数组被识别成 STRING、数字精度被降），重要表要显式 schema 并把 JSON 文件里的空数组/缺字段处理规则（`ignore_unknown_values`、`allow_quoted_newlines` 等）想清楚。

**延伸考点：** `UNNEST` 在 BigQuery 里是"运算符"还是"表函数"？`FROM t, UNNEST(t.x)` 与 `FROM t LEFT JOIN UNNEST(t.x) ON TRUE` 的行数语义差异为什么重要？

---

### Q12. 每次报表查询都要扫 2TB 明细，怎么用物化视图优化？它的刷新机制和局限性是什么？

**问题：** 一张 2TB 的明细表，每天有几十个"按天/按省份聚合"的报表查询反复扫它，怎么优化？团队想用物化视图，问它的刷新机制、命中条件和局限性。

**期望加分项：**
- 能说清物化视图的定位：后台自动增量维护的"预聚合结果"，查询命中时不再扫明细基表
- 能说清刷新机制：BigQuery 后台自动刷新（不完全实时、有延迟），基表变更后增量更新，不需要用户写调度；也支持手动刷新
- 能说清命中条件与局限：物化视图覆盖的是"可被查询重写匹配"的聚合（单表 + 确定性聚合函数为主）；JOIN、非确定性函数（`CURRENT_TIMESTAMP`）、部分复杂表达式不支持
- 能对比三个层次：结果缓存（24h 内完全同文本才命中）< BI Engine（内存缓存，按需模式下对重复子查询自动加速）< 物化视图（结果预计算，可被任意匹配查询复用）
- 有成本意识：物化视图有存储成本与刷新资源成本，粒度过细/基表更新过频要算账
- 能给出替代：聚合口径复杂的，直接建"预聚合表"由管道维护，比物化视图更可控

**减分项：**
- 把物化视图和普通视图混为一谈，或不知道"自动增量刷新"这一点
- 不知道物化视图不支持多表 JOIN、非确定性函数
- 不区分物化视图、结果缓存、BI Engine 三个东西
- 不考虑存储/刷新成本，无脑建一堆物化视图
- 不知道"查询重写是否命中"取决于优化器，答不出验证手段（看查询是否还扫基表）

**解答：**
先分清三个"缓存"：结果缓存命中要求 24 小时内、查询文本完全一致且不含非确定性函数，只对"完全重复"的查询生效；BI Engine 在按需模式下对重复子查询做内存加速，不改变查询语义；物化视图是"预计算结果 + 后台增量维护"，最妙的是它不需要查询文本一致——任何一条聚合查询只要能被优化器重写命中物化视图，就直接读小结果而非扫 2TB 基表。物化视图的刷新是 BigQuery 后台自动做的：基表有新数据（load/insert 等）后自动增量刷新，对用户是黑盒，一般有分钟级延迟，也支持 `REFRESH MATERIALIZED VIEW` 手动刷。局限必须讲清：单表为主（不支持多表 JOIN）、聚合函数限定在确定性函数（`SUM`/`COUNT`/`AVG`/`MIN`/`MAX`、`COUNT(DISTINCT)` 部分支持）、不支持非确定性表达式，且"查询能不能命中"由优化器判断，写出来的查询语法上合法也可能完全绕开物化视图。落地建议：对"固定口径 + 高频 + 大表"的指标建物化视图（典型如按天/省份/渠道聚合）；口径复杂或需要跨表 JOIN 的，直接维护一张预聚合表更可控；粒度的选择要算账——物化视图也占存储、刷新也消耗资源，视图粒度越细（如按天×用户×商品）越接近原始表，收益越小；最后验证命中：在查询前看执行计划/job 统计，确认 `total_bytes_processed` 明显小于基表扫描量，否则说明没命中，检查视图定义与查询的匹配度。

**延伸考点：** 物化视图的"自动刷新"在基表频繁流式写入时会怎样表现？对"秒级新鲜度"的查询需求，物化视图还是正确方案吗？

---

### Q13. 相同报表反复执行，有时秒回有时很慢？BigQuery 结果缓存的命中条件是什么？

**问题：** 同一张报表的查询在控制台反复跑，有时秒回、有时还是扫全表。BigQuery 的结果缓存（result cache）命中条件是什么？为什么线上经常"没缓存可命中"？

**期望加分项：**
- 能列出命中条件：24 小时内、查询文本逐字符一致、不含非确定性函数（`CURRENT_TIMESTAMP`、`RAND` 等）、不引用外部表/临时表/带副作用语句、表数据未变化
- 能解释"线上难命中"的常见原因：BI 工具每次生成带不同时间参数的 SQL、查询里带 `CURRENT_DATE`、参数化/宏展开文本不同
- 能说清缓存命中不计费、也不消耗 slot，但结果可能过期——对"报表看到旧数据"要能解释
- 能区分结果缓存与 BI Engine、物化视图的差异（重复文本命中 vs 语义命中 vs 预计算）
- 有测试意识：对比性能时用 `--nouse_cache`（或 `useQueryCache=false`）排除缓存干扰，保证公平
- 能说清 cache 与"表结构变化/DDL"的关系：表被重写后缓存失效

**减分项：**
- 只知道"有缓存"三个字，列不出命中条件
- 不知道非确定性函数会让缓存失效
- 把结果缓存和物化视图/BI Engine 混为一谈
- 不知道表更新/DDL 会失效缓存，无法解释"报表查到了旧数据"
- 做性能对比时不排除缓存干扰，测出假数据

**解答：**
BigQuery 的结果缓存规则可以背下来四条：一、缓存窗口 24 小时，超过即失效；二、查询文本必须逐字符一致，任何空格/大小写/注释差异都算不同查询；三、查询不能含非确定性元素——`CURRENT_TIMESTAMP()`、`CURRENT_DATE`、`RAND()`、`UUID()`、`SESSION_USER()` 这类值每次运行都变，带它们必不命中；四、依赖的表在查询执行后不能被修改（任何 DML/load/DDL 都使相关缓存失效），且外部表（GCS/Drive/BigLake）、临时表、视图层叠情况不参与结果缓存。线上"每次都很慢"的根子大多在两条：BI 工具把参数模板展开成具体日期写进 SQL（`WHERE dt BETWEEN '2026-08-01' AND '2026-08-07'`），明天再跑文本就变了；或者 SQL 里写了 `WHERE dt = CURRENT_DATE` 这种"每次都变"的条件。命中缓存的查询不计费也不占 slot，但代价是结果可能过期——表刚被 ETL 更新，同一文本查询仍可能返回缓存旧值，业务上要能解释这个语义。和另外两个机制分清楚：结果缓存是"完全重复的文本"才命中；BI Engine 是"语义相近/子查询重复"也能加速（按需模式下自动启用）；物化视图是"预计算结果被任意匹配查询复用"。工程习惯：性能对比/压测前必须用 `bq query --nouse_cache` 或 job 配置 `useQueryCache=false` 关掉缓存，否则测出的"1 秒"是缓存命中，不是真实性能；想验证某查询是否命中缓存，看 job 统计里 `cacheHit` 字段即可。

**延伸考点：** 为什么说"带 `CURRENT_TIMESTAMP` 的查询永远不会命中缓存"？如果要让"今天的数据"还能走缓存，工程上怎么设计（比如物化日期分区 + 参数化）？

---

### Q14. 团队上生产数仓，按需定价和容量（slot）定价怎么选？slot 耗尽时查询表现如何？

**问题：** 数据团队要正式上生产，几十人每天跑数百个查询，采购同学问：按需定价和容量定价怎么选？如果选了容量模式，slot 被占满时查询会怎么样，怎么观察和治理？

**期望加分项：**
- 能说清两种模式的本质：按需按扫描字节计费（$6.25/TB 量级）、无并发保障、适合波动大的探索型负载；容量按固定 slot 计费、有并发保障与成本上限、适合稳定批量负载
- 能给出选型判断：先统计现状（`INFORMATION_SCHEMA.JOBS` 聚合每日 slot 消耗与 TB 扫描），"固定开销 vs 按量开销"盈亏平衡点之外还看波动性、峰值、SLA
- 能说 slot 耗尽的症状：查询排队（queued）、执行计划里 waits 高、整体吞吐下降而非单条报错
- 能说观察手段：`INFORMATION_SCHEMA.JOBS_BY_PROJECT` 的 `total_slot_ms`、admin console 的 reservation 用量图、`Jobs` 页的排队状态
- 能谈治理：reservation 里设 `max slots per job`、把长尾大查询错峰、按团队/项目划分 reservation 配额
- 有架构认知：Dremel 树形执行 + Colossus 共享存储 + slot 作为调度单元，能一句话说清"slot 是什么"

**减分项：**
- 只会背价格，说不出两种模式在"并发保障、波动性、治理手段"上的实质差异
- 不知道 slot 耗尽的表现是排队/等待，而非查询直接失败
- 不会用 `INFORMATION_SCHEMA.JOBS` 做现状量化（每日 slot 小时、扫描 TB）
- 没有"给大查询限 slot、按团队分 reservation"的治理手段
- 选型只凭直觉，不看数据

**解答：**
先建立两个锚点：按需（on-demand）定价按扫描字节计费，量=钱，没有并发上限承诺，适合探索分析、负载波动大的团队，缺点是"跑得越多花得越多、高峰期互相抢资源没说法"；容量（capacity）定价是你买固定数量的 slot，费=固定月费，slot 是 BigQuery 的调度资源单位（Dremel 把查询拆成树形执行计划，各节点并行处理分片，slot 就是"一个 worker 单元"），买多少用多少，成本可控、查询有排队保障，适合批量报表、SLA 明确的团队。选型别拍脑袋，先量化现状：用 `INFORMATION_SCHEMA.JOBS_BY_PROJECT` 统计近 30 天每日 `total_slot_ms`（换算成平均/峰值 slot 需求）和每日扫描 TB，对比"买 N 个 slot 的固定费用"与"按需费用的 P50/P90"，再叠加两个判断——负载是否有明显波峰（容量模式里尖峰只能排队或加购）、SLA 与预算上限哪个更硬。slot 耗尽的表现要讲准：不是查询报错，而是排队（job 状态 queued）或执行变慢（执行计划里 waits 占比飙升），整体吞吐塌方但单条看似正常。观察手段：reservation 管理页看 slot 使用曲线、`INFORMATION_SCHEMA.JOBS` 里看 `total_slot_ms` 与排队时长。治理三板斧：给大查询设 `max slots per job` 防止单条吃光全部 slot、把批量管道与 ad-hoc 划到不同 reservation（互相隔离）、对重查询错峰调度。最后记住按需和容量可以共存（部分 workload 走 reservation、部分走按需），迁移也支持灰度切换，别一刀切。

**延伸考点：** 一个"每天固定 8 点跑 50 个批量查询"的团队，为什么可能"买了 1000 slot 还是排队"？`max slots per job`、reservation 拆分、作业优先级（priority）各自解决什么问题？

---

### Q15. 实时链路：埋点每秒上千条写 BigQuery，选哪种写入方式？Storage Write API 的取舍是什么？

**问题：** 埋点数据每秒上千条要实时进 BigQuery，供下游分钟级看板使用。写入方案怎么选？为什么 Google 官方建议新项目用 Storage Write API 而不是老的 `tabledata.insertAll` 流式接口？

**期望加分项：**
- 能说清 `tabledata.insertAll`（legacy streaming）的历史地位与现状：已被官方标记弃用、新项目不再支持，维护成本与功能天花板低
- 能说清 Storage Write API 的三个核心优势：gRPC 二进制协议高吞吐（可达百万行/秒级）、批/流统一（同一套 API 支持批量 load 与流式写入）、支持 exactly-once 语义（应用侧提供去重键）
- 能谈成本与延迟：流式写入按"流式插入字节"计费，吞吐越高摊薄成本；写后读取有 stream buffer 时效（自有写入立即可读，其他查询最终一致）
- 能谈工程配套：用 Dataflow/Apache Beam 或自定义采集服务接入、默认分区按时间流式写入、写失败重试与去重键设计（幂等重试防重复）
- 有取舍意识：不是所有"实时需求"都该上流式——分钟级延迟可用小批量 load 代替，成本低一个量级
- 能提到批量 load 与流式混用的切换策略（写小文件定时 load vs 流式直写）

**减分项：**
- 不知道 `insertAll` 已废弃，或不知道 Storage Write API 的存在
- 只会说"用流式"，讲不出 exactly-once、去重键、成本模型这些工程细节
- 不分"最终一致"与"强一致"：说不清写后立即查的语义
- 不考虑重试导致的重复数据问题
- 所有实时需求一刀切用流式，没有批流取舍判断

**解答：**
先定性：2025 年后 Google 已宣布 `tabledata.insertAll`（老流式接口）弃用、新项目禁止启用，所以新链路的标准答案是 Storage Write API（gRPC，即 `google.cloud.bigquery.storage` 的 `BigQueryWrite` 客户端）。Storage Write API 的核心优势：一是吞吐高，单连接可达每秒数万行、水平扩展后百万行/秒量级，远超 JSON 逐行 POST 的老接口；二是批流统一，批量写入（默认流）与流式写入（默认流/提交流）共用一套协议，管道可以从"攒批 load"平滑演进到"实时写入"；三是 exactly-once，应用侧在写请求里带 `request_id`（去重键），服务端对相同去重键的重复提交只生效一次，配合"失败重试 + 同去重键重放"即可做到不重不丢。成本与语义要讲清：流式写入按"流式插入字节"计费（与查询计费独立），高吞吐场景成本可控；写入后数据先进 stream buffer，自有写入立即可见（strong consistency 对 writer 自身），但其他查询/任务读到的可能是最终一致的视图，做实时看板要在管道里容忍这个窗口。落地注意点：按时间分区流式写入时，分区由数据里的时间字段决定，老数据回填要小心写进过期分区；重试幂等必须配去重键，否则网络重试就是重复数据；另外"是不是真需要流式"要想清楚——分钟级延迟的看板，用 GCS 攒小文件 + 定时 `bq load`（比如每 5 分钟一次）成本可能低一个量级，流式适合"秒级新鲜度 + 高吞吐"的真实时场景，很多团队是两者混用：核心事件走流式、全量明细走批量。

**延伸考点：** Storage Write API 的"默认流"与"提交流（committed stream）"有什么区别？为什么"每秒上千条"的场景通常不用 committed stream 而用默认流 + 定期 flush？

---

### Q16. 数据在 GCS 上的 Parquet/Delta，想"查询但不搬迁"，BigLake 外部表方案怎么做？查询慢怎么办？

**问题：** 数据以 Parquet（带 Hive 风格分区目录）存在 GCS，需求是直接在 BigQuery 里查询且不搬迁数据。怎么建表？线上查询比原生表慢很多、扫描量也不对，怎么排查优化？

**期望加分项：**
- 能给出建表路径：BigLake 表（`CREATE EXTERNAL TABLE` 指定 `CONNECTION` + `METADATA_CACHE_MODE=AUTOMATIC`，格式 Parquet/Delta/Iceberg/Avro/ORC），配好 GCS 权限与连接（外部连接 + 服务账号）
- 能说清 BigLake 表 vs 普通 GCS 外部表的区别：BigLake 提供统一权限（用 BigQuery 侧 IAM 控制）、支持表级操作与 metadata cache，普通 external table 是直接映射
- 能讲性能优化：文件大小与数量（建议 128MB~1GB、目录文件别太碎）、压缩格式（Snappy/ZSTD）、列裁剪与谓词下推（Parquet 的 row group 裁剪）、metadata cache 刷新避免每次列目录元数据
- 能讲扫描与成本：外部表查询按扫描字节计费、无缓存/物化视图能力弱、Hive 分区目录要能被识别（`_PARTITIONTIME` 外的分区列正确声明）
- 有取舍意识：查询频率高的数据迁入 BigQuery 原生表（load），低频归档数据留外部表，讲清"存储费 vs 查询费"的置换
- 能指出 BigLake 支持 Delta/Iceberg 的差异：需要正确版本与配置，读的是 manifest/commit log

**减分项：**
- 不知道 BigLake 表与普通外部表的区别，或不知道 metadata cache
- 不关心文件大小/数量，遇到"外部表慢"只会说"加内存"
- 不知道谓词/列裁剪下推是外部表性能的关键
- 不考虑权限（连接、服务账号、对象 ACL vs IAM）
- 无脑让所有数据都走外部表，没有"热数据该迁入"的判断

**解答：**
方案主干是 BigLake 表：先在 BigQuery 建外部连接（Cloud Resource connection，授予对应服务账号 GCS 读权限），然后 `CREATE EXTERNAL TABLE ... WITH CONNECTION ... OPTIONS (format='PARQUET', uris=['gs://bucket/path/*.parquet'], metadata_cache_mode='AUTOMATIC')`，支持 Parquet、Avro、ORC 以及 Delta/Iceberg（lake 模式，读 commit log/manifest）。与最老的 GCS 外部表相比，BigLake 把权限统一到 BigQuery 侧（按 BigQuery 身份鉴权，而不是直接依赖对象 ACL），还带了 metadata cache——这是性能的分水岭：没有 cache 时每次查询都要实时列出并读取 GCS 元数据，文件多、目录深就慢；开 `AUTOMATIC` cache 后元数据被快照缓存并按窗口刷新，查询只按文件路径直接读。查询慢的排查清单按优先级：一、文件又小又碎——外部表性能对"文件数量"极其敏感，GCS 里几百万个小文件会拖垮 scan，优先治理上游，把文件合并到 128MB~1GB、目录按 Hive 分区布局（`dt=2026-08-01/`）；二、谓词下推——Parquet 支持按 row group 裁剪，`WHERE` 里的分区列（目录级）和普通列（文件内统计信息级）能否下推直接决定扫描量，检查查询是否命中声明好的分区列、有没有函数包裹列；三、压缩与编码——Snappy/ZSTD 是标配，文本/未压缩的换掉；四、cache 过期窗口内读到旧 schema/旧文件列表，重大文件变更后要手动刷新 cache。最后讲取舍：外部表省的是存储费、省不掉查询费（按扫描字节计费 + 无物化视图等优化手段），所以"高频查询 + 数据不大"的表应该 load 进原生表（还能用分区/聚簇/物化视图），"低频归档 + 大数据量"才留在 GCS 走 BigLake——这是存储费与查询费的置换，不是技术洁癖。

**延伸考点：** Delta 表在 BigLake 里以什么形式被读取（commit log 扫描）？如果 Delta 表频繁做 `OPTIMIZE`/`VACUUM` 产生大量小文件，对 BigLake 查询有什么影响？

---

### Q17. 报表团队只能看脱敏手机号、销售按区域只看本区域数据，BigQuery 怎么实现行列级安全？

**问题：** 合规要求：报表团队查 `user_phone` 只能看到脱敏值（如 `138****1234`），不能看原始列；销售团队只能查自己负责区域（region）的行。在 BigQuery 怎么落地？

**期望加分项：**
- 能说清列级访问控制（policy tag + Data Catalog taxonomy）：给敏感列打 policy tag，未授权用户默认看"受限值"（或不可见）
- 能说清 data masking（数据掩码）：在 policy tag 上配 masking rule（固定掩码 `138****1234`、哈希、null 等），授权用户看明文、未授权用户看掩码——比"列级不可见"更细
- 能说清行级安全：row-level security（`CREATE ROW ACCESS POLICY`），按 `SESSION_USER()`/group 动态过滤，sales 只能读 `region = 自己区域` 的行
- 能组合落地：列掩码 + 行过滤可以叠加，且掩码对导出/复制/`SELECT INTO` 同样生效（继承调用者权限）
- 有边界意识：row access policy 会阻止查询下推/影响分区裁剪与聚簇收益；外部表/BigLake 表不支持 RLS；masking 与 BI 工具的兼容性
- 能谈权限模型：先给组授数据集/表权限，再叠加列 tag 与行 policy，避免逐用户授权

**减分项：**
- 不知道 policy tag / masking / row access policy 这些原生能力，只会说"建视图"
- 分不清列级访问与掩码的区别（隐藏 vs 脱敏是两种语义）
- 不知道 RLS 对性能（分区裁剪失效）的影响
- 不知道导出/复制场景下掩码是否生效
- 逐用户授权，没有组/角色的组织思路

**解答：**
分三层实现。第一层列级访问控制：在 Data Catalog 建 taxonomy（分类体系），把 `user_phone` 这类敏感列打上 policy tag，并给 tag 配好"成员+权限"（谁可读明文）；打 tag 后，未授权用户查询该列默认被拦截或只能看到受控值——这是"能不能看这列"的开关。第二层数据掩码（data masking）：在 policy tag 上配置 masking rule（如固定掩码 `CONCAT(SUBSTR(phone,1,3),'****',SUBSTR(phone,9,4))`、哈希 SHA256、或直接 NULL），授权用户读明文、其他用户读掩码——这是"能看这列但看脱敏值"的开关，比整列隐藏更细。第三层行级访问：`CREATE ROW ACCESS POLICY` 或 `CREATE ROW ACCESS POLICY ap ON ds.sales FOR FILTER USING (region = SESSION_USER() 所在区域)`，销售查表时只返回命中区域的行，支持按 user/group/domain 写过滤表达式。组合使用没问题：一张表可以"列掩码 + 行过滤"同时生效，且掩码在 `SELECT`、导出、`CREATE TABLE AS` 里都继承调用者身份，防止"绕道导出拿到明文"。三个实践坑：一是 row access policy 加在表上后，优化器无法对表做分区裁剪和聚簇下推（必须先过过滤表达式），这类表查询扫描量会明显上升，行级安全要按表谨慎启用；二是 BigLake/外部表目前不支持 RLS，只有原生表能享受完整方案；三是掩码规则写错（比如掩码表达式抛异常）会让所有查询报错，上线前要灰度验证；权限组织上不要逐用户授权，用 group + 数据集级角色，policy tag 和 row access policy 都按 group 挂，否则权限矩阵会失控。

**延伸考点：** "列掩码"和"列级访问控制"在语义上有什么区别（隐藏 vs 脱敏）？报表团队既要 `user_phone` 的掩码、又要能 `GROUP BY` 它做聚合，怎么设计 tag 与权限才不冲突？

---

### Q18. 误删了一张分区表/分区，能恢复吗？快照（snapshot）与 time travel 怎么配合做备份？

**问题：** 生产上有人误删了一张分区表的某个分区（甚至整张表），现在要恢复。BigQuery 提供哪些恢复手段？怎么设计日常的备份/快照策略避免"删了救不回"？

**期望加分项：**
- 能说清 time travel：默认 7 天内可查任意时间点（`FOR SYSTEM_TIME AS OF`），误删的整表在 7 天内可用 `CREATE TABLE ... AS SELECT ... FOR SYSTEM_TIME AS OF` 或控制台恢复
- 能说清分区恢复：删除分区后 7 天内可用 `ALTER TABLE ... RESTORE`/`CREATE TABLE ... FOR SYSTEM_TIME AS OF` 恢复指定分区
- 能说清 BigQuery snapshots：创建快照不复制数据（写时复制/引用底层存储，零额外存储成本），可恢复（`CREATE TABLE ... CLONE`/copy from snapshot）、可跨区域保留；快照是只读、clone 是可写副本
- 能设计备份策略：快照窗口无法覆盖 7 天以上的需求 → 定期快照 + 把重要表复制到另一区域/项目做容灾
- 有边界意识：7 天是上限不是保险，删除前的 DDL 会覆盖同窗内旧版本；误删后要立即停止对该表的写入
- 能说清快照与 clone 的区别与各自用途

**减分项：**
- 不知道 time travel 窗口（说成 30 天/永久）
- 只知道"表删了能找回"，不知道分区级恢复语法
- 不知道 snapshot 零存储成本与"写时复制"原理
- 没有"跨 7 天必须靠快照/复制"的备份设计意识
- 把 snapshot 和 clone 混为一谈

**解答：**
先讲 BigQuery 内置的两层"后悔药"。第一层 time travel：任何表（含分区表）默认保留 7 天历史，任意时间点的数据都能查——`SELECT * FROM ds.t FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 2 DAY)`；表被删除后 7 天内可直接从控制台恢复，或用 `CREATE TABLE ds.t_restored AS SELECT * FROM ds.t FOR SYSTEM_TIME AS OF ...` 重建；误删分区的恢复同理：`ALTER TABLE ds.t RESTORE PARTITION ...`（或先建临时表取回分区数据再写回）。关键认知：time travel 的 7 天不是"保险"，而是"恢复窗口上限"，表被反复删建、DDL 变更都会干扰恢复路径，误删后第一步是**停止一切对该表路径的写入**，再动手恢复。第二层快照：`CREATE SNAPSHOT TABLE ds.t_snap CLONE ds.t FOR SYSTEM_TIME AS OF ...` 生成快照，快照引用底层存储块、不复制数据（写时复制），创建快照本身不产生额外存储费；恢复时 `CREATE TABLE ds.t_restored CLONE ds.t_snap` 即可。快照与 clone 的区别：snapshot 只读、适合备份与回滚源；clone 是可写副本、适合"复制一张表去改"。备份策略的现实答案：7 天窗口内靠 time travel；要跨 7 天（合规审计、防误删大于 7 天才发现），就定期（每天/每周）打 snapshot 保留 N 份，并把关键表用 `bq cp` 复制到另一个区域的项目做容灾——BigQuery 的复制是物理级、成本可接受。实践中的坑：snapshot 依赖底层存储块，源表 DDL 重建后旧快照可能失效，快照策略要和表的"变更流程"绑定（重大变更前打快照）；快照保留要设过期（`SNAPSHOT_TIME` 外可以配过期时间），否则存储引用会一直占着账。

**延伸考点：** 快照"不复制数据"是真的零成本吗？它的存储计费到底怎么算（写时复制 + 过期时间）？如果源表每天被"全量重刷"（数据全变），快照的增量成本会发生什么变化？

---

### Q19. Airflow 编排 GCS → BigQuery 的每日管道，GCSToBigQueryOperator 怎么用？有哪些必踩的坑？

**问题：** 数仓管道用 Airflow 编排：每天把 GCS 上的 CSV 加载进 BigQuery，再跑转换 SQL 产出报表表。加载和调度上要处理哪些工程细节？`GCSToBigQueryOperator`（新版 `GCSToBigQueryOperator`/`BigQueryInsertJobOperator`）常见坑有哪些？

**期望加分项：**
- 能给出 DAG 骨架：`GCSToBigQueryOperator`（load）→ `BigQueryInsertJobOperator`（转换 SQL）→ 校验/告警任务，并用 `data_interval_start` 参数化日期分区
- 能说清 load 关键参数：`write_disposition`（`WRITE_TRUNCATE`/`APPEND`/`EMPTY`）、`autodetect` vs 显式 schema、`skip_leading_rows`、`field_delimiter`、`schema_update_options`（`ALLOW_FIELD_ADDITION`）
- 能讲幂等：load 到分区表用 `WRITE_TRUNCATE` 限定目标分区（或 `bq load` 的 partition decorator），失败重试不会重复 append
- 能讲 schema 变更：CSV 加列后要 `ALLOW_FIELD_ADDITION`，否则 load 失败——这是线上最常见的"今天突然挂了"原因
- 能讲重试与告警：任务失败重试要有上限，转换 SQL 跑完要做行数校验（`BigQueryCheckOperator`/自定义 SQL）再放行下游
- 有取舍意识：简单"定时把 GCS 文件 load 进 BQ"也可以直接用 transfer service（`bq mk --transfer_config`），Airflow 适合复杂 DAG 与跨系统依赖

**减分项：**
- 不知道 `GCSToBigQueryOperator` 的存在，或说不出 load 参数
- 不处理 schema 变更（不加 `ALLOW_FIELD_ADDITION`），遇到"上游加列"就懵
- 幂等设计缺失：重试导致 append 重复数据
- 不做数据校验直接放行下游
- 把 Airflow 和 transfer service 的选择讲不清楚

**解答：**
DAG 主干三段：`GCSToBigQueryOperator` 负责 load（新版推荐 `BigQueryInsertJobOperator` 包 `bq load` 语义），转换用 `BigQueryInsertJobOperator` 跑 SQL，最后加校验任务。工程细节按坑的排序讲：第一，schema 变更——CSV 文件列和表结构不一致是 load 失败的头部原因，`autodetect` 只能救新表，已存在的表必须配 `schema_update_options=['ALLOW_FIELD_ADDITION']`（必要时加 `ALLOW_FIELD_RELAXATION`），否则上游一加列，当天管道直接红；第二，幂等与重试——load 的目标分区用 `WRITE_TRUNCATE`（配合分区装饰器/按 `dt` 写分区），这样"失败重试"是覆盖而非追加，避免重复数据；如果管道用 `APPEND`，重试一次就多一份数据，必须在下游用 `DISTINCT` 或先删后插对冲；第三，CSV 本身——`skip_leading_rows=1`、字段分隔符、引号/换行（`allow_quoted_newlines`）、编码（UTF-8 vs GBK 中文乱码是真实事故），文件里带 BOM 也会坑掉首列；第四，调度参数——用 `{{ data_interval_start }}` 生成目标分区 `dt`，让"补数/重跑"天然正确；第五，校验——load 后、转换前加 `BigQueryCheckOperator` 或自定义 SQL 检查行数/主键是否与源文件对账，再放行下游；失败重试上限（`retries`）+ 告警（`on_failure_callback`）是标配。选型边界：如果只是"每天固定把 GCS 某目录 load 进 BQ"，直接用 BigQuery transfer service（GCS transfer 或 scheduled query）更省事，Airflow 的价值在于跨系统依赖（先等上游系统就绪）、分支、复杂重跑语义，别为一条简单 load 管道硬上 Airflow。

**延伸考点：** `data_interval_start` 与 `execution_date` 在新版 Airflow 里的关系是什么？跨时区跑"北京时间昨天"的分区，模板里怎么取才不出错？

---

### Q20. 从 Hive/Redshift 迁到 BigQuery，选型对比什么？迁移中最常见的坑有哪些？

**问题：** 公司要把自建 Hive 或自管 Redshift 的数仓迁到 BigQuery。选型上怎么对比？如果已经决定迁，过程中最容易踩的坑和验证方式是什么？

**期望加分项：**
- 能说清三者定位差异：BigQuery（serverless、按扫描/slot 计费、GCP 生态、内建 ML）、Snowflake（计算存储分离 + 三副本、独立于云厂商、按 credit 计费）、Redshift（自管集群、固定成本、AWS 生态），按运维能力/成本模型/多云策略/生态绑定四维对比
- 能点出成本模型差异是最大认知冲击：从"固定集群成本"到"按扫描字节计费"，同样的 SQL 写法迁移后费用可能完全不同；迁移前必须重建成本治理（dry run 拦截、分区设计）
- 能说迁移执行：数据搬运（Hive 用 Dataproc/GCS 中转、Redshift 用 UNLOAD 到 S3 再跨云/直接 COPY）、SQL 方言改写（Hive QL：SERDE、`INSERT OVERWRITE`、动态分区；Redshift 方言差异；UDF 重写）、分区/聚簇策略重设计
- 能讲最容易踩的坑：小文件——Hive 的小文件问题原样搬过去，BigQuery 列存 + 分区下小文件拖慢扫描；`''` 空串 vs NULL 语义差异；Hive `STRING` 与 BigQuery 类型映射（BYTES/NUMERIC 精度）；时间精度（timestamp 微秒 vs 毫秒）；`LIMIT`/半连接等语法差异
- 能讲验证：双跑对账（两边并行跑 N 天，按关键指标行数/汇总值比对）、SLA 基线、查询超时对比
- 有渐进迁移意识：不是"一夜切换"，是"并行双跑 → 灰度切读 → 完全切流"

**减分项：**
- 选型只比功能清单，不比成本模型与运维模型
- 迁移只讲"搬数据"，不讲 SQL 改写、分区重设计、成本模型重建
- 不知道小文件、空串 vs NULL、类型映射这些数据层面的坑
- 没有对账验证方案，拍胸脯说"迁移完就行"
- 想"一刀切"切换，没有双跑/灰度设计

**解答：**
选型对比放四个维度：成本模型——BigQuery 按扫描字节/按 slot、Snowflake 按 warehouse 算力（compute credit）、Redshift 按固定集群资源，"用多少付多少"vs"买了就付"是最大分水岭，决定团队的用量治理机制；运维模型——BigQuery serverless 无集群可管，Snowflake 也是托管但 warehouse 规模要调，Redshift 要自己管节点、扩容、vacuum；生态绑定——用 GCP 全家桶（Dataflow、Looker、Vertex AI）选 BigQuery 集成最顺，AWS 深度绑定选 Redshift，想多云中立、跨云数据共享选 Snowflake（其共享机制是差异化优势）；能力面——BigQuery 内建 ML（BigQuery ML）、BI Engine、地理/时间序列函数是一体化优势。决定迁移后，坑集中在四类：一是数据层——Hive 小文件问题原样搬过去会让扫描被文件开销拖垮，迁移时顺手按 BigQuery 的分区/聚簇原则重建布局；空串与 NULL 语义（Hive 里 `''` 和 NULL 混用）、`STRING` 类型映射（BigQuery 的 `STRING` 是 UTF-8、`BYTES` 才是二进制）、`NUMERIC` 精度（Hive DECIMAL 到 BigQuery NUMERIC 有精度上限）都要在 schema 评审时逐个过；二是 SQL 层——`INSERT OVERWRITE`/动态分区语法、`SERDE` 表定义、UDF（Hive UDF 要改写成 BigQuery UDF/函数）、`QUALIFY` 等新语法引入，语法改写清单要提前整理；三是成本层——同一个"查全表"的报表从 Redshift（固定成本）迁到 BigQuery 按需模式，费用直接变成"扫描量×单价"，迁移必须同步上成本治理（dry run 拦截、分区裁剪、物化视图）；四是验证层——双跑对账：并行跑 N 天，按关键表行数、主键去重数、汇总指标逐日比对，对账通过才切读；迁移节奏用"并行双跑 → 灰度切读 → 完全切流"，并保留回退路径（旧系统继续保留一个周期）。最后提醒：迁移不是技术项目而是成本治理项目，立项时就把"迁后 3 个月的扫描量红线/账单看板"一起做了。

**延伸考点：** "同样的 SQL 在 Redshift 上是固定成本、在 BigQuery 按需模式下按字节计费"——迁移后哪些"以前没人管的查询"会变成账单炸弹？迁移计划里你怎么设计成本护栏？
