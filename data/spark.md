# 数据 · Spark（面试题库）

本文件聚焦数据工程师在 Spark 上的真实工程能力，覆盖执行原理（宽窄依赖、Stage/Job 划分、Shuffle）、Spark SQL 执行计划与 Catalyst/AQE、数据倾斜治理（groupBy/join 加盐）、内存与 JVM 调优（executor 内存、OOM、Kryo、缓存级别）、小文件治理、Structured Streaming（微批、窗口、watermark、exactly-once）、批流一体、读写性能优化（Parquet 裁剪、谓词下推、JDBC 并行度）以及资源规划、血缘与作业稳定性等主题。题目均为线上可复现的场景化问题，重点考察排查链路、量化依据与取舍判断，不考概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

### Q1. 同样的作业，新同学问"Job、Stage、Task 是怎么切出来的"，你怎么用一张图讲清楚？

**问题：** 团队里新人接手一个 Spark SQL 任务（Hive 表 JOIN + GROUP BY + 排序写出），问你一个 action 触发的执行过程里 Job、Stage、Task 分别怎么划分，宽窄依赖在这中间起什么作用，你会怎么讲？

**期望加分项：**
- 能从 action 触发 → Job 切分讲起，明确"Stage 的分界点就是宽依赖/Shuffle"
- 能讲清窄依赖算子（map/filter/coalesce 上游合并）可以流水线化进同一个 Stage，Task 数与分区数一一对应
- 能结合具体 SQL 指出 ShuffleDependency 出现在哪几个算子（groupBy 的 HashAggregate、join 的 SortMergeJoin、distribute by）
- 知道 Spark SQL 下物理计划里 `Exchange` 算子就是宽依赖的落点，Task 并行度由分区数决定
- 能说清 Spark UI 上 Job/Stage/Task 的对应关系，以及 DAG 图里哪些节点能一眼看出性能隐患
- 会主动提到 `spark.default.parallelism`/`spark.sql.shuffle.partitions` 决定 shuffle 阶段分区数

**减分项：**
- 把"宽窄依赖"背成定义，但说不清它和 Stage 划分的直接关系
- 分不清 Job（action 触发）与 Stage（宽依赖切分）的层级
- 不知道 `Exchange`/ShuffleDependency 在物理计划中的位置
- 讲完原理给不出从 Spark UI 验证的路径（哪个 Tab 看 Stage、哪个看 DAG）
- 忽略 Task 并行度 = 分区数这个关键连接点

**解答：**
先给一句话锚点：**Job 由 action 触发，Stage 按宽依赖切分，Task 按分区数生成**。讲的时候拿具体 SQL 走一遍：读两张表各是一个 `Scan`（若干并行 Task，分区数由文件分片决定）；`JOIN` 在物理计划里变成 `Exchange`（ShuffleDependency）+ `SortMergeJoin`，这个 Exchange 就是 Stage 分界点，上游两边各一个 Stage，shuffle 后按 join key 重分区再进入下游 Stage；`GROUP BY` 同理，`HashAggregate` 之前也有一个 Exchange（partitions 数受 `spark.sql.shuffle.partitions` 控制）；最后的 `Sort`/写出属于同一个下游 Stage。一张图就是"Scan → StageA →（shuffle）→ StageB →（shuffle）→ StageC"，Stage 内所有算子可流水线执行，数据在内存/本地流式传递；Stage 之间要落盘 shuffle 文件并等待所有上游 Task 完成，这是任务慢的主因。给新人的实操验证路径：Spark UI 的 Jobs 页按 action 分 Job，Stages 页显示每个 Stage 的 Task 数与完成时间，DAG Visualization 上带 `shuffle` 标号的节点即宽依赖。坑：窄依赖出现背压（如单个 map 慢）说明数据分片不均或单条记录处理重；而宽依赖阶段出现长尾则优先怀疑数据倾斜——这两类问题的排查方向完全不同，这也是为什么"先判断宽窄"是定位一切 Spark 性能问题的第一步。

**延伸考点：** 同一个 Stage 里如果某个 Task 特别慢，和宽依赖阶段某个 Task 特别慢，排查思路有什么不同？什么情况下一个 Stage 的 Task 数会远大于分区数（如 task 被重试/推测执行）？

---

### Q2. GROUP BY 之后某个 Stage 有 199 个 Task 几秒跑完，剩下 1 个 Task 跑了 40 分钟，怎么定位和救？

**问题：** 线上一个 Spark SQL 任务，Shuffle 后的聚合 Stage 共 200 个 Task，其中 1 个 Task 卡了 40 分钟、其他几秒完成，任务整体超时。你从哪几步入手定位，能现场给出处理方案吗？

**期望加分项：**
- 第一时间从 Spark UI 的 Stage 页看 Task 分布：输入数据量、Shuffle Read 量的 max/median 对比，确认是倾斜而非单任务本身慢
- 能继续确认倾斜发生在 map 端输出还是 reduce 端拉取（Shuffle Write 大小、GC 时间、内存溢出迹象）
- 给出分层方案：先临时止血（提高该任务资源/调大 shuffle partitions 分散热点），再根治（对 key 做业务分析，热点 key 加盐/拆 key）
- 能提到通过 `spark.sql.adaptive.skewJoin.enabled` 或手动 AQE 优化让框架自动拆分倾斜分区
- 有量化意识：用 "max Shuffle Read / median" 的比值说明倾斜严重程度
- 主动考虑边界：热点是单 key 还是少数几个 key、是否业务上可接受拆分后合表

**减分项：**
- 不看数据直接说"加资源/加分区数"，说不清为什么分区数能缓解
- 把倾斜定位停留在"猜测"，没有用 UI/指标（Shuffle Read 量）佐证
- 只给"加盐"两个字，说不清加盐后怎么聚合、怎么去盐，以及加盐的代价
- 忽略 AQE 的自动倾斜处理，也不知道它和手动加盐怎么取舍
- 不检查倾斜是否由 join 引起、是否重复 key 本来就多（业务脏数据）

**解答：**
定位三步走。第一步看 Spark UI：进到慢的 Stage 页，对比各 Task 的 Shuffle Read 大小，若 max 是 median 的几十上百倍，基本确认数据倾斜；再看输入与 Shuffle Write 是否均匀，排除是某台机器 IO/GC 问题。第二步确定热点形态：`group by key` 阶段倾斜，多半是少数热点 key（如 null 值、明星店铺、默认渠道）被分到同一个分区；可以临时 `select key, count(*) group by key order by count(*) desc limit 10` 验证（注意这个查询本身也可能慢，可加过滤采样）。第三步处理：临时止血——若热点 key 就一两个，可用 `filter` 把热点 key 单独拆出去聚合再 union；治本——热点 key 加盐：对倾斜 key 拼接随机后缀（如 `concat(key, '_', rand()*N)`）先打散到 N 个分区做部分聚合，再对结果去掉盐后缀做第二次聚合；对 null 这种"脏 key"，先看业务上能否过滤或置默认值。框架层：Spark 3.x 开启 `spark.sql.adaptive.skewJoin.enabled`（配 `skewedPartitionFactor`/`skewedPartitionThresholdInBytes`），AQE 会在 shuffle 后识别并拆分倾斜分区，适合 join 型倾斜；但 groupBy 型倾斜 AQE 目前不自动处理，需手动加盐。实践中的坑：加盐后输出行数若仍需保序需额外处理；对本来就是唯一 key 的数据加盐会白增一次 shuffle；先确认倾斜是"单 key 巨大"还是"少量 key 巨大"，决定拆 key 还是加盐，方案完全不同。

**延伸考点：** join 场景的倾斜和 groupBy 场景的倾斜，为什么 AQE 只对前者自动生效？加盐粒度 N 怎么选，加太大有什么副作用？

---

### Q3. 同一段 SQL 逻辑，新同事用 DataFrame API 写出来跑得慢，你让他先干什么？

**问题：** 同样的查询，同事先用 Spark SQL 写，后来改成 DataFrame API 写法，发现新版本慢了很多，且逻辑看不出明显问题。你让他第一步干什么，如何利用执行计划定位？

**期望加分项：**
- 明确第一步不是看代码，而是 `explain()`（`df.explain(true)`/`EXPLAIN EXTENDED`）对比两版物理计划，看是否产生了多余的 Shuffle 或缺少谓词下推/列裁剪
- 能读执行计划关键点：`Scan` 的 `PushedFilters`、`Exchange` 数量、`Sort` 是否出现、Broadcast 是否生效
- 知道 DataFrame API 常见坑：`filter` 后 `count` 前触发了两次 action、`select` 中函数导致列裁剪失效、`join` 条件书写方式影响 BroadcastJoin 触发
- 知道 Catalyst 优化器阶段（逻辑计划 → 优化规则 → 物理计划 → 代码生成），能说出谓词下推、列裁剪、常量折叠等常见优化
- 能利用 AQE 与动态分区裁剪：分区表查询没走分区过滤（`PushedFilters` 缺 partition filter）导致全表扫
- 会用 `spark.sql("EXPLAIN")`、History Server 看执行计划与耗时

**减分项：**
- 只会说"看执行计划"但列不出具体看什么字段
- 不知道 UnresolvedLogicalPlan → LogicalPlan → OptimizedLogicalPlan → PhysicalPlan 的链路
- 说不出 DataFrame 写法常见的反模式（如循环中多次 `collect`、`udf` 阻断优化）
- 不检查是否多扫了列/多拉了分区
- 直接建议"加缓存"这类没有依据的动作

**解答：**
第一步就是对比执行计划：新版 `df.explain("extended")` 输出 4 段——Parsed/Logical/Optimized/Physical，物理计划重点看三个信号：① `Scan` 节点上的 `PushedFilters` 与 `PushedDownPredicates`，确认过滤下推到了数据源（分区裁剪靠 `PartitionFilters`）；② `Exchange` 节点个数，多出的 `Exchange` 意味着引入了多余 shuffle，往往是 join 条件写反/类型不一致导致 `BroadcastHashJoin` 退化为 `SortMergeJoin`；③ 物理计划里的 `*` 号（Codegen）是否普遍生效。再对照优化的 LogicalPlan，看 Catalyst 是否做了列裁剪——DataFrame API 常见坑是 `select(col("a"), some_udf(col("b")))` 或 `withColumn` 链把 UDF 挂在过滤条件上，导致下推失效；另一个高频坑是 `filter($"ts" > "2024-01-01")` 这种把分区列和字符串比较写错类型，使 partition pruning 失效，日志里会看到 Scan 的 `PartitionFilters` 为空。经验流程：先跑 `explain(true)` 拿到物理计划 → 数 `Exchange` 和 `Sort` → 看 `Scan` 的裁剪字段 → 再对照 SQL 版本找差异。实践中的坑：不要只看一版计划，要对比改动前后的两版；用 `spark.sql.adaptive.enabled` 时物理计划展示的是优化后结果，需结合 `EXPLAIN EXTENDED` 和运行时的 `QueryExecution` 事件（可在 History Server 的 SQL 页看 `Query Details`）综合判断；写 UDF 尽量用内置函数或改用 SQL 表达，避免阻断 Catalyst 优化链路。

**延伸考点：** 分区表查询没有走分区裁剪时，你能从哪几处证据确认（执行计划、Task 输入量、History Server）？BroadcastJoin 退化成 SortMergeJoin 的常见触发原因有哪些？

---

### Q4. 广播变量和累加器都用过吗？说说线上用它们踩过的坑。

**问题：** 团队统计里多次用到广播变量（broadcast join 阈值外的大字典）和累加器（统计脏数据行数），最近出现"广播变量序列化失败""累加器计数不对"两个线上问题，你能分析原因并给出正确用法吗？

**期望加分项：**
- 能讲清广播变量的本质：driver 端收集 → 序列化 → 分发给每个 executor 的每个 task（实际上是每 executor 一份块），与闭包捕获 `collect` 的区分
- 知道广播数据过大（默认阈值 10MB、强制广播可到几百 MB）会撑爆 driver/executor 内存与网络，且序列化要求对象可序列化（case class/POJO 加 `@transient`、Kryo 注册）
- 能说出累加器的坑：**在 action 里更新累加器不保证执行**（Spark 只保证每个 task 的累加器更新最多应用一次，transform 中多次执行/重算会导致重复或漏计），正确做法是在 action 中更新或只依赖 job 完成后读取
- 知道累加器不能用于 RDD 之间的跨 Stage 精确依赖、宽依赖下重算会重复累加，用 `sparkContext.longAccumulator` 并注意 driver 端读值时机
- 能提 `broadcast` 与 `broadcast join` 的关系、`spark.sql.autoBroadcastJoinThreshold` 的动态调整，以及 `TreeBroadcast` 分块发送机制
- 实践中先 `df.count()`/`show` 触发 job 后累加器值才可用，注意 action 前读取为 0

**减分项：**
- 分不清广播变量与闭包捕获：以为每次 `map` 里引用的大对象会自动广播
- 累加器只在 transform 中 `+=`，然后立刻在 driver 读，得到 0 或不准，讲不出原因
- 不知道广播变量是每 executor 一份而不是每 task 一份，导致对内存开销估计错
- 把累加器用在 `foreachPartition` 内做业务写库前的计数，却没考虑重试重复计数
- 序列化失败只会"换成 Kryo"但说不出原因（未注册、`@transient` 未加）

**解答：**
两个问题分开答。**广播变量**：当 join 小表超过 `spark.sql.autoBroadcastJoinThreshold`（默认 10MB）但业务上仍希望广播（如维度表几百 MB、且 shuffle 代价更高）时，可用 `sparkContext.broadcast(map)` 手工广播，并通过 `spark.sql.autoBroadcastJoinThreshold=-1` 关闭自动广播、或直接用 `hint /*+ BROADCAST(t) */`。广播的坑有三类：① 内存：广播数据是每 executor 一份（而非每 task），几十个 executor × 500MB 字典，加上 Kryo 序列化副本，很容易把 executor 堆打爆，表现为 executor 频繁 Full GC 或 OOM——先评估 `executor数 × 广播大小` 是否可承受，必要时按维度拆多个小广播或改用 join 前预过滤；② 序列化：广播的 case class 若引用了 driver 端不可序列化的资源（JDBC 连接、`ObjectInputStream`），会报 `Task not serializable`——把资源对象标 `@transient`、在 task 内按需创建连接；③ 更新语义：广播后想要"动态更新维度"是做不到的，只能重建新广播变量，这是与闭包捕获的最大区别。**累加器**：用来做只增不减的统计（脏数据计数、跳过行数）。核心坑是**累加器更新只保证"每个 task 最多应用一次"**，因此绝不能在 `map`/`filter` 这类可能被重算（stage 重试、speculative 推测执行）的 transform 里用它做依赖逻辑，计数会重复；正确姿势是：在 `foreach`/`foreachPartition`（action）内更新，job 完成（`action` 返回后）在 driver 读 `value`。另一个坑：同一个 `DataFrame` 被多次 action 触发时累加器会累加多次，需要先 `cache` 或按 job 复位。实践建议：能用 `spark.sql("SELECT count(...)")` 表达的就用 SQL，累加器只用于"无法用 SQL 表达的旁路统计"。

**延伸考点：** 推测执行（speculation）开启时，累加器会怎样？广播变量更新"维度表每日变更"有什么替代方案（如用外部表 join）？

---

### Q5. 任务每天都因"小文件过多"变慢，你在写入侧怎么改？

**问题：** 一个每天跑的任务：读取上游明细 → 按天分区写入 Hive 表。最近下游查询越来越慢，检查发现每个分区下有几万个 KB 级小文件。你在写入侧会做哪些改造（coalesce/repartition/动态分区），依据是什么？

**期望加分项：**
- 能指出小文件产生的直接原因：写出 task 数 = 最终分区数（`spark.sql.shuffle.partitions` 默认 200 或上游分片数），每个 task 各写一批文件；若开启动态分区，写入分区数 × shuffle 分区数叠加放大
- 能区分 `coalesce(n)`（窄依赖、只减分区、适合在 shuffle 后的结果上缩）与 `repartition(n)`（全量 shuffle、可增可减），并说出什么时候必须用 repartition
- 结合分区键写出场景：写前 `repartition(col("dt"))` 保证同一分区数据落在同一批 task，配合控制 task 数使"分区数 × 每分区 task 数"合理
- 给出量化目标：单文件 64~128MB 量级，结合数据量推算 target 分区数（数据量 / 目标文件大小）
- 能提动态分区开关 `spark.sql.hive.convertMetastoreParquet`、`spark.sql.optimizer.dynamicPartitionPruning`（分区裁剪）以及写入时 `partitionOverwriteMode=dynamic` 的正确姿势
- 能提后续配套治理：AQE 的 post-shuffle 分区合并、`OPTIMIZE`/compaction 方案，或调 `spark.sql.adaptive.coalescePartitions` 让框架自动合并

**减分项：**
- 只会说"写之前 coalesce(1)"，不知道 coalesce 窄依赖只影响分区数量、数据倾斜时反而更慢
- 不知道动态分区 + 大 shuffle partitions 是小文件放大的核心机制
- 分不清 coalesce 与 repartition 何时等价何时不等价（是否有 shuffle）
- 不提目标文件大小依据，拍脑袋定分区数
- 不考虑下游读取方式（点查 vs 全扫）对文件大小的不同要求

**解答：**
先量化目标：假设单分区数据 10GB、目标单文件 128MB，则写该分区时并行度约 80 即可，多出来的并行度只会把小文件翻倍。写法上推荐"先按分区键 repartition，再控制输出并行度"两步：`df.repartition(80, col("dt")).write.mode("overwrite").partitionBy("dt")`——`repartition(80, dt)` 保证同一 dt 的数据进同一批分区（还带来写入侧优化，Spark 2.4+ 对 `repartition`+`partitionBy` 写路径有按分区合并小文件的优化），输出 task 数≈80。若数据本来就来自 shuffle 结果（如 groupBy 后），用 `coalesce` 收拢更省一次 shuffle：`df.coalesce(80).write...`。为什么是 coalesce：它是窄依赖，只把相邻分区合并，没有网络传输；但注意 coalesce 不能"增"分区，且若上游并行度极低（如单分区数据 100GB 只有 1 个 task），coalesce 救不了，需 repartition 先打散。动态分区场景的坑：`partitionBy("dt")` + `spark.sql.shuffle.partitions=200` 时，写一个分区可能由多个 task 各写一份，产生"分区数 × 200"的文件；且开启 `dynamic` 分区模式（`spark.sql.sources.partitionOverwriteMode=dynamic`）时按分区覆盖，若上游并行度未收敛，小文件会成倍放大。框架级兜底：Spark 3.x 开 AQE 后 `spark.sql.adaptive.coalescePartitions.enabled`（默认开）会在 shuffle 后按 `advisoryPartitionSizeInBytes`（默认 64MB）自动合并分区，直接缓解"groupBy 写出的文件多"问题。实践中的坑：不要为压小文件无脑 coalesce(1)，单分区数据量大时单个 task 写 1GB+ 反而拖慢写出且后续并发读受损；文件大小目标要与下游引擎（Hive/Impala/Presto）读取模型匹配；对小表任务，写前 `cache` 防止 `repartition` 触发两次计算。

**延伸考点：** `coalesce` 在什么场景下会比预期更慢（如上游 task 分布不均、数据局部性差）？`partitionOverwriteMode=dynamic` 与静态覆盖混用时有什么数据风险？

---

### Q6. 任务报 OOM，怎么判断是 driver 还是 executor、堆内还是堆外？

**问题：** Spark 任务隔三差五 OOM，报错信息有时是 executor 端 `java.lang.OutOfMemoryError: Java heap space`，有时是 `Direct buffer memory`。你会怎么定位是 driver 还是 executor、堆内还是堆外，并给出针对性配置与代码修改？

**期望加分项：**
- 明确判断入口：看报错发生在 driver 日志还是 executor 日志（driver 日志见 `spark-submit` 控制台/AM 日志，executor 日志在 YARN container 日志或 `stdout/stderr`），报错堆栈中的类（如 `ShuffleExternalSorter`、`TaskMemoryManager`）指向堆内执行内存
- 能讲清 executor 内存模型：堆内 `spark.executor.memory`，其中 `spark.memory.fraction`（默认 0.6）内执行/存储共享、`spark.memory.storageFraction`（0.5）为存储保护带；堆外 `spark.memory.offHeap.enabled`+`spark.memory.offHeap.size`；`spark.executor.memoryOverhead`（默认 max(384MB, 10%)）用于 direct buffer/网络/线程栈
- 常见堆内 OOM 场景能举例：collect 大结果到 driver、广播变量过大、shuffle 数据量大导致执行内存不足、缓存过多挤占执行内存（storageFraction 保护失效）
- 常见堆外 OOM：Kryo 序列化 buffer、Parquet/ORC 读写的 direct buffer、`spark.network` 传输——报 `Direct buffer memory`/`Unable to reserve X bytes of direct buffer memory`
- 给方案分优先级：先改代码（去掉 `collect`、`repartition` 控制、cache 前评估），再调参数（`spark.executor.memory`、`spark.memory.offHeap.size`、`spark.executor.memoryOverhead`、`spark.driver.maxResultSize` 默认 1G）
- 会用 Spark UI Executors 页看内存柱状图（Storage Memory / Execution Memory / Overhead）与 GC 时间曲线

**减分项：**
- 只知道"加内存"，说不清加哪一块（堆内 vs 堆外、driver vs executor）
- 分不清 `spark.executor.memory` 与 `spark.executor.memoryOverhead` 的用途
- 报 `Direct buffer memory` 却去加 `spark.executor.memory`，方向错误
- 不检查代码层面的根因（如大 `collect`、`count` 前的 `cache` 未释放）
- 不会用 UI 确认当前内存分布，只会看报错文本

**解答：**
判断顺序：先看日志归属（driver vs executor）再分堆内外。**driver OOM 的典型**：`collect()`/`take()` 大结果、广播变量创建时收集大数据、`spark.driver.maxResultSize`（默认 1G）超限报错——特征是报错在提交端/AM 日志，堆栈指向 `DriverEndpoint`/`SparkContext` 相关类；修法是改代码（分批 collect、`df.write` 落盘替代 collect）+ `spark.driver.memory`。**executor 堆内 OOM**：报 `Java heap space`，堆栈常见 `ShuffleExternalSorter`（shuffle 写侧内存）、`AppendOnlyMap`（聚合内存）、`RowIterator`（读侧缓冲）；这是执行内存（`spark.memory.fraction` 内的 Execution 部分）不足，或倾斜导致单 task 数据超内存。排查时进 Executors 页看 Execution Memory 是否打满、GC 时间是否占比高，配合看是否 cache 占了 Storage 内存挤占执行区（`spark.memory.storageFraction` 是保护带，但超额会驱逐缓存，极端下仍可能触发 evict 后执行内存不足）。修法优先级：先压数据（过滤无效字段、`repartition` 分散、倾斜加盐），再调 `spark.executor.memory` 与 `spark.sql.shuffle.partitions`，必要时 `spark.memory.fraction` 调大给执行区（但存储区变薄，缓存易被逐出，取舍要说明）。**executor 堆外 OOM**：报 `Direct buffer memory`/`OutOfMemoryError`（非 heap），堆栈出现 `sun.nio.ch`、`io.netty`、Kryo `Output/Input`——这是 direct buffer 用爆了 `spark.executor.memoryOverhead`（默认按 memory 的 10% 计算且不低于 384MB）；调大 overhead，同时检查是否单 partition 数据量超大（Parquet 读用 direct buffer 做 decode）、是否 `spark.io.compression` 相关。实践中的坑：OOM 后 `spark-submit` 侧看到 executor 被杀（YARN 报 `Container killed by YARN`），要区分是"内存超 YARN 限制被杀"（此时该加 memory+overhead 总和）还是"堆内 OOM"（堆栈明确 Java heap space）；K8s 场景 container 被 OOMKilled 同理看 `exit code 137`。核心心法：OOM 十有八九是倾斜/collect/广播，先看代码再调参。

**延伸考点：** YARN 上报 "Container killed by YARN for exceeding memory limits" 与 executor 端 Java heap space，两者处理差异是什么？`spark.memory.fraction` 调大的副作用是什么？

---

### Q7. 缓存了某中间 DataFrame 后任务反而变慢甚至 OOM，怎么回事？

**问题：** 你把一个中间结果 `df.cache()` 后供后续多个 action 复用，结果任务更慢了，还偶发 executor OOM。同事说"缓存不一定都有用"，你怎么判断缓存该不该用、用哪种缓存级别，以及缓存失效与内存不足的处理？

**期望加分项：**
- 能讲清缓存收益条件：同一 DataFrame 被多次 action 复用且每次都要重新计算整条血缘时才有收益；只 action 一次则缓存是纯开销
- 能说清缓存级别取舍：`MEMORY_ONLY`（默认，反序列化对象，CPU 友好但占堆）vs `MEMORY_AND_DISK`（内存放不下落盘，防 OOM 兜底）vs `MEMORY_ONLY_SER`/`MEMORY_AND_DISK_SER`（Kryo 序列化，省内存省 GC，代价是每次读要反序列化）vs `OFF_HEAP`（堆外，配 Tachyon/堆外内存，免 GC）
- 能结合数据量给判断依据：估算 `行数 × 行大小`（可用 `df.storageLevel` 后看 UI Storage 页实际占用），对比可用存储内存
- 知道缓存与"shuffle 中间文件复用"的区别：Spark 对 shuffle 结果本身会复用（`shuffleFilesToBeDeleted` 机制），不需要重复计算的部分不必 cache
- 缓存失效处理：`unpersist()` 时机、`spark.cleaner.referenceTracking`、UI Storage 页看缓存是否被逐出（`RDD x Storage Level: Memory Deserialized, Size in Memory`）
- 能提 `checkpoint()` 与 cache 的区别（截断血缘 vs 保留血缘）

**减分项：**
- 以为 cache 后所有 action 都必然变快，答不出"只算一次时缓存无收益"
- 说不清 `_SER` 后缀与无后缀缓存级别的内存/CPU 差异
- 不知道 Storage 页怎么确认缓存真实占用与逐出情况
- 缓存对象是 shuffle 产物时看不出"重复计算 vs 复用一个 shuffle 文件"的差别
- 缓存后不 `unpersist`，多个大缓存叠加把执行内存挤爆

**解答：**
先做收益判断：缓存只对"血缘重复计算"有价值。比如 `val base = raw.filter(...).join(...)`，后面有三个 `base.groupBy(...)`，若不缓存，每个 groupBy 的 action 都会把整条 base 血缘重算一遍（map 端 stage 重跑），此时缓存收益明确。但若 base 是某次 shuffle 的输出、或后续只 action 一次，则缓存纯属多余——甚至更慢：序列化/反序列化开销 + 占存储内存挤占执行内存。判断方法：`base.rdd.getStorageLevel` 不重要，重要的是先用 `base.count()` 触发计算并 cache 后去 UI **Storage 页**看 `Size in Memory` 与 `RDD Blocks`，估算可用存储内存（executor 数 × `spark.executor.memory` × fraction × storageFraction）能不能装下；装不下时默认 `MEMORY_ONLY` 会触发部分驱逐或落盘（`MEMORY_ONLY` 无磁盘后备时超出的分区会在下次使用时**重算**，这是"缓存后反而变慢"的典型原因——你以为缓存了，实际大部分分区被逐出）。解法：改用 `MEMORY_AND_DISK`（超限落盘、防重算防 OOM，但读盘慢）或 `MEMORY_ONLY_SER`/`MEMORY_AND_DISK_SER`（Kryo 序列化后体积通常小 3~5 倍，配合 `spark.serializer=KryoSerializer`，省堆省 GC；代价是每次读取要反序列化，适合"计算重、缓存数据大"的场景）。OFF_HEAP 需要额外配置堆外内存，仅当堆内被 JVM 结构挤满时才有收益。另外两个易漏点：① 缓存的 RDD 分区**不均衡**时（缓存前未处理倾斜），大分区超限被逐出，热点反复重算，症状与"缓存没用"一致；② shuffle 输出本身会被 Spark 复用（spill/block 管理），若 base 是 join/groupBy 产物，下游再复用未必需要 cache。收尾：用完 `unpersist()`；需要容错且血缘过长时用 `checkpoint()`（截断血缘、落可靠存储），它与 cache 配合（cache 后 checkpoint）是最稳的组合。

**延伸考点：** 缓存分区被逐出后 Spark 如何恢复它（重算血缘 vs 磁盘备份）？`checkpoint` 与 `cache` 在容错语义上的本质区别是什么？

---

### Q8. 接手一张日增量写入、已有上万个分区、每个分区上千小文件的 Hive 大表，你打算怎么系统性治理？

**问题：** 你接手一张核心事实表，每天增量写入，积压了上万个分区，每个分区几百到上千个 KB 级小文件，下游 Spark 任务读它越来越慢。你如何设计一套"不停业、可持续"的治理方案？

**期望加分项：**
- 先分级：按"读取频次 × 数据量"把表分区分层（热分区/冷分区），治理策略不同，避免一刀切全量合并
- 能给出合并方案的取舍：Spark 重写（`repartition`/`coalesce` 重写）vs Hive `concatenate`（ORC 支持、不动数据只合 Stripes）vs 数据湖 `OPTIMIZE`（Delta/Iceberg compaction，增量文件合并）
- 知道合并作业自身的坑：合并过程要控制并行度与内存、要错峰执行、要监控合并前后的文件数/大小/耗时，避免合并作业比业务作业还重
- 有"读时合并"思路：对某些场景（如列存格式 + 谓词下推）小文件读取开销可通过 `spark.sql.hive.convertMetastoreParquet`、并行度与动态分区裁剪缓解，不一定要全部物理合并
- 能讲清增量治理闭环：写入端控制单文件大小（结合 Q5）→ 定期 compaction → 指标监控（文件数、平均大小、每分区文件数）→ 超阈值告警
- 主动考虑边界：下游有实时点查/join 场景的权衡、合并期间的读写一致性问题（Hive 目录 rename vs 数据湖 snapshot）

**减分项：**
- 只会"写个 Spark 任务全表重写一遍"，不考虑表大小（几 TB 全量重写成本）与停业窗口
- 不知道 `concatenate` 只对 ORC 有效，也不知道它和重写的区别（不重算数据）
- 不做分层/优先级，无差别治理
- 没有监控闭环，合并完三个月又变回原样
- 忽略合并与下游读取的并发影响

**解答：**
设计分四步。第一步盘点与分层：用 `show partitions`/`hdfs dfs -count` 按分区统计文件数与大小，把表按"读频 × 数据量"分成热分区（近 N 天、常被查询）与冷分区；热分区文件大小目标 128MB 量级，冷分区可容忍合并到 256MB+（点查少、全扫多）。第二步选合并手段：① 冷分区/归档：一次性的 Spark 重写，`df.repartition(n).write.mode("overwrite")` 重建，或对 ORC 表用 `ALTER TABLE t PARTITION(...) CONCATENATE`——concatenate 只合并物理文件/Stripe、不重算数据，成本低，但前提是表必须是 ORC 且 Hive 元数据支持；② 增量表：如果表已经迁到数据湖（Delta/Iceberg），用 `OPTIMIZE t ZORDER BY ...` 做 compaction + 布局优化，后台跑、不阻塞读写，这是长期方案；③ 写前控制：结合写入侧改造（Q5），保证新分区不再产生小文件。第三步节奏与窗口：合并作业放到凌晨低峰，控制 `spark.executor.cores` 与并行度避免与在线作业抢资源；合并前对表加锁或确认上游无并发写（Hive 场景避免 merge 与 append 并发导致数据丢失——用分区级处理而非全表）。第四步监控闭环：按表/分区粒度记录文件数、总大小、平均文件大小到监控系统，设阈值（如"近 7 天分区平均文件 < 64MB 或文件数 > 500"）告警，治理后每周看趋势。实践中的坑：全表重写几 TB 的表可能要跑几个小时且占用大量磁盘做临时写，先按分区错峰；concatenate 后 HDFS 是原文件删除+新文件，注意 NameNode 瞬时压力；小文件问题的本质是"任务并行度与写文件数不匹配"，治理端永远要回补写入端规则，否则只治标。另外读侧短期缓解：开 `spark.sql.files.maxPartitionBytes`、`spark.sql.files.openCostInBytes`（默认 4MB）合理估值、增大并行度，让单个 task 能合并读取多个小文件，能先拖住业务再从容治理。

**延伸考点：** `spark.sql.files.openCostInBytes` 和 `spark.sql.files.maxPartitionBytes` 对读取小文件任务的并行度如何起作用？增量写 + 并发合并时，Hive 场景怎么避免数据丢失/重复？

---

### Q9. 你们 Spark 3 开了 AQE，但有个任务还是慢，你怎么验证 AQE 有没有真正生效？

**问题：** 团队把 Spark 升级到 3.x 并开启了 AQE（自适应查询执行），但某几个任务该慢还是慢。你怎么确认 AQE 是否在起作用、哪些场景它救不了，以及什么时候该关掉它退回手动调优？

**期望加分项：**
- 知道 AQE 三大能力及各自开关：动态合并 shuffle 分区（`spark.sql.adaptive.coalescePartitions.enabled` + `advisoryPartitionSizeInBytes`）、动态切换 join 策略（`spark.sql.adaptive.localShuffleReader.enabled` 与 `spark.sql.adaptive.optimizeSkewsInRebalancePartitions`）、倾斜 join 优化（`spark.sql.adaptive.skewJoin.enabled`）
- 能给出验证路径：Spark UI SQL 页看物理计划是否出现 `AdaptiveSparkPlan`、Stage 页看 shuffle 分区数是否被合并/拆分、对比开启前后的执行计划差异（History Server 里查 `Query Execution` 事件）
- 能说出 AQE 生效前提：发生在 **shuffle 边界之后**（依赖 shuffle 输出的 map 端统计），所以窄依赖链路、读源阶段的并行度问题 AQE 管不了
- 能讲清 AQE 与手动调优的取舍：AQE 动态化后很多静态参数（`spark.sql.shuffle.partitions`）不再是"必须手调"，但 AQE 无法处理 groupBy 型倾斜、UDF 阻断、源端分区不均等场景
- 有实践证据意识：同一任务对比开启/关闭 AQE 的耗时与 shuffle 分区数变化，用数据说话
- 知道 `spark.sql.adaptive.coalescePartitions.minPartitionSize`/`maxNumPostShufflePartitions` 边界与 `REBALANCE` 算子

**减分项：**
- 只说"开了 AQE 就自动优化"，不知道它优化的是哪个环节
- 无法从 UI/日志证明 AQE 生效（说不出 AdaptiveSparkPlan 在哪看）
- 不知道 AQE 基于 shuffle map 端统计，第一阶段/读源阶段慢它无能为力
- 把 AQE 当万能药，遇到任何慢都先怀疑 AQE 没生效
- 不知道 AQE 与"手动 hint 指定 join 策略"同时存在时的优先级

**解答：**
先明确 AQE 能干什么：它在每个 **shuffle 边界后**基于 map 端真实输出统计做三类动态决策——① 动态合并分区（默认开启）：shuffle 后输出数据远小于 `spark.sql.shuffle.partitions`（默认 200）时，自动合并为按 `spark.sql.adaptive.advisoryPartitionSizeInBytes`（默认 64MB）估算的较少分区，减少下游 task 数与空转；② 动态切换 join 策略：小表广播阈值内自动用 BroadcastHashJoin、读侧满足条件时用 `localShuffleReader` 去掉多余 shuffle；③ 倾斜 join 拆分（`skewJoin.enabled`）：识别超过 `skewedPartitionFactor`（默认 5）× 中位数、且大于 `skewedPartitionThresholdInBytes`（默认 256MB）的倾斜分区，拆成多个 task 并行处理。验证是否生效：跑完在 **Spark UI → SQL 页**看物理计划是否以 `AdaptiveSparkPlan` 为根节点，且计划内出现 `CoalescedShufflePartitions`/`OptimizedSkewedJoin`；Stage 页对比 shuffle 后分区数与配置分区数是否被改写；更严谨的是到 History Server 的事件日志里搜 `QueryExecution` 的 `physicalPlan` 前后变化。排查"开了还慢"的步骤：① 确认 AQE 相关开关确实生效（`spark.sql.adaptive.enabled=true` 且非 Spark 2.x）；② 看慢在哪个 Stage——若慢在**首个读源 Stage**（Task 数由文件分片决定）或窄依赖阶段，AQE 管不到，该做的是修文件分片/并行度/谓词下推；③ 若慢在 shuffle 后但分区数没变，检查是否 AQE 因 `spark.sql.adaptive.coalescePartitions.minPartitionSize`（默认 1MB）限制认为无需合并，或自定义算子阻断了优化（如 `repartition` 显式指定分区数、UDF、`sortWithinPartitions` 等会让 AQE 跳过）。取舍：AQE 出现后，`spark.sql.shuffle.partitions` 这类静态参数可以不再细抠（仍作为兜底上限），但 AQE 救不了三类场景——groupBy 型倾斜（只做 join 倾斜）、源端读取慢、以及 shuffle 之前已经打满的执行内存；遇到这类慢任务，正确姿势是关掉 AQE 单点对比、回到手动方案（加盐/拆任务/改源端）。实践中的坑：AQE 合并分区是"预估"，若下游算子对分区数敏感（如写文件想控制文件数）仍建议显式 `repartition`，别依赖 AQE 的"自动"；另外 AQE 会重算物理计划，超长 SQL 的 planning 时间会略增。

**延伸考点：** AQE 的动态合并为什么依赖"shuffle map 端统计"，这个统计怎么保证准确性（map 端 partition 提前结束/失败重试）？`spark.sql.shuffle.partitions` 在 AQE 开启后还怎么用？

---

### Q10. Structured Streaming 统计每 10 分钟的用户活跃数，但个别事件延迟了半小时才到，统计结果总不对，怎么改？

**问题：** 你用 Structured Streaming 做实时指标：`groupBy(window(event_time, "10 minutes")) count` 输出到下游。业务反馈某些事件因客户端缓存延迟半小时到达，导致对应窗口统计偏小，且窗口关闭后迟到的数据被丢弃。你如何设计 watermark 与输出策略？

**期望加分项：**
- 能讲清 watermark 语义：`withWatermark("event_time", "30 minutes")` 定义"允许晚到 30 分钟"，窗口何时关闭/状态何时清理由 watermark 决定，而不是固定到点关窗
- 知道 watermark 与窗口的关系：窗口关闭时间 = 窗口末尾 + watermark 延迟，watermark 之前的数据进入窗口聚合，之后到达的视为过期丢弃
- 能区分三种 output mode 对 watermark 推进的影响：`append` 模式要等窗口关闭才输出、`update` 模式持续输出部分聚合、`complete` 模式全量重发——以及 watermark 只在 append/update 下随事件推进（complete 不下推丢弃旧状态）
- 结合场景给方案：活跃数这种可累加指标用 `update`（或 append + 延迟下游）模式，加 watermark 30 分钟，迟到数据自动补算，且状态不会无限膨胀
- 能提 state 存储与清理：`spark.sql.streaming.statefulOperator.checkpointLocation`、TTL 由 watermark 决定，state 过大会拖慢 checkpoint
- 主动考虑边界：多 watermark 时取最小、流表 join 的 watermark、以及 event_time 缺失/异常时间戳（乱序过大）如何处理

**减分项：**
- 认为"加了 watermark 就等 30 分钟才出结果"，混淆"窗口关闭时间"与"数据输出时间"
- 不知道 watermark 推进依赖事件时间，只输入处理时间则 watermark 不前进
- 说不清 append/update/complete 三模式在窗口聚合下的输出时机差异
- 不考虑状态膨胀：无界键聚合（如按 user_id 分组）配大 watermark 会撑爆 state
- 忽略 checkpoint 目录与状态恢复的关系

**解答：**
先纠正一个常见误解：watermark 不是"延迟 30 分钟出结果"，而是定义**事件时间的容忍窗口**。`withWatermark("event_time", "30 minutes")` 表示引擎维护一个不断前进的水位线（当前见过的最大事件时间 − 30 分钟）。窗口在"水位线越过窗口末尾"时才判定过期：`[T, T+10min)` 的窗口要等收到事件时间 ≥ T+10min+30min 的数据才关闭。于是：延迟 30 分钟以内的迟到事件会**并入对应窗口补算**（聚合用 update 模式会修正之前输出）；超过容忍范围的被视为过期丢弃（可另写 `foreachBatch` 落到死信表）。关键设计：① 用 `update` output mode——窗口未关闭前每批输出部分聚合（含修正值），下游是可累加指标时直接 upsert；若下游只接受一次写入（append 且要求"窗口结束后再发"），则要接受延迟：窗口关闭才输出，即"结果晚 30 分钟但完整"；② 注意 `complete` 模式不会利用 watermark 清理状态，无界键场景必须避免；③ 状态膨胀控制：聚合 key 维度越高、watermark 越大，保留的中间状态越多，`update` 模式下每个活跃窗口都要保留 state，必要时用 `mapGroupsWithState` 自定义 TTL；④ checkpoint 目录是状态持久化位置，必须配置到可靠存储（HDFS/S3），否则任务重启丢状态。实践中的坑：输入源（Kafka）必须带事件时间且延迟分布已知，watermark 设太小（如 1 分钟）客户端抖动就丢数据，设太大则窗口关闭晚、下游指标延迟高——根据延迟数据分位数（P95/P99）设值，并配死信监控"过期丢弃量"；同一流 join 时 watermark 取两边最小，别踩"一边没设 watermark 导致状态不清理"的坑；`trigger(ProcessingTime("1 minute"))` 与 watermark 无关，微批间隔只影响吞吐。验证手段：Streaming UI 的 Query 页看 watermark 值、state 大小、被丢弃的记录数（`dropped late records` 指标在 Event Timeline 里）。

**延伸考点：** 如果延迟数据量巨大（延迟 2 小时的事件占比 30%），watermark 方案和"事件分桶先落数仓再修正"方案你怎么选？`append` 模式窗口输出对下游 exactly-once 依赖有什么影响？

---

### Q11. 流作业从 Kafka 读、写数据库，要求不重不丢，你怎么保证，出了重复怎么排查？

**问题：** 实时入湖/入数仓任务：Kafka → Structured Streaming → 写 MySQL/数仓，SLA 要求 exactly-once。某天下游发现一批主键重复数据，你从哪几个环节排查，并把架构改成可证明"端到端不重不丢"的形态？

**期望加分项：**
- 能讲清 Structured Streaming 的容错机制：offset 持久化到 checkpoint（`spark.sql.streaming.checkpointLocation`），重放从 checkpoint 恢复，配合**输出端幂等**才构成端到端 exactly-once
- 知道"至少一次"与"精确一次"的差距在输出端：写库/入湖若不做幂等，同一批数据因任务重启/重复提交会重复写入
- 给出幂等落地方案：① 唯一键 upsert（MySQL 主键冲突 `ON DUPLICATE KEY`/`REPLACE`，按批次+业务键去重）；② 数据湖场景用 Delta `merge`/Iceberg upsert 或按（batch_id, 业务键）去重；③ Kafka 自身 offset 提交的 `kafka.group.id` 与 checkpoint 关系
- 排查链路能讲具体：先看 checkpoint 里 offset 与源端 partition offset 是否一致，再看写库 SQL 是否带唯一约束，再看任务重启日志（`Restarting...`/`Found new offsets`），最后看重复数据的时间窗口是否恰在重启点附近
- 知道 **exactly-once 输出**（`KafkaSink` 用 transactional 写入）与"输出幂等"两种路径，以及流式 join/聚合下"状态重放"的语义
- 能提 `foreachBatch` + 手动事务（同一事务内写库与提交 offset 的"单事务写+记录 offset"模式）这种强一致方案，以及它的代价（吞吐下降、写入串行）

**减分项：**
- 认为"Structured Streaming 天生 exactly-once"，说不出 checkpoint + 幂等输出两个条件
- 排查时只盯代码不看 checkpoint offset 与重启日志
- 写库不带唯一键约束，重复排查时没有"证据锚点"
- 不知道 `append`/`update` 模式与重复输出之间的关系
- 只会说"用幂等"，给不出 upsert/merge 的具体写法与取舍

**解答：**
先讲清机制：Structured Streaming 把 Kafka offset 存在 **checkpoint 目录**（HDFS/S3）的 `offsets` 子目录，任务重启后从上次提交的 offset 继续读——这保证"不丢"；但"不重"取决于输出端：任何一次任务失败重试（task 重试、driver 重启、推测执行）都可能让同一批数据被**重复写一次**，因为 offset 提交与输出写入不是原子操作。所以端到端 exactly-once = **checkpoint 断点续读 + 输出端幂等/事务**。排查重复三步走：① 看 checkpoint 的 offset 进度：`.../checkpoint/offsets/` 与 Kafka 各 partition 实际 offset 对比，确认"读到哪写到哪"；② 看写库逻辑是否幂等：如果写的 SQL 是无条件 `INSERT` 且表没有唯一键，重复 100% 来自重启重放；③ 看重启时间线：重复数据的时间戳集中在任务重启/网络抖动点附近，配合 executor 日志里 `Task ... re-run`、driver 日志的 `Restarting Structured Streaming query` 可锁定。修复方案分层：**方案 A（简单、推荐）**：输出幂等——目标表建唯一键（业务自然键或 `kafka_offset_hash`），写库用 `INSERT ... ON DUPLICATE KEY UPDATE`（MySQL）或 Delta `merge`/`spark.sql.merge`；这样即使重放也只会覆盖相同键，不产生重复行。**方案 B（强一致）**：`foreachBatch` 里把"写数据 + 提交 offset"放进**同一个外部事务**——如写 MySQL 时把 `(batch_id, offset)` 写进同一张 offset 表、或事务内写库，事务提交成功才推进 checkpoint（实际是 driver 侧 `checkpoint` 在 batch 完成后提交，配合"先写业务表+offset 表再 commit"）；代价是写入吞吐受限、复杂度高，通常只在"下游无法去重"时用。**方案 C（数据湖专用）**：Delta/Iceberg 的 upsert（`merge`）天然幂等，配合 checkpoint 即可声明 exactly-once。实践中的坑：checkpoint 目录必须放在持久化存储且**一个 query 一个目录**，误用共享目录会 offset 错乱；升级 Spark 版本时 checkpoint 兼容性（StateStore 格式）可能不向后兼容，需规划状态迁移；写 MySQL 的 `foreachBatch` 里注意连接复用与批次大小，别为幂等引入每行一次 RPC 的低吞吐。

**延伸考点：** checkpoint 里的 StateStore（HDFSBackedStateStoreProvider）损坏时如何恢复（丢弃状态 vs 重建）？`processingTime` 微批的 offset 提交与"窗口聚合状态"在重启时如何协同恢复？

---

### Q12. 同样的 Parquet 表，只查 3 个字段加一个过滤条件，比查全表慢 5 倍，为什么，怎么优化？

**问题：** 一张 500GB 的 Parquet 分区表（30 个字段、按天分区、按 `user_id` 和 `event_time` 排序写入），查询 `SELECT user_id, amount, event_time FROM t WHERE dt='2024-01-01' AND user_id='u123'` 却要扫几十秒。你会检查哪些环节，如何让它秒级返回？

**期望加分项：**
- 明确优化链路与检查点：分区裁剪（`PartitionFilters`）→ 列裁剪（只读 3 列）→ Parquet row group 级数据跳过（min/max 统计）→ 谓词下推（`PushedFilters`）→ 是否命中索引/排序（文件内聚性）
- 能指出最可能的根因：数据写入时未按查询键排序/聚簇，`user_id` 分布在整个表里，谓词过滤只能靠"读完全部 row group 的统计信息 + 全文件扫描"兜底；或谓词下推没生效（如对分区字段的类型写错）
- 给出优化方案：① 校验 `EXPLAIN` 的 `PartitionFilters`/`PushedFilters`/`pruned columns`；② 重排写入：按高频查询键（`user_id`、`event_time`）排序/`bucketBy` 或数据湖的 `ZORDER BY`/`OPTIMIZE`，让同 user 数据聚拢到少量 row group，配合 Parquet min/max 实现文件级/行组级跳过
- 能提 `spark.sql.parquet.filterPushdown`、`spark.sql.parquet.recordLevelFilter.enabled`（旧版）以及分区内 `dynamic partition pruning` 与广播 join 时的自动裁剪
- 有量化验证：对比优化前后"扫描的字节数"（UI 上 Input Size/Records Read）与耗时，而不是只看总时长
- 主动考虑边界：点查高频场景是否该换存储（KV/列存索引如 Doris/ClickHouse/iceberg + 索引），Parquet 是"扫描友好"格式，单点查询不是它的强项

**减分项：**
- 直接下结论"加索引"却不看 Spark 对 Parquet 是否支持行级索引（parquet 无 B-tree 索引，靠统计跳过）
- 不看执行计划就说"谓词下推没生效"，没有证据
- 不知道 Parquet 跳过机制依赖数据在文件内按值聚簇（排序写入），随机分布时跳过失效
- 优化只调参数（如开 filterPushdown）却不动写入布局
- 忽略列裁剪，查 30 个字段 vs 3 个字段读盘量差一个数量级却没意识到

**解答：**
按"读少 + 跳多"两条线排查。第一步看执行计划：`EXPLAIN` 的 `Scan parquet` 节点确认 `PartitionFilters=[dt=...]`（分区裁剪是否命中，若为空说明 `dt` 过滤没下推，常见原因是把字符串分区列和日期类型比较出错）、`PushedFilters=[user_id=...]`（谓词是否到文件层）、以及只 scan 了 3 列（列裁剪）。第二步理解 Parquet 的跳过机制：Parquet 文件按 **row group**（默认 128MB 级别）存储，每个 row group 记录各列的 **min/max/空值统计**；Spark 读文件时先用统计信息做"文件级 + row group 级跳过"，只有可能命中的 row group 才会真正解压读列数据。这套机制生效的前提是**数据在文件内按过滤键聚簇**——如果 `user_id` 在文件里乱序分散，几乎每个 row group 的 min/max 都覆盖 `u123`，跳过失效，退化为全文件扫列。你表的现状（按天分区、未按 user 排序）正是后者。第三步优化（按性价比排）：① 写入侧重排——高频查询键排序写：写表时 `df.sortWithinPartitions("user_id", "event_time")`（或 `repartitionByRange`）让同 user 数据聚拢，重写后同分区文件数量不变但每个文件内聚簇；② 若用数据湖，`OPTIMIZE t ZORDER BY (user_id, event_time)` 一次完成 compaction + 布局优化，Spark 3.x 支持 ZORDER 排序统计；③ 更极致的点查（毫秒级）考虑 `bucketBy(16, "user_id")` + 分桶表：按 user_id 哈希分桶，查询可直接定位到桶文件，配合谓词下推实现"文件数级裁剪"，代价是写入要 shuffle、桶数要按数据量选好；④ 校验参数：`spark.sql.parquet.filterPushdown=true`（默认开），必要时开 `spark.sql.files.maxPartitionBytes` 减小分区粒度增加跳过粒度。第四步验证：UI 上对比优化前后 `Input Size`（扫了多少字节）与耗时，`SELECT count(*)` 走全扫做基准。实践中的坑：ZORDER 对多列有衰减（一般 ≤4 列有效），高频过滤键放最前；排序写入会增加写任务耗时与 shuffle，只对"读多写少"的热表做；Parquet 的统计跳过对 `LIKE '%x%'`、UDF 条件无效（下推不了）；跨分区点查注意分区数过多时 partition pruning 本身也有目录列开销。若业务是毫秒级点查，应直说 Parquet 不是最优解，考虑引入 OLAP 引擎做二级存储。

**延伸考点：** `bucketBy` 与 `sortBy`/`clusterBy` 的物理布局差异是什么？ZORDER 与"多列联合排序"的查询收益差异（等值 vs 范围查询）？

---

### Q13. 用 Spark JDBC 读写 MySQL，读的时候慢、写的时候把库打挂，怎么改？

**问题：** 一个 ETL 任务从 MySQL 读一张 5000 万行的业务表，再写结果回另一张 MySQL 表。当前实现很慢，且写库时数据库 CPU 打满、连接数报警。你会如何优化读写两端的并行度与行为？

**期望加分项：**
- 读端能讲清分区并行原理：`numPartitions` + `column`/`lowerBound`/`upperBound` 决定如何把查询拆成多个分区并行读（每分区一条带 `WHERE (column >= x AND column < y)` 的 SQL）；没指定则单分区串行读，`partitionColumn` 必须是有序数字列
- 知道读端坑：`lowerBound/upperBound` 只是拆分依据不是过滤条件；分区键选择不当（倾斜/非单调）会导致热点分区；`fetchsize`（默认 0=流式全部拉回）要设置如 1000~5000 减少逐行 RPC
- 写端能讲清风险与对策：默认 `write` 模式是 `INSERT`（DataFrameWriter 的 `SaveMode.Append` 且每 executor 一条连接串行插入），要控制 `batchsize`（如 1000）、开启 `rewriteBatchedStatements`（MySQL JDBC）让驱动真正批量执行
- 知道并发写要用 `repartition` 控制写并行度（如 8~16 个分区），而不是让 task 数无限放大；配合 `spark.executor.cores` 与连接池避免连接风暴
- 能提 `truncate` 覆盖写（`mode("overwrite")` + `truncate=true`）避免全表 DROP+INSERT 的资源峰值，以及 `SaveMode` 语义
- 面对大表给架构级建议：一次性全量迁移考虑 JDBC 之外的工具（如专门导数工具/CDC + 数仓），Spark 更适合"按条件抽数"场景

**减分项：**
- 读端只给 `numPartitions` 不给分区列，或不知道分区列要求（数字/有序、单调）
- 不知道 `fetchsize` 与流式读的关系，逐行拉 5000 万行必然慢
- 写端无脑加并行度/多 executor，反而把数据库连接打满
- 不知道 `rewriteBatchedStatements` 对批量写的重要性
- 不考虑目标库容量与写入窗口，把 OLTP 库当数仓写

**解答：**
**读端**：核心是"分区并行 + 批量拉取"。配置示例：`spark.read.format("jdbc").option("url",...).option("dbtable", "(SELECT id, name, ts FROM t WHERE ts >= ?) sub").option("partitionColumn", "id").option("lowerBound", 1).option("upperBound", 50000000).option("numPartitions", 20).option("fetchsize", 5000).load()`。要点：① `partitionColumn` 必须是数值型、单调有序的列，Spark 按 `(upperBound-lowerBound)/numPartitions` 生成 20 条带 `id` 范围条件的并行查询；若主键分布不均（如大量删除导致空洞、id 不连续），分区数据量会不均，此时改用 `predicates` 数组手写分区条件更可控；② `fetchsize` 默认 0 意味着 JDBC 用流式读（一次一行返回），改成 5000 走批量 fetch，能显著减少 RPC；③ 注意 `dbtable` 里如果传子查询，需要外层别名，且子查询的过滤条件可以下推减少传输；④ 读大表时尽量只 select 需要的列（列裁剪），别 `select *`。**写端**：风险是"task 数 × 每 task 串行 insert"把库打挂。正确姿势：① 控制并行度——写前 `df.repartition(8)`（或按库表写入能力设 4~16），配合 `spark.executor.cores=1` 时每 executor 1 task，全局并发写连接 = 分区数；② 批量——`option("batchsize", 1000)`，并强烈建议在 JDBC URL 加 `rewriteBatchedStatements=true`（MySQL），否则驱动默认不真正合并 SQL，批量形同虚设；③ 覆盖写场景用 `mode("overwrite").option("truncate", "true")` 走 TRUNCATE+INSERT 而非 DROP+CREATE，降低 DDL 开销；④ 写前评估目标表：若有唯一键冲突，明确业务语义用 `INSERT IGNORE`/`ON DUPLICATE KEY` 得靠 `foreachBatch` 手写（Spark JDBC 写不支持 upsert 语义），或先落临时表再在库内 merge。实践中的坑：分区列与查询条件冲突（下推谓词把分区列过滤后，某些分区永远为空也要建连接，白白产生 20 条空查询）；`numPartitions` 过大时每个分区各建一条连接，连接数 = 分区数 × 批次，注意 MySQL `max_connections`；数据库是 OLTP 时，写入窗口/限流（QPS 上限）要比吞吐更重要——必要时 `foreachBatch` 里自己实现限流与重试。架构提醒：5000 万行全量同步这种需求，优先考虑用专用工具/CDC 走数据管道，Spark JDBC 适合增量抽数与加工后回写，别把 OLTP 库当数仓做大全量。

**延伸考点：** 分区列用时间戳（如 `created_at`）而不是自增 id 时，分区拆分有什么坑（非均匀、类型转换）？写库怎么实现"不重复"（幂等键）且不依赖数据库唯一约束？

---

### Q14. 大表 JOIN 大表，一个 key 占了一半数据，整个 join 卡死，你怎么救？

**问题：** 两张各 2TB 的事实表按用户维度 join，其中"默认用户/空用户"这类 key 占数据量一半，SortMergeJoin 的某一个 reduce 分区要处理整张表一半的数据，任务跑不完。你给出排查路径与完整解法（含 SQL/代码思路）。

**期望加分项：**
- 先确认倾斜位置：UI Stage 页看 shuffle read 最大的分区与 key 分布统计（`group by key count`），区分"空值/默认值这类少数超大 key"与"大量普通 key"
- 能给出分层解法：① 空值/脏 key 先处理——业务语义上能否过滤、置默认、或单独 join；② 少数热点 key 加盐——小表侧对该 key 复制 N 份（膨胀 N 倍）、大表侧加随机后缀拆分，join 后去盐聚合；③ 全量倾斜用随机加盐（两表都加同盐但只打散不复制，丢失 join 准确性——需解释为什么只能用于"聚合型 join"而非明细 join）
- 知道加盐的代价与边界：小表膨胀 N 倍的内存/shuffle 放大、join 输出可能重复（需二次聚合去重）、非等值/范围 join 无法加盐
- 能对比 AQE `skewJoin` 与手动加盐：AQE 自动拆分倾斜分区适合等值 join 且倾斜分区可检测；手动加盐适合 AQE 救不了的场景（如倾斜在 map 端、小表无法广播）
- 有替代思路：小表能广播就广播（哪怕超过阈值强制广播）、`map-side join`/`salting` 分两段 join（热点 key 单独一段）
- 验证闭环：改完对比 UI 上 max/median shuffle read 比值与总耗时

**减分项：**
- 只会说"加盐"，但说不清加盐在两张表上分别怎么加、结果怎么还原
- 不知道空值/默认值是 join 倾斜最高频根因，也不讨论"这个 key 该不该出现在明细 join 结果里"（语义问题）
- 加盐后不处理重复（join 命中多份），产生错误数据
- 不知道 AQE skewJoin 的存在或适用范围
- 不看执行计划就动手，改完没对比数据，说不清是否真的解决

**解答：**
排查：UI 找到慢的 Stage，看 shuffle read 的 max/median 比值；再跑一个临时 `select join_key, count(*) group by join_key order by 2 desc limit 10`（大表侧抽样即可）确认热点 key 形态。分三类处理：**① 空值/默认值**（最高频）：先问业务——明细 join 结果里空用户是否应该保留？若该 key 无实际意义，`filter join_key is not null` 或统一置 `'unknown'` 后按"是否与维表匹配"决定 inner/left 语义；若必须保留且维表有对应行，把它当一个独立小 join 单独做。**② 少数热点 key**（如 top10 key 占 30%）：手动加盐——大表侧把热点 key 拆成 `key_0..key_N-1`（随机打散），维表侧把热点 key 复制 N 份（`key_i` 各一份），然后 join，结果按原 key 聚合还原。伪代码思路：`left.withColumn("join_key", when(hot, concat(join_key, "_", rand()*N))).join(right.withColumn("join_key", explode_salt(hot_keys, N)), "join_key").groupBy(原始key).agg(...)`——小表膨胀 N 倍是内存代价，N 选 8~32 按热点占比折中。**③ 全局倾斜**（key 普遍不均匀）：随机加盐但**只能用于聚合型 join**——两边都拼随机后缀再 join，会人为切断真实匹配关系（同一 key 的两行可能被拆到不同盐值），因此仅适用于"先 join 后聚合"且聚合函数对乱序不敏感的场景（如 count distinct 不行，sum 可以）；输出需再按原始 key 聚合一次。**框架层**：Spark 3.x 开 `spark.sql.adaptive.skewJoin.enabled`（配合 `skewedPartitionFactor=5`、`skewedPartitionThresholdInBytes=256MB`），AQE 检测到某个 reduce 分区超过阈值时自动把它拆成多个 task 并行处理——实现机制是"把大分区的数据复制多份、小分区表广播多份"（本质也是加盐，但由框架自动做），**适合小表可广播或分区不均衡的等值 join**；它要求 join 是 shuffle 型且能在运行时感知 map 端统计，groupBy 倾斜和"map 端已倾斜"（如单文件超大）它管不了。落地顺序建议：先开 AQE skewJoin 看效果（零成本），再针对 top key 手动加盐，最后才考虑拆任务/改语义。验证：改完对比 UI max shuffle read 与整任务耗时，同时用结果集去重校验（加盐 join 后 `groupBy` 还原，确保行数 = 正确 join 结果，防止重复）。坑：加盐粒度太大导致维表膨胀内存 OOM；热点 key 识别阈值拍脑袋（建议按 `count / 总量 > 1/N` 界定）；join 输出被下游当明细用时，加盐还原逻辑必须不漏不重。

**延伸考点：** AQE 的 skewJoin 内部实现（把大分区拆成多份 + 广播）在"小表几十 GB、热点 key 占 60%"时为什么可能失效？范围 join（非等值）倾斜怎么处理？

---

### Q15. 团队想"批流一体"：同一套口径，批任务和流任务各写一套，经常对不上数，你怎么设计？

**问题：** 公司里同一个指标（如"当日 GMV"）有批任务（T+1 全量算）和流任务（实时算），两套代码、两套口径，经常对不上账。你如何设计一套"批流一体"方案让两边的口径、代码、结果尽量收敛？

**期望加分项：**
- 能先讲清批流一体的本质目标：**同一套口径/同一份代码/同一个存储**，而不是"一个引擎跑所有"；给出 Lambda vs Kappa 的取舍，以及国内主流落地是"批为主体 + 流做增量预聚合 + 批流对账"的组合形态
- 能讲清关键做法：① 统一口径——把指标定义下沉为"可复用的计算函数/SQL 模板"，批流共用；② 统一存储——基于 Delta/Iceberg/Hudi 做"流写批读"或"批写流读"，流写小文件 + 批 compaction；③ 用 Structured Streaming 的 `foreachBatch` 复用 Spark DataFrame 批处理 API，同一套逻辑跑批和流
- 知道流侧特有的坑：状态/窗口语义与批的差异（watermark、乱序）、幂等与 exactly-once 的代价、小文件问题在流写场景的放大
- 有对账意识：实时数与 T+1 数的差异来源分析（迟到数据、截断时间、幂等覆盖），设计"准实时 + 定时修正"的收敛机制
- 能谈数据质量与血缘：指标口径元数据化（如指标字典），跨批流作业血缘统一在 Catalog
- 能承认边界：实时性要求秒级且逻辑极复杂时，Kappa 可能反而更贵，选型要回到业务 SLA

**减分项：**
- 只会说"用 Flink/Spark 一个引擎统一"，说不出批流语义差异在哪
- 不提口径统一（指标定义、时区、截断边界），只说技术栈
- 不知道 `foreachBatch` 为什么能复用批逻辑（它就是"每批跑一次 batch query"）
- 不考虑对账与修正机制，声称"完全一致"但讲不出差异来源
- 忽略流写产生的文件碎片与 compaction 治理

**解答：**
先定义清楚：批流一体的核心收益是**口径收敛 + 代码复用 + 存储统一**，不是必须"一个引擎跑到底"。落地形态建议分三层。**第一层口径层**：把每个指标固化为一个"计算模板"（SQL 或 DataFrame 函数），参数是时间范围与粒度，批流都调用同一模板——这是对账一致的前提；同时统一时间边界（用事件时间还是处理时间、自然日怎么截断、时区），把边界写进模板而不是散落各任务。**第二层执行层**：Spark 生态里用 Structured Streaming 的 **`foreachBatch`**——它把每个微批作为一个 batch DataFrame 执行，`foreachBatch { df => 批处理逻辑(df) }`，同一份"批逻辑"既能被每日全量批任务调用，也能被实时微批调用；配合 `trigger(ProcessingTime("5 minutes"))` 控制延迟。注意 `foreachBatch` 里不能再用 `awaitTermination` 等流式 API，且对"窗口聚合状态"类逻辑（groupBy window）需要额外设计——窗口聚合建议放在流侧保留 watermark 语义，批侧用同样的 SQL 重算即可（两边都基于统一模板）。**第三层存储层**：推荐数据湖（Delta/Iceberg）做"流写批读 + 批补偿"：流任务 `merge`/upsert 写增量到湖表（天然幂等），批任务对同一张表做全量重算或 compaction，双方读同一份数据，天然对齐；小文件治理交给湖的 `OPTIMIZE`/compaction 后台任务。**对账与修正**：任何批流一体都逃不过"实时数先出、T+1 数修正"的收敛过程——设计每日对账任务：比对流侧累计值与批侧重算值的差异（迟到数据、幂等覆盖、截断边界都会造成差异），差异超过阈值告警，并在 T+1 用批结果覆盖修正，保证最终口径一致。实践中的坑：① 别在流侧实现批侧没有的"自定义逻辑"（如流侧为了性能提前做二次聚合），口径会悄悄分叉；② 流写小文件要设 `spark.sql.streaming.schemaInference` 与合理的分区策略，配合定频 compaction；③ 指标字典/血缘：把模板注册到 Catalog 或元数据中心，批流作业血缘都指向同一指标定义，后续改口径才能一处修改处处生效。最后诚实说明边界：秒级 SLA + 复杂状态逻辑时，Kappa 全流方案的成本可能超过收益，选型回到业务需求，"批流一体"是手段不是教条。

**延伸考点：** `foreachBatch` 里做窗口聚合和直接在流上做窗口聚合，语义差别是什么（状态管理 vs 每批独立）？实时值与 T+1 值对不上时，你如何确定差异是"迟到数据"还是"逻辑 bug"？

---

### Q16. 把 Spark 任务从 YARN 迁到 K8s，或反过来，你会怎么评估和做资源规划？

**问题：** 公司要把批量 Spark 作业从 YARN 迁移到 Kubernetes（或反向），你作为数据平台负责人，怎么评估两种调度器的差异，迁移时资源配置（driver/executor 的内存、核数、本地性）和稳定性要重点注意什么？

**期望加分项：**
- 能讲清两种模式的本质差异：YARN（client/cluster 模式、AM 管理、队列调度、与 HDFS 同机架本地性天然好）vs K8s（driver/executor 都是 Pod、`spark.kubernetes.*` 配置、无队列概念、资源由调度器分配、本地性依赖 `nodeSelector`/污点/亲和性）
- 知道迁移的核心改动点：`spark-submit` 的 `--master k8s://...` 与 `--master yarn`、镜像构建（`spark.kubernetes.container.image`）、driver/executor 的资源声明方式（`spark.driver.memory`/`spark.executor.memory` 依然有效，但 overhead 与 Pod limits 的关系要重新配）
- 能谈 K8s 特有坑：executor Pod 被 OOMKilled（`exit 137`）时 Spark 重试策略、Pod 抢占导致 task 全灭（`spark.kubernetes.executor.deleteOnTermination`、`spark.dynamicAllocation` 与 Pod 删除的配合）、存储挂载（本地盘/网络盘）、镜像仓库网络
- 能讲 YARN 侧优势：与 HDFS 数据本地性（`spark.locality`）、队列容量规划（公平/容量调度）、日志聚合（`yarn logs`）成熟；K8s 优势：弹性、多租户隔离（namespace/配额）、与微服务运维栈统一
- 动态资源分配在两种模式下的差异：YARN 的 `dynamicAllocation.shuffleTracking` 成熟；K8s 下 executor 启停慢（Pod 调度 + 镜像拉取），要评估启动开销，常关闭或加大 executor 复用
- 迁移验证方法：先跑灰度任务对比耗时/成功率/资源利用率，用 `spark.kubernetes.report.interval` 采集 executor 指标，评估 pod 启动耗时占比

**减分项：**
- 不知道 YARN client 与 cluster 模式的区别（driver 跑在哪、日志去哪看）
- 迁移只改 `--master`，不重新评估内存 overhead 与 Pod limits 的关系
- 忽略数据本地性：K8s 集群与 HDFS 不在同一网络时 shuffle/读数据全走网络，性能掉档
- 不知道 K8s 下 executor 被驱逐/OOMKilled 的语义（非 graceful，任务直接失败重试）
- 不评估 Pod 启动耗时对小任务（分钟级）的影响

**解答：**
先明确评估维度：调度模型、资源隔离、数据本地性、运维生态、弹性。**YARN 侧**：`--master yarn` 下分 client/cluster 模式——cluster 模式 driver 跑在 AM container 里（提交后本地无进程，日志在 `yarn logs -applicationId`），client 模式 driver 跑在提交节点（适合交互式调试，但提交节点宕机任务即挂）；队列调度（fair/capacity）成熟，适合"多团队共集群、按队列配额分配"的场景；与 HDFS 天然同机架，读数据本地性好。**K8s 侧**：driver 是 `spark-driver-*` Pod，executor 是 `spark-exec-*` Pod，用 `spark.kubernetes.driver.podTemplateFile`/executor template 精细控制；资源声明仍用 `spark.executor.memory`（堆）+ `spark.executor.memoryOverhead`（堆外），但要确保 **Pod 的 limits 与两者之和匹配**——否则 executor 实际可用的 direct memory 不够或 YARN 式"超限被杀"变成 K8s 的 OOMKilled（`exit code 137`），且 executor 被 OOMKilled 是非优雅终止，任务直接失败、靠 Spark 的 task 重试兜底，重试风暴时要看 driver 日志确认原因。**数据本地性**是最容易踩的坑：K8s 集群若与 HDFS 异机架/跨机房，`spark.locality.wait` 策略失效，所有读都走网络，吞吐掉 30%+——评估时先做"同机房部署 + 亲和性"（`nodeSelector` 把 executor 调度到靠近 DataNode 的节点），或直接用云对象存储（S3/GCS）配合 `spark.hadoop.fs.s3a.*`，本地性退化为"远程读 vs 对象存储"的取舍。**资源规划**：executor 内存沿用"堆 + overhead"模型，核数建议 2~5（超线程下 `spark.executor.cores` 别给满，留 OS/Kryo buffer 余量）；小任务（分钟级）在 K8s 下要评估 Pod 启动（镜像拉取+调度）的 30~60 秒开销是否可接受，通常 `dynamicAllocation` 关掉或设最小 executor 数、用常驻复用；YARN 下则可以利用成熟 shuffle tracking 开动态分配。**稳定性**：K8s 下要配 `spark.kubernetes.executor.deleteOnTermination=true`（清理失败残留 Pod）、driver 用 `RestartPolicy: OnFailure`、executor 驱逐（eviction）事件监控；YARN 下注意队列容量与 `spark.yarn.maxAppAttempts`。最后迁移节奏：先灰度 10% 任务双跑对比耗时/成功率/成本（K8s 按 Pod 算账 vs YARN 按队列），确认指标一致再全量切。

**延伸考点：** K8s 下 executor Pod 频繁 OOMKilled，但 Spark UI 显示堆内使用率不高，怎么排查（overhead 配置、direct buffer、节点内存 pressure）？K8s 与 YARN 的日志/指标采集链路怎么统一（如对接 Prometheus + Grafana vs ResourceManager 指标）？

---

### Q17. 新任务上线，executor 数、每 executor 核数和内存、shuffle 分区数你按什么公式推？

**问题：** 一个新数据量任务（输入 5TB、无倾斜、Shuffle 中间量约 2TB）要上线，你需要给出初始资源配置：executor 数量、每 executor 核数/内存、`spark.sql.shuffle.partitions`，并说明推导依据与验证方法。你会怎么设计？

**期望加分项：**
- 能给出推导链路：先定"单 task 处理量"与"每 executor 并发 task 数"，再反推 executor 数与内存，而不是拍脑袋
- 有量化意识：输入 5TB 按 HDFS 块 128MB ≈ 4 万 个初始 task；shuffle 输出 2TB 按单分区 128~256MB 反推 shuffle partitions ≈ 8000~16000（或交给 AQE 自动合并后设一个上界）
- 能算 executor 内存：堆内 = 单 task 内存需求（如 2~4GB）× 每 executor 并发 task（cores）+ 缓存余量 + overhead；给出"总并发度 × 单 task 内存"与集群总资源的平衡
- 知道"核数不要给满"（默认 1 core 或 2~5 core 之间权衡：core 多则 task 并发高但 GC/内存争用加剧），以及 JVM 大堆（>32GB）的 GC 长暂停问题
- 能结合 shuffle 特点：shuffle read 侧内存峰值 ≈ 分区大小 × 并发 task 数，据此校验 executor 内存是否够
- 有验证闭环：先按推导值跑，看 UI 的 Task 数/时长/GC 时间/资源利用率（`%CPU`、executor 平均使用率）再迭代调参
- 提 AQE：开启后 `spark.sql.shuffle.partitions` 只需给一个上界（如默认 200 偏小可调到 4096），实际分区由 `advisoryPartitionSizeInBytes` 收敛

**减分项：**
- 只会说"看集群剩多少资源就分多少"，没有从单 task 数据量反推的思路
- executor 内存只报总量，说不出"每 task 需要多少内存"的依据
- 不知道 shuffle partitions 与"shuffle 输出总量 ÷ 目标单分区大小"的关系，一律 200
- 核数给 8/16 却不谈内存争用与 GC
- 上完线不看 UI 指标迭代，一次配完"听天由命"

**解答：**
推导顺序：**先定 task 粒度 → 再定每 executor 并发 → 反推 executor 数与内存 → shuffle 分区数**。第一步 task 粒度：读侧初始 task 数 = 输入 5TB ÷ 128MB ≈ 4 万个（可由 `spark.sql.files.maxPartitionBytes` 调整）；shuffle 后分区数按"shuffle 输出 2TB ÷ 目标分区 128~256MB"≈ 8000~16000。第二步每 executor 并发：设 `spark.executor.cores=4`（2~5 是常见折中，再多并发 task 间内存争用与 GC 加剧），即每 executor 同时跑 4 个 task。第三步 executor 数与内存：单 task 内存需求按"该 task 处理的最大数据集"估——shuffle read 侧单分区 256MB + 聚合/sort 开销按 2~4 倍内存算，单 task 预留 2~4GB 堆内；则每 executor 堆内 ≈ 4 cores × 2~4GB = 8~16GB，加 1~2GB 缓存余量与 10% overhead（≥384MB），即 `spark.executor.memory=12~18g`、`memoryOverhead=2g`。总并发度 = 4 万 /（期望每 task 2~3 分钟）≈ 同时 ~600 task → executor 数 ≈ 600/4 = 150 个（若集群没那么大，则拉长单 task 数据量、用更少 executor 接受更长耗时——这是资源与 SLA 的取舍，要明确说出来）。第四步 shuffle 分区：开 AQE（`spark.sql.adaptive.enabled=true`）后，`spark.sql.shuffle.partitions` 给上界即可（如 4096~8192），实际分区由 `advisoryPartitionSizeInBytes`（默认 64MB）自动收敛，避免"200 个分区每个 10GB 或 16000 个分区每个 128KB"两种极端。验证：跑起来后看 UI——① 每个 Stage 的 Task 平均时长是否在 1~5 分钟（太短说明分区过细、调度开销占比高；太长说明分区过粗或倾斜）；② Executors 页 GC 时间占比（>20% 说明堆小了或缓存多了）；③ 集群资源利用率（`%CPU`/内存水位）是否接近目标；④ 对比预期耗时与实测，按"最慢 Stage"迭代调参（加分区数、加 executor 内存、压缩）。实践中的坑：大堆 executor（`spark.executor.memory > 32g`）GC 暂停时间可达秒级，宁多小 executor 少大 executor；`spark.executor.cores=1` 虽能最大化内存隔离（Kryo/IO 独占）但 executor 数翻倍、元数据与心跳开销上升；不要在无 AQE 的老版本上把 shuffle partitions 拍成 100000，宁可按数据量×1.5~2 倍粗配再实测收敛。整体心法：**先粗后细，用 UI 数据迭代**，任何公式都是初始值，线上数据才是最终裁判。

**延伸考点：** 集群总资源固定（如 200 executor × 16GB），你是优先保"并发度"还是保"单 task 内存"，依据是什么？开启 `dynamicAllocation` 后上述静态推导还要不要做？

---

### Q18. 一个每天稳定跑 2 小时的任务，某天凌晨突然失败，重启后又能跑，你怎么定位和加固？

**问题：** 一个跑了半年的日批任务昨天突然失败（executor 日志显示 fetch 失败/内存问题/个别 task 重试多次），今天重跑又成功了。这类"偶发失败"最考验人，你会怎么系统性定位根因并让任务真正变稳？

**期望加分项：**
- 定位链路清晰：先确认失败 Stage 与 Task 数 → 看失败 Task 的错误类型（shuffle fetch 失败 / OOM / 磁盘满 / 节点故障 / 推测执行问题）→ 结合重试与失败率判断是"单点故障"还是"系统性问题"
- 能区分失败根因类别：① 资源瞬时抖动（YARN/K8s 节点驱逐、磁盘 transient）、② 数据变化（上游某天数据量/分布异常触发倾斜或超内存）、③ 慢节点/坏盘（task 反复失败后 stage 重试）、④ 代码/参数临界（内存余量太小、超时阈值太紧）
- 给加固方案：`spark.task.maxFailures`/`spark.yarn.maxAppAttempts` 重试策略、`spark.speculation` 开启、失败后自动重跑（workflow 级重试 + 幂等写保证重跑安全）、checkpoint 缩短恢复时间
- 有"从失败到改进"的闭环：每次偶发失败都沉淀为 checklists/自动化诊断（如失败自动采集 driver/executor 日志 + 执行计划 + 输入数据分布快照）
- 能意识到"重跑成功 ≠ 问题解决"：偶发失败往往是隐患信号（资源余量、数据质量、代码边界），要用监控与压测验证
- 知道幂等重跑的前提：写入侧要支持重跑不产生重复/脏数据（覆盖写/唯一键）

**减分项：**
- 重跑成功就说"没事了"，不做根因分析
- 只加"重试次数"参数，不区分失败类型（OOM 重试 10 次也不会成功，反而拖长故障时间）
- 不知道 shuffle fetch failed 的常见诱因（executor 被回收/节点宕机导致 shuffle 文件丢失）与 `spark.shuffle.io.retryWait`/`maxRetries` 的关系
- 不检查失败当天的数据量/上游表变更
- 没有重跑幂等设计，手动重跑靠运气

**解答：**
偶发失败按"先分类、再定位、后加固"处理。**第一步分类**：从 History Server/Spark UI 找到失败的那次运行，看失败 Stage、失败 Task 数与错误摘要：① 少量 task `FetchFailed`（shuffle fetch failed）——通常是某个 executor 因节点故障/被 YARN 回收而消失，导致它的 shuffle 中间文件不可读，Spark 会重算对应 map 端（自动 stage 重试），若重试后成功即属"节点级抖动"；② 全部 task 同时失败（如 OOM、`SparkException` 在 driver 侧）——大概率是参数/数据问题；③ 单 task 反复失败后整个 stage 失败——要么该 task 数据倾斜/超内存，要么所在节点坏盘（`DiskStore`/`ShuffleBlockManager` 报 IOException）。**第二步定位根因**：看失败当天与平时的差异——上游表数据量/分区数变化（某天多了一张异常大分区触发内存溢出）、集群变更（节点下线、磁盘容量告警）、任务自身参数临界（`spark.task.maxFailures=4` 恰好被 5 次抖动击穿）。重点确认**是不是 OOM 类失败**：OOM 重试 100 次也不会成功，此时重试参数无效，必须改内存/数据，别把 OOM 当"偶发"处理。**第三步加固**（按成本排序）：① 重试策略——task 级 `spark.task.maxFailures`（默认 4）、shuffle fetch 重试 `spark.shuffle.io.maxRetries`/`retryWait`（默认 3 次/5 秒，网络抖动场景调大）、应用级 `spark.yarn.maxAppAttempts`（YARN，driver 重启会丢内存态，配合 checkpoint 状态恢复）；② 推测执行 `spark.speculation=true`（仅对无状态/幂等任务，拖慢任务时能自动换机重跑）；③ 编排层重试——在 Airflow/调度器里对"幂等任务"配失败自动重跑，前提是**写入必须幂等**（覆盖写、分区级 overwrite、唯一键 upsert），否则重跑=双份数据，这是最容易漏的坑；④ 数据质量护栏——对上游表做"数据量/分区数/最大值"的当日波动告警，数据异常在任务跑之前就暴露。**第四步闭环**：把每次偶发失败做成"事后复盘清单"——采集失败 job 的 driver/executor 日志、执行计划、输入数据摘要，沉淀成自动化诊断脚本；对高频偶发（如每周一次 fetch failed）要主动排查节点健康与网络，而不是无限加重试。实践心法：**偶发失败 = 系统还没修好的必然失败**，重跑成功只是运气窗口，根因可能随时复发。

**延伸考点：** `FetchFailed` 触发 stage 重算时，Spark 如何决定重算哪些分区（shuffle 文件丢失的 block 粒度）？driver 侧 OOM 与 executor 侧 OOM 在"任务重试能否成功"上有什么本质区别？

---

### Q19. 同一段 SQL，数据量翻 3 倍后从 1 小时变成 6 小时，你怎么定位瓶颈在哪？

**问题：** 一个跑了一年的 Spark SQL 任务，上游数据量最近涨了 3 倍，任务耗时从 1 小时涨到 6 小时（远超线性）。你如何系统定位是哪个环节恶化，并给出针对性优化？

**期望加分项：**
- 先建立"耗时分布"视图：Spark UI 按 Stage 看各阶段耗时与输入量，找出**被放大的 Stage**（耗时占比从 20% 涨到 70% 的那个），而不是全任务瞎调
- 能判断"超线性恶化"的典型根因：① shuffle 数据量超线性增长（join 笛卡尔化/膨胀）、② 数据倾斜出现/加剧（新增了热点 key）、③ 内存压力导致 spill 剧增（shuffle/聚合落盘，磁盘 IO 成为瓶颈）、④ 单 task 数据超内存触发 GC 风暴、⑤ 小文件/分区数暴涨导致 task 数爆炸
- 会用关键证据定位：对比"现在 vs 三个月前"同一任务的 Stage 耗时、Shuffle Write/Read 量、spill 大小、GC 时间（History Server 里可对比历史运行）；用 `max shuffle read / median` 看倾斜是否出现
- 优化按杠杆排序：先修倾斜/膨胀（数据层面，收益最大），再调并行度与内存（参数层面），最后考虑改算法/换引擎
- 能说清"超线性"的量化验证：如果纯数据量线性放大，2~3 倍耗时可接受；6 倍说明存在二次项（膨胀 join、重复计算、GC 放大）
- 有止损手段：任务超时前的快速诊断脚本（自动抓最慢 stage 与 top task）

**减分项：**
- 不看 Stage 分布就笼统说"加资源"
- 把"数据量涨了"当最终结论，不继续问"为什么是 6 倍而不是 3 倍"
- 忽略 shuffle 膨胀（join 后行数放大）与倾斜这两个超线性主因
- 不会用 History Server 对比历史运行，只凭印象
- 优化一步到位，没有"改一处验证一处"的迭代

**解答：**
关键认知：**数据量涨 3 倍，耗时涨 6 倍，说明存在超线性放大项，一定不是"均匀变慢"**。定位按四步：**第一步 Stage 分解**：History Server 打开本次运行（与历史运行并排对比），按耗时排序 Stage，找出从"1 小时里占 20 分钟"变成"6 小时里占 4 小时"的 Stage——它就是要优化的主战场。**第二步看放大信号**：针对嫌疑 Stage 对比三个指标——① Shuffle Write/Read 量：如果 read 量是历史 5~6 倍，说明 shuffle 阶段出现膨胀（join 输出行数放大、或广播退化为 shuffle join、或新增了导致笛卡尔展开的关联条件）；② max vs median shuffle read：比值从 1.x 涨到几十，说明**新增倾斜**（上游某字段分布变化，如新活动引入了热点用户）；③ Spill 大小与 GC 时间：shuffle/聚合 spill 到磁盘的次数暴涨，说明内存被击穿——`spark.executor.memory` 相对"单 task 处理量"不够了，或并发 task 数 × 单 task 内存超堆。**第三步定位根因**：按概率排序——join 膨胀（对 join 后的行数做 `count` 对比预期）> 倾斜加剧（`group by key` topN 分布）> 内存击穿 spill > 源端小文件/task 爆炸（分区数 × 文件数暴涨导致 task 数几万、调度开销吞掉算力）。**第四步优化**（按杠杆排序）：① 数据层面（收益最大）：修 join 膨胀（拆条件、提前过滤、避免笛卡尔）、修倾斜（加盐/广播/拆 key，见 Q2/Q14）；② 参数层面：对"spill 型"问题调大 executor 内存/核数配比、提高 shuffle partitions 分散单 task 压力（配合 AQE）；③ 任务层面：把 SQL 拆成多个 checkpoint 落中间表（隔离失败域、断点续跑）、中间结果压缩（`spark.sql.parquet.compression.codec=zstd`）；④ 如果膨胀来自"重复计算"（如同一子查询被多次引用未物化），加缓存/落表。每一步改完都回到 History Server 对比 Stage 耗时与 shuffle 量，确认瓶颈是否转移。实践中的坑：只看总耗时会被"多个 Stage 都慢了"迷惑，要锁定主矛盾；注意**广播表随数据量增长越过阈值**的隐性变化（小表变大全表广播失败自动降级 shuffle join，Shuffle Read 量瞬间爆炸——这是"数据量翻 3 倍、耗时涨 6 倍"的高频元凶）；History Server 的 event log 默认保留期（`spark.history.retainedApplications`）可能只有几天，重要任务的历史对比数据要提前留存。

**延伸考点：** 如何从执行计划判断 join 结果"膨胀"（如 `Explain` 的行数估算 vs 实际）？广播 join 因小表变大而自动降级时，你能从哪个 UI 指标第一时间发现？

---

### Q20. 老板让你把全公司 Spark 作业成本降 30%，你从哪几个方向动手，怎么量化？

**问题：** 你是数据平台负责人，公司几百个 Spark 作业的集群成本居高不下，老板要求在不影响 SLA 的前提下降 30% 成本。你如何规划治理路线、选择优先事项、并设计量化验证的机制？

**期望加分项：**
- 先建立"成本账本"：按作业/团队/队列核算 CPU 时长与内存占用（YARN 的 `yarn application -list`/`ApplicationHistoryServer` 指标、K8s 的 Pod 用量），找出"大头"——通常 top 10 作业占 60%+ 成本，先打大头
- 治理方向成体系：① 资源侧——动态资源分配（`dynamicAllocation`）、executor 核数/内存配比优化、闲时队列复用、低峰调度错峰；② 计算侧——消除无效 shuffle/重复计算、缓存复用、AQE 开全、倾斜治理；③ 存储侧——压缩格式（zstd/snappy vs 无压缩）、数据保留策略（冷数据降副本/迁移对象存储）；④ 作业侧——低频任务降频、小任务合并、优先级与队列配额
- 有量化验证机制：改造前抓基线（每作业 CPU 时/内存时/耗时），改造后对比同口径指标，用"成本 = 资源 × 时间"公式说明每一项的节省逻辑
- 能谈取舍与风险：省成本的每一项都可能有 SLA/稳定性代价（如动态分配有启动延迟、压缩有 CPU 开销），要分作业类型定策略
- 有抓手案例：先挑 3~5 个"高成本 + 低风险"作业试点（如跑得最久的大 join 任务），跑通方法论再推广
- 有监控与治理闭环：成本预算告警（超支即报警）、按团队成本分摊（让业务感知自己的成本）、季度复盘

**减分项：**
- 没有成本核算就开口"砍作业"
- 只谈"关掉不用的集群"这种一次性动作，没有可持续机制
- 优化与 SLA 的权衡讲不清（一刀切降并发导致核心任务超时）
- 不量化、不试点，全量上参数改动
- 忽略"成本大头"分布，在几百个任务上平均用力

**解答：**
先算账再动手。**第一步成本基线**：从资源管理器拉出近 30 天每个作业的"核时 + 内存时"（YARN 的 AHS 接口 / K8s 的 metrics-server 数据），按作业×团队汇总，得到 Top 20 作业清单——如果 Top 20 占了 70% 成本，治理就聚焦这 20 个，而不是对几百个作业平均用力。**第二步分类施策**（按"成本占比 × 改造风险"矩阵排序，先做高收益低风险）：① **资源闲置**（最常见、最安全）：检查作业实际使用率——很多任务配了 100 executor 实际只有 30% 利用率，开启 `spark.dynamicAllocation.enabled=true` + `spark.dynamicAllocation.shuffleTracking.enabled=true`（shuffle 场景安全回收 executor），仅这一项常能省 20~40% 资源；配合 `spark.executor.cores`/内存配比优化（Q17 的推导反向应用，去掉超额内存）；② **计算冗余**：对 Top 作业逐个看执行计划——重复的 shuffle（多个 action 各自重算）、未用 AQE、广播降级、倾斜未治，这类修复既省钱又提速，是"双赢"项；中间结果落表/复用（同一个宽表被多个下游作业重复 join 时，先物化成中间表）；③ **存储成本**：Parquet 换 zstd 压缩（典型省 30~50% 存储，CPU 代价可控）、冷表/冷分区数据迁移对象存储或降副本、清理无引用临时表；④ **调度与频率**：与业务确认低频任务的降频/并批（如 1 小时跑 1 次改成 4 小时），大任务错峰到低电价/低峰窗口（若云上按量计费）。**第三步量化与试点**：每个方向先选 1~2 个代表性作业试点——记录改造前后的"核时、内存时、耗时、失败率"，用成本公式（核时 × 单价 + 内存时 × 单价）算节省额，确认无损后推广。**第四步机制化**：建"作业成本预算 + 超支告警"（按团队配额，超支自动发账单），季度盘点 Top 成本作业清单，把成本治理做成持续流程而不是一次性项目。**关键取舍要讲明**：dynamicAllocation 在 K8s 下 executor 启停慢、对分钟级小任务不划算（该任务用固定 executor）；压缩节省存储但增加读侧 CPU；降频影响时效性，必须与业务确认 SLA；如果部分任务是"深夜批 + 早高峰报表"依赖，错峰要守住下游依赖链。最后汇报时用一张表：方向、试点作业、改造前后核时/内存时、节省额、风险等级——用数据说服而不是用口号。

**延伸考点：** YARN 与 K8s 下"核时/内存时"的核算口径分别怎么取（如何把 Spark UI 的 executor 用量对应到资源管理器的账单）？省成本与"任务稳定性"冲突时（如回收 executor 导致 shuffle 重算），你的决策框架是什么？


