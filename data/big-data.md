# 数据 · 大数据基础（面试题库）

本文件聚焦大数据平台工程师/数据工程师在 Hadoop 生态上的真实工程能力，覆盖 HDFS 存储治理（小文件、块策略、NameNode 高可用、数据均衡）、YARN 资源调度与队列规划、MapReduce/Hive 作业调优（倾斜、慢任务排查、分区分桶、UDF）、存储格式与压缩选型（Parquet/ORC、Snappy/ZSTD）、跨集群传输、任务稳定性治理，以及数据湖（Delta Lake / Iceberg / Hudi）选型与 ACID 保障等通用基础设施主题。题目均为线上可复现的场景化问题，重点考察排查链路、量化依据与取舍判断，不考概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

### Q1. HDFS 上几亿个小文件导致 NameNode 内存告警，怎么系统性治理？

**问题：** 集群 NameNode 堆内存告警，统计发现 HDFS 上散落着几亿个 KB 级小文件（很多是上游 Flume/Hive 任务产生的），你如何系统性治理？

**期望加分项：**
- 能先量化：明确每个文件/块的元数据内存开销（约 150 字节量级），据此估算总内存并判断"减存量"还是"控增量"是主矛盾
- 治理思路分层：入口控增量（刷写策略/合并写入）+ 存量合并（distcp 重组、Hive concatenate、Spark 重写）
- 能结合业务说取舍：归档类数据与实时读取类数据治理策略不同
- 有监控闭环意识：建文件数/小文件占比监控，验证治理效果
- 主动提到分区数 × 文件数乘法效应、目录层级过深等隐藏源头
- 知道不能靠一味调大 NameNode 堆内存来兜底，堆内存过大也会拖慢 GC 与故障恢复

**减分项：**
- 只会说"合并小文件"，说不出合并的具体工具和命令
- 不先统计分布（哪个目录、哪张表、哪个任务产生的），无脑全量合并
- 只减存量不控增量，治理完三个月又涨回来
- 不知道小文件对 MapReduce/Spark 的副作用不止内存，还有任务数与调度开销
- 盲目调大 NameNode 堆内存并认为问题解决

**解答：**
先量化再动手。HDFS 上每个文件、每个块在 NameNode 内存中约占 150 字节（含 inode 与 block 对象，含副本信息），1 亿个小文件即约 1.5GB 元数据开销，且写入时 block 汇报、GC 压力都随之上升。第一步做存量盘点：按目录/表统计文件数与大小分布（`hdfs fsck` 可对目录输出文件数），定位 Top 来源，通常集中在 Flume/Hive 动态分区写入、Spark `coalesce(1)` 未生效等场景。第二步控增量（治本）：Flume 侧调大 `hdfs.rollInterval`/`hdfs.rollSize` 按大小滚动；Hive 写入端设置每个 reducer/mapper 输出合并（`hive.merge.mapfiles`/`hive.merge.mapredfiles` 并调 `hive.merge.size.per.task`）；Spark 用 `coalesce`/`repartition` 控制分区数，目标单文件 128MB 量级。第三步减存量（治标）：对纯归档目录用 distcp 合并重写；Hive 表可用 `ALTER TABLE ... CONCATENATE`（ORC 支持，直接合并 Stripes 不动数据）；或 Spark 读源表按分区重写。第四步建监控：按天统计新增文件数，超阈值告警。实践中的坑：合并前先确认目标表/目录的读取方，避免合并大文件后点查类下游延迟变高；对仍在持续写入的目录要先停写再合并，否则产生新碎片；调大堆内存只作为临时缓解，NameNode 堆上 64GB 后 Full GC 会显著拖慢故障切换。

**延伸考点：** 小文件对下游 Spark/Hive 作业的影响机制是什么（task 数、调度开销、shuffle）？合并后的文件大小定多少合适，依据是什么？

---

### Q2. 新集群建表时块大小设多少、副本配几份，你按什么定？

**问题：** 搭建一个新 Hadoop 集群，数据以 128MB 以上的大文件扫描型分析为主，也有少量秒级点查。块大小与副本数你会怎么设，依据是什么？

**期望加分项：**
- 能讲清块大小权衡：过大则并行度不足、单 task 数据跨度大，过小则元数据膨胀、task 数暴涨
- 知道默认 128MB 的由来（2.x 起），并能结合磁盘 IO 带宽、map 并行度量化选择
- 副本数能区分"可靠性兜底"与"读取本地性"两个诉求，给出 3 副本的取舍（存储成本 3 倍）
- 能谈机架感知：副本分布（本地 1、同机架 1、跨机架 1），以及跨 AZ/机房时的调整
- 能提冷数据降副本或 EC（erasure coding）替代副本的适用边界
- 知道改块大小只影响新写入文件，存量需迁移才生效

**减分项：**
- 只会背"默认 128MB、副本 3"，说不出为什么
- 不知道副本数与机架/故障域的关系，认为副本越多越好
- 拿 EC 当万能药，不清楚 EC 对随机读与热数据写入的代价
- 不考虑点查类访问模式，纯扫描视角定参数
- 答不出"块大小改动只对新文件生效"这个边界

**解答：**
先定块大小：它决定单文件的并行处理粒度。分析型集群默认 128MB 是平衡点——假设单盘顺序读约 150-200MB/s，128MB 单块约 1 秒读完，map 本地读命中高、task 启动与调度开销可控。若数据多是大文件扫描、task 内存有限，可调 256MB 减少 task 数与元数据量；若文件普遍几十 MB 或需要更细粒度并行，维持 128MB 甚至更小。判断标准是让"map 数 × 单 task 时长"落在合理区间（单 task 几分钟量级），并控制全集群文件块数在 NameNode 内存承受范围内。再定副本：3 副本对应"容忍任意单节点/单机架故障"，机架感知下第 1 副本写本机、第 2 副本写同机架另一节点、第 3 副本写其他机架，兼顾写入带宽与容灾。存储成本敏感且数据可重建时，可降为 2 副本或对冷数据用 EC（如 RS-6-3，1.5 倍开销，读放大和重建计算量大，适合只读归档，不适合热数据与随机小读）。点查类访问不要靠副本解决——副本只提升本地性，不解决检索效率，应走索引/缓存/预聚合。实践中的坑：块大小调大后 map 本地读命中率可能因跨块读取下降；副本数修改只对修改后写入的块生效；EC 目录与副本目录不能混在同一目录（HDFS 按目录策略区分），迁移时要注意。

**延伸考点：** 机架感知配置错了会出现什么问题（副本同机架堆积、跨机架读放大）？EC 与副本在故障恢复时间和写放大上的差异怎么量化？

---

### Q3. NameNode 进程挂掉/磁盘写满导致集群不可写，怎么恢复与预防？

**问题：** 某天 NameNode 所在机器磁盘写满（edits 日志目录），集群进入只读保护，任务大面积失败；另一次是 NN 进程被 OOM 杀掉。两次故障你分别怎么处理，长期怎么预防？

**期望加分项：**
- 能区分故障类型对症下药：磁盘满先清理/扩容恢复写入，OOM 则分析堆内存与元数据规模
- 知道 edits 写不进去时集群会进入 safe mode 只读，先理解再操作
- 有 HA 的话讲清切换机制：QJM + ZKFC，Active 与 Standby 的状态确认（`hdfs haadmin -getAllServiceState`）
- 能说预防体系：edits 与 fsimage 分盘、NN 堆内存与元数据量监控、GC 监控、定期检查点调优
- 知道恢复后要做校验（fsck 或对比 block 副本数），确认无数据丢失
- 主动提"磁盘满"对 fsimage 合并（checkpoint）的影响

**减分项：**
- 不判断类型直接 `-safemode leave`，导致元数据不一致
- 不知道 edits 磁盘满会连带 checkpoint 失败，只清数据盘
- 只会"重启大法"，重启后才发现 fsimage 落后
- 没有 HA 时不会用 fsimage/edits 手工恢复的流程
- 恢复完不做数据完整性校验

**解答：**
先分两类。磁盘满场景：edits 目录所在盘满了 → 新 edit 无法落盘 → NN 强制进入 safe mode 只读并报 `Unable to start log segment`。处理顺序：先确认是 edits 盘还是数据盘满（`df -h`），若是 edits 盘：优先清理同盘其他目录腾空间，或临时挂载新盘迁移 edits 目录，恢复可写后再观察 NN 退出 safe mode；期间不要贸然重启。若此时还有 HA：确认 Standby 状态与 JournalNode 同步是否正常，可手动触发切换让 Standby 接管，再修盘上的 Active。OOM 场景：看 `hs_err`/GC 日志判断是元数据规模超堆（小文件膨胀）还是内存泄漏，短期调大堆并重启，长期靠治理小文件（见 Q1），并监控 GC 停顿与堆水位。预防体系四件套：① edits 与 fsimage 分开挂载（edits 用 SSD，fsimage 放另一块盘），避免互相挤占；② 每 2 小时或 1GB edits 做一次 checkpoint（`dfs.namenode.checkpoint.period/checkpoint.txns`），控制 fsimage 落后窗口；③ 监控 NN 堆内存、JVM GC、edits 目录磁盘水位，低于阈值告警；④ 定期 fsck 与副本健康检查。实践中的坑：safe mode 下不要手动 leave，除非已确认 blocks 满足阈值（`dfs.namenode.safemode.threshold-pct`）；HA 下严禁在两边同时做维护操作，避免双 Active 脑裂；恢复后先用 `hdfs fsck /` 抽查副本数，再放业务流量。

**延伸考点：** 双 Active（脑裂）是怎么产生的，ZKFC 的 fencing 机制如何兜底？fsimage 落后于 edits 太多时会有什么恢复风险？

---

### Q4. 新加了 10 个 DataNode，数据却没流过去，集群使用率仍不均衡，怎么办？

**问题：** 集群扩容加了 10 台新节点，几天后看各节点磁盘使用率，老节点 85%、新节点 20%，数据没有自动均衡，你如何处理？均衡过程中要注意什么？

**期望加分项：**
- 能解释为什么不会自动均衡：写入仍优先本地/随机，新节点只有在被选中写副本时才参与
- 会用 balancer 并说清参数：`hdfs balancer -threshold 10`（节点使用率与均值差 ±10%）、带宽限制 `-D dfs.datanode.balance.bandwidthPerSec`、运行窗口
- 知道 threshold 的量化含义：10 表示与平均值差 10% 以内视为均衡，越小越耗时
- 能提 diskbalancer 处理单节点多盘不均衡、mover 处理存储分层（hot/warm/cold）
- 结合业务选时间窗口：低峰期跑、限带宽，避免挤占在线任务
- 能说清"数据均衡 ≠ 热点均衡"，热点写目录不会因 balancer 而分散

**减分项：**
- 以为加节点后数据会自动均匀分布
- 只会敲 `hdfs balancer`，说不出 threshold 与带宽参数的含义和取值依据
- 高峰期跑均衡，把在线任务拖垮
- 不看结果指标（节点使用率方差/max-min）验证效果
- 不知道新节点上线后要关注写入路径是否真正用上（机架感知、写策略）

**解答：**
新节点加进来时不会自动搬数据：写入会按写策略选择目标节点，副本挑选有随机性，存量数据更不会自己流动，所以使用率差异是必然的。处理用 balancer：先看当前各节点使用率分布（`hdfs dfsadmin -report` 或 RM UI 的 DataNode 页），确定阈值——通常 `-threshold 10` 即"与集群平均使用率相差 10% 以内视为均衡"，追求更匀可设 5，但耗时和移动量成倍上升，不推荐低于 5。限速跑：`hdfs balancer -threshold 10 -D dfs.datanode.balance.bandwidthPerSec=50m`，在业务低峰（如凌晨）执行，避免挤占在线任务带宽与磁盘 IO；可写成 cron 任务每周跑。单节点内部多块盘不匀用 `hdfs diskbalancer`（按盘生成计划并执行）；异构存储（SSD/HDD 分层）用 `hdfs mover`。实践中的坑：① 均衡是"数据块搬移"，会占用源/目标节点的磁盘 IO 与网络，务必限带宽并选低峰；② threshold 阈值设太小会反复触发搬运，集群刚扩容完毕就停掉；③ 均衡不解决"热点写入目录"问题——某个热目录持续被写时，新节点依然可能没份，热点靠写入端分区策略或更细的块分布策略解决；④ 用使用率标准差或 max-min 差值做效果指标，别只看单个节点；⑤ 若集群有 Rack 配置错误，均衡会大量跨机架搬移，先查机架感知再跑。

**延伸考点：** balancer 的搬移策略如何避免把刚均衡的数据又搬到热点节点？动态节点下线（decommission）与均衡的配合流程？

---

### Q5. 一个跑了两小时的 MapReduce/Hive 作业，怎么一步步排查慢在哪里？

**问题：** 某离线作业平时 40 分钟跑完，今天两小时还没结束。你按什么顺序、用哪些手段定位瓶颈？

**期望加分项：**
- 先看宏观再定位微观：作业时间线分 map / shuffle / reduce 三段，先确定慢在哪一段
- 区分"排队等资源"与"执行慢"：先看 job 状态是否长时间 PENDING/RUNNING、队列资源是否充足
- 会用 Counter 与日志：GC time、spill 次数、bytes read/written、shuffle 传输量
- 会看 task 执行时间分布：max/avg 差距大 → 倾斜或单 task 数据量大；整体变慢 → 资源或集群问题
- 有对比思维：与历史基线对比（同一作业不同日期的运行时长/输入量），先确认是数据量涨了还是环境劣化
- 能给出从估算到定位的链路：输入规模 × 单位吞吐 ≈ 理论耗时，先算再查

**减分项：**
- 一上来就调参（加大内存、加并行度），不做定位
- 只看整体耗时，不看阶段分布，把 map 慢当 reduce 慢治
- 不区分等资源与执行慢，白等半天
- 不看数据量变化，作业慢以为是环境问题，实际是输入翻倍
- 不会用 Counter/日志，只会看进度条

**解答：**
先定段再定位。第一步看作业时间线（RM UI/JobHistory），把总耗时拆成 map 阶段、shuffle 传输、reduce 阶段三段，明确"慢在哪"。同时看作业状态：若长时间处于等待调度（PENDING）或 task 大量 `Waiting for container`，是资源问题——查队列利用率、是否有其他大作业抢占（capacity/fair 调度下查队列详情）。第二步判断是否数据量变化：对比该作业历史运行的输入字节数与完成时间，输入翻倍导致的慢是预期内，不算故障。第三步深入执行慢的段：map 慢 → 看本地命中率（`DATA_LOCAL` task 占比）与单 task 输入大小是否失衡（个别 map 处理远超均值 → 输入文件大小不均）；shuffle 慢 → 看 spill 次数（`SPILLED_RECORDS` 大说明排序缓冲不足）、中间数据压缩是否开启、reduce 拉取是否串行瓶颈；reduce 慢 → 看各 reduce 处理时长分布（max/avg 差距大 = 数据倾斜）与 GC time 占比。第四步对照估算：输入 1TB、单核扫描吞吐约 50-100MB/s，集群 200 vcore 理论下限 ~1-2 分钟——任何阶段远超量级都可疑，结合排查。实践中的坑：① 别只看总时长，三段分布才能定位；② 加内存治 shuffle spill 前，先确认 spill 是否真的多；③ 小数据量作业慢往往不是性能问题而是调度/启动开销，排查方向完全不同；④ 用作业级监控（运行时长、输入量、GC、spill）建立基线，慢作业靠"与基线对比"秒级定位。

**延伸考点：** 如果发现是输入文件大小严重不均（一个大文件 vs 一堆小文件），怎么调？reduce 拉取慢与 map 输出大小、压缩格式有什么关系？

---

### Q6. 一个 group by 作业 200 个 reduce 只有 1 个跑 1 小时，其余几分钟，怎么定位和治理倾斜？

**问题：** 日活统计 SQL 里 `group by user_id`，200 个 reduce 中 1 个跑了 1 小时没结束，其余几分钟就完了。你如何确认是倾斜、怎么治理？

**期望加分项：**
- 能确认倾斜：task 执行时长分布 max/avg、shuffle 输入记录数看单 reducer 数据量
- 区分 group by 倾斜与 join 倾斜，治理手段不同
- 会讲加盐（salting）方案：大 key 打散 + 两阶段聚合，并指出对 count/去重类指标的处理差异
- 能提 hive 参数兜底（`hive.map.aggr` 开 map 端聚合）与倾斜 key 识别（抽样统计 Top key）
- 主动谈边界：加盐后 group by 结果需二次合并，join 倾斜不能直接用同一招
- 能从源头预防：业务字段设计（如把空值/默认值洗掉）、写入端均衡

**减分项：**
- 只会说"加随机前缀"，说不出两阶段聚合的完整 SQL 与去重陷阱
- 不先定位就怀疑倾斜，直接全表重算
- 把 group by 倾斜和 join 倾斜混为一谈
- 不知道空值/默认值（如 user_id=0）是倾斜高发源
- 治理后不验证单 reducer 数据量是否回归均衡

**解答：**
先确认再治理。在 job 详情里看每个 reduce 的 shuffle 输入记录数与执行时长：单 reducer 输入量是均值几十倍且时长同步放大，即可确认倾斜；再用 SQL 抽样统计 key 分布（`select user_id, count(*) c from t group by user_id order by c desc limit 10`）找出热点 key——常见元凶是空值、默认值（user_id=0、''）和少量超级用户。治理分两种。若是 group by 倾斜：加盐两阶段聚合——第一段对 key 加随机前缀打散并行聚合，第二段去掉前缀再聚一次，但必须区分指标类型：count 可拆两段；sum 需按前缀拆开重加；count distinct 需去重（用 `approx` 或改造为按前缀分组后 distinct 再合），直接套模板会算错。SQL 示意：

```sql
-- 第一阶段：加盐打散
select substr(salt, 1, 2) as salt, user_id, count(*) as c
from (select user_id, concat(cast(rand()*100 as int), '-', user_id) as salt from t) tmp
group by substr(salt,1,2), user_id;
-- 第二阶段：按 user_id 合并（count 类指标可直接合）
```

若是 join 倾斜：把热点 key 行拆出来单独 join 再 union（热点侧广播或加盐+去重还原），普通 key 走原 join。参数兜底：开 map 端聚合 `hive.map.aggr=true` 对 group by 有显著效果。实践中的坑：① 空值倾斜先清洗（where user_id != ''）比加盐更划算；② 加盐的随机范围要覆盖 reduce 并发度，否则打散不充分；③ 动态分区写入时倾斜与"单分区写爆"叠加，注意分区数上限；④ 治理后对比验证：看单 reducer 输入量的 max 是否回落到均值几倍以内，别只看作业是否跑完。

**延伸考点：** join 场景下"热点 key 单独处理"的具体实现与广播 join 的边界？count(distinct) 在加盐两阶段下怎么保证正确？

---

### Q7. shuffle 阶段 spill 频繁、耗时占比 60%，MapReduce 作业怎么调优？

**问题：** 一个 MapReduce 作业总耗时 40 分钟，shuffle 占了 25 分钟，日志里 `SPILLED_RECORDS` 巨大。你从哪些参数入手调优，依据是什么？

**期望加分项：**
- 先理解 shuffle 瓶颈链路：map 端排序溢出 → merge 因子 → 压缩 → reduce 端并行拉取 → merge，按环节定位
- 会调 map 端排序缓冲与溢出阈值：`mapreduce.task.io.sort.mb`、`mapreduce.map.sort.spill.percent`、merge factor `mapreduce.task.io.sort.factor`
- 会开中间数据压缩：`mapreduce.map.output.compress=true` + Snappy，降低网络传输量
- 能调 reduce 拉取并行度：`mapreduce.reduce.shuffle.parallelcopies`，并注意 fetch 失败重试参数
- 有内存配比意识：container 内存与 JVM 堆（`mapreduce.map.memory.mb` 与 `-Xmx`）的关系，避免堆外 OOM
- 知道 spill 多不一定是坏事（阈值 0.8 时溢出是设计行为），要先看 merge 与网络才是真瓶颈

**减分项：**
- 一看到 SPILLED_RECORDS 就加 sort.mb，不理解 spill 是正常机制
- 只调内存不调压缩，网络瓶颈没解决
- 把 JVM 堆开到接近 container 内存，导致堆外（shuffle 缓冲、native）OOM
- 不区分 map 端 spill 与 reduce 端 merge 的开销差异
- 参数调整后不做对比验证，靠感觉

**解答：**
先分清 shuffle 的三段开销：map 端（排序 + spill + merge）、传输（网络拷贝）、reduce 端（并行拉取 + merge + 喂给 reduce）。日志里 SPILLED_RECORDS 大说明 map 端排序缓冲频繁溢出——注意这是阈值 0.8 时的正常设计，真正要问的是"溢出后的 merge 次数与传输量"是否成为瓶颈。调优按顺序来：① 中间数据压缩最划算：`mapreduce.map.output.compress=true`、`mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec`，网络与磁盘 IO 双降，CPU 代价小；② map 端排序缓冲：`mapreduce.task.io.sort.mb` 从默认 100MB 提到 200-400MB（受 container 内存约束），`mapreduce.map.sort.spill.percent=0.8` 保持，调大缓冲减少 spill 次数；③ merge factor `mapreduce.task.io.sort.factor` 从 10 提到 32-64，减少 merge 轮次（对大量小 spill 文件关键）；④ reduce 端拉取并行度 `mapreduce.reduce.shuffle.parallelcopies` 默认 5 可提到 10-20（网络带宽够时），`mapreduce.reduce.shuffle.fetch.retries` 控制失败重试；⑤ 内存配比：container 4GB 时 JVM 堆 `-Xmx` 留 30-40% 给堆外（shuffle 缓冲、压缩、native），堆开满必然堆外 OOM。实践中的坑：① 排序缓冲调大要同步看 GC——堆内大数组复制反而引入 GC 压力；② reduce 数少时 reduce 端 merge 的是海量 map 输出，此时调大 `sort.factor` 收益最明显；③ 中间压缩与最终输出压缩是两回事，别只配了最终输出；④ 每调一个参数跑同一份数据对比总耗时与 shuffle 占比，避免参数叠加大起大落。

**延伸考点：** reduce 端 merge 的内存上限由哪个参数控制，与 sort.factor 如何联动？推测执行（speculative execution）在资源紧张时该开还是关？

---

### Q8. 分区表每天写入几万个小文件，查询越来越慢，分区策略怎么设计才合理？

**问题：** 某 Hive 分区表按天分区，但每天经动态分区写入产生几万个小文件（分钟级粒度分区或并行度过高），查询扫描 IO 暴涨。你会怎么设计分区与写入策略？

**期望加分项：**
- 先诊断小文件来源：并行度 × 动态分区组合（每 task 每分区一个文件）
- 讲清分区粒度权衡：分区用于裁剪，但粒度过细（小时/分钟级）在数据量不足时产生海量小文件
- 会开合并：`hive.merge.mapfiles` / `hive.merge.mapredfiles`、`hive.merge.size.per.task`、`hive.merge.smallfiles.avgsize`
- 动态分区参数兜底：`hive.exec.dynamic.partition`、strict 模式、`hive.exec.max.dynamic.partitions` 上限
- 能根据数据量与查询模式定粒度：日增量小就按天，日增量大且按小时查询才按小时
- 主动提到分区裁剪失效的坑：过滤条件对分区列做函数运算

**减分项：**
- 只会说"分区按天"，不考虑数据量级与查询模式
- 不知道动态分区 + 高并行度 = 小文件爆炸的乘法效应
- 合并参数只开一半（只开 mapfiles 不开 mapredfiles）
- 分区列类型/顺序设计随意，导致存储布局难维护
- 忽略 msck repair 同步分区元数据这类运维动作

**解答：**
先治存量再定规则。小文件乘法效应：1000 个 task × 10 个动态分区 = 1 万个文件/天，30 天就是 30 万个小文件——NameNode 内存和查询扫描 IO 双爆。定分区粒度前先量化：日增量 < 几 GB 时按天足够，裁剪收益靠"分区数少、每分区文件规整"；日增量几十 GB 且查询常带小时过滤才考虑按小时，并确保每小时数据量 ≥ 1-2 个块大小，否则每小时几 MB 的分区就是移动的小文件堆积。写入端兜底三件套：`set hive.merge.mapfiles=true;`（map-only 任务合并小文件）、`set hive.merge.mapredfiles=true;`（MR 任务输出合并）、`set hive.merge.size.per.task=268435456;`（合并目标 256MB）、`set hive.merge.smallfiles.avgsize=16777216;`（平均小于 16MB 就触发合并）；配合动态分区：`hive.exec.dynamic.partition.mode=nonstrict`（全动态）并显式设 `hive.exec.max.dynamic.partitions`（默认 1000 量级，按业务确认上限），partition 列建议放在 SELECT 末尾。查询侧检查裁剪：`where substr(dt,1,7)='2026-08'` 这种对分区列做函数运算会失效，改 `where dt >= '2026-08-01' and dt < '2026-09-01'`。实践中的坑：① 合并触发条件是"平均文件小 + 总数多"，文件已经规整时不会重复合并，放心开；② 动态分区写入后若发现分区元数据与 HDFS 不一致（外部表或手动删文件），`msck repair table t;` 同步；③ 分区列别用中文/长字符串，影响 HDFS 路径长度与查询 SQL 可读性；④ 治理后按天监控"每分区文件数与平均文件大小"，确保规则长期有效。

**延伸考点：** 动态分区写入时如何限制单个 task 产生的分区数，避免某 task 写爆内存（partition buffer）？静态+动态混合分区的适用场景？

---

### Q9. 两张千万级事实表频繁 join，怎么用分桶把 join 提速，分桶怎么定？

**问题：** 两张日增量千万级的事实表，每天全量 join 做指标计算，走了普通 join 后 shuffle 数据量大、耗时高。你想用分桶优化，具体怎么做、分桶数怎么定、有什么前提条件？

**期望加分项：**
- 讲清分桶原理：按桶列 hash 分桶，同桶号数据可本地 join（bucket map join / SMB join），省 shuffle
- 能说分桶数选择依据：桶大小与块大小、文件数匹配（目标每桶 128MB-1GB），且关联两表桶数相同或成倍数
- 会说前提条件：两表桶列 = join 列、桶数一致（SMB 要求相同或倍数）、数据按桶列有序（sorted）时才能走 sort-merge join
- 能写出建表语句：`CLUSTERED BY (user_id) INTO 128 BUCKETS` 与 `SORTED BY (user_id)`
- 开启参数：`hive.optimize.bucketmapjoin=true`、`hive.optimize.bucketmapjoin.sortedmerge=true`，并验证执行计划是否走 BucketJoin
- 知道分桶表写入的坑：动态分区写入可能破坏桶分布，需按桶列 sort 后写入

**减分项：**
- 分桶数拍脑袋，不看数据量与块大小
- 桶列与 join 列不一致还说能优化
- 不知道 SMB 还需要 sorted + 同桶数前提，以为分桶就自动快
- 只建表不验证执行计划（EXPLAIN 里没出现 Bucket/SMB Join 还觉得成功了）
- 不关心写入侧是否会破坏分桶结构

**解答：**
分桶的价值在于让"同桶号的数据在 join 时本地匹配"，省掉整表 shuffle，但要满足硬前提：① 两表桶列 = join 列；② 桶数相同或成整数倍；③ 走 SMB（sort-merge bucket join）时还要求两表桶内数据按桶列排序（`SORTED BY`）。分桶数依据数据量定：日增量千万级（约 1-2GB 压缩后），目标每桶 128MB 量级，取 16-64 桶（`CLUSTERED BY (user_id) INTO 32 BUCKETS`）；桶数太少桶内数据太大、太多则小文件与元数据膨胀，经验上"总数据量 / 目标桶大小（128MB-256MB）"取 2 的幂。建表：

```sql
CREATE TABLE fact_a (
  user_id bigint, ...)
CLUSTERED BY (user_id) INTO 32 BUCKETS
SORTED BY (user_id) STORED AS ORC;
```

写入时要保证数据真正按桶分布：map-reduce 写入默认按桶列 partitioner 分桶，动态分区 + 高并行度写入容易在桶内再产生碎片；必要时 `DISTRIBUTE BY user_id SORT BY user_id` 显式控制。查询侧开启并验证：`set hive.optimize.bucketmapjoin=true; set hive.optimize.bucketmapjoin.sortedmerge=true; set hive.input.format=org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;`，执行 `EXPLAIN` 确认计划中出现 BucketMapJoin/SMB Map Join，而不是"自认为优化了"——这是最常见的无效优化。实践中的坑：① 分桶表 alter 桶数要重建表，迁移成本高，前期定好就别频繁改；② 小表 join 分桶大表，先考虑普通 MapJoin（广播）更简单；③ 分桶优化只解决 join shuffle，group by 倾斜与分桶无关；④ 桶内文件如果被合并打乱有序性（CONCATENATE），SMB 会降级，合并后复查执行计划。

**延伸考点：** bucket map join 与 sort-merge bucket join 的区别，什么情况下前者就够了？分桶对抽样（tablesample）和后续增量写入有什么额外帮助？

---

### Q10. 要写一个解析 JSON 的 UDF 和一个多行聚合的 UDAF，有哪些工程坑？

**问题：** 业务要在 Hive SQL 里对 JSON 字段提取多层字段，还要做一个"按用户聚合最早/最新事件"的自定义聚合。写 UDF 和 UDAF 时，从实现到上线你考虑哪些点？

**期望加分项：**
- UDF 选型：能用 `get_json_object` 等内置就不写；要写用 GenericUDF（支持复杂类型与类型判断）而非 Simple 版
- 知道 UDF 的 evaluate 是每行调用：避免每行创建大对象（JSON 解析器复用）、避免在 initialize 里做重活
- UDAF 结构：GenericUDAFResolver2 + evaluator，实现 map 端部分聚合（iterate/terminatePartial）与 reduce 端 merge
- 会控制聚合内存：iterate 里维护紧凑中间结构（如只保留最早/最新记录的字段），而非全量 buffer 到 terminate
- 能谈类型与 null 处理：initialize 返回正确 ObjectInspector，null 输入不抛异常
- 上线流程：先小数据验证、再灰度跑、观察 GC 与单 task 内存

**减分项：**
- 能用内置函数却硬写 UDF（维护成本白付）
- UDF 里每行 new 解析器/连接，性能差且 GC 高
- UDAF 在 iterate 里攒全量数据到内存，OOM 才醒悟
- 不处理 null/空字符串，线上跑挂
- 不知道 Resolver 按参数类型分发重载

**解答：**
先做减法：JSON 提取优先 `get_json_object`/`json_tuple`，只在内置函数覆盖不了（嵌套数组遍历、复杂逻辑、性能敏感）时才写 UDF。写 UDF 用 GenericUDF：initialize 里校验参数类型并返回结果的 ObjectInspector（决定序列化与类型），evaluate 每行被调用——核心工程点是把昂贵的对象放 initialize 创建并复用：如 JSON 解析器实例放成员变量，evaluate 里只做解析复用，避免每行 new 一个解析器（百万行就是百万次分配，GC 压力巨大）；行内异常捕获后返回 null 而非抛异常，避免单条脏数据拖死整个 task。写 UDAF 的结构是 GenericUDAFResolver2（按参数类型分派 evaluator 类）+ 自定义 evaluator 实现四个阶段：iterate（喂数据）、terminatePartial（map 端部分结果）、merge（合并部分结果）、terminate（输出最终值）。关键工程点在"聚合中间状态"的设计：比如"按 user 聚合最早/最新事件"，iterate 里只需比较时间戳并保留一条最优记录（或紧凑结构），而不是把该用户全部事件 buffer 起来等 terminate 再算——数据倾斜用户会导致单 task 内存爆。实践中的坑：① `ObjectInspector` 类型不匹配是运行时最常见的错（initialize 返回了 basic 类型但 evaluate 返回自定义对象），前后端类型必须一一对应；② 聚合窗口长、单用户事件多时，即使保留最优记录也要警惕字符串拼接产生的中间对象，用可变 StringBuffer 或 byte 数组；③ 函数名避免与内置冲突，测试用 `select my_udaf(user_id, col) from t group by user_id` 先小表跑通再上生产；④ 上线后对比作业 GC 与单 task 内存监控，确认无泄漏。

**延伸考点：** GenericUDF 里如何处理参数个数可变（重载）？UDAF 在数据倾斜下的 partial 聚合为什么能缓解，原理是什么？

---

### Q11. Hive Metastore 用嵌入式模式部署导致线上报锁错误，元数据服务怎么架构才稳？

**问题：** 线上 Hive Metastore 时不时报 `lock` 相关的错误，查下来发现有人用了嵌入式 derby 模式跑 metastore 服务，多个客户端并发直接冲突。怎么正确部署 Metastore，以及元数据怎么保证高可用？

**期望加分项：**
- 能指出根因：derby 是单用户嵌入式库，只能本地单进程访问，多客户端并发必然锁冲突
- 讲清正确架构：Metastore 独立进程 + MySQL（或 RDS），HiveServer2/Spark 通过 `hive.metastore.uris` 走 thrift 服务
- 高可用方案：Metastore 多实例部署 + `hive.metastore.uris` 配多个地址（客户端自动 failover），后端 MySQL 用主从/RDS
- 讲清 derby 的正确用途：仅用于本地测试（hive-site.xml 里 derby 连接串仅限本机）
- 能说运维要点：schematool 管理 schema 升级、备份 metastore 数据库、监控慢 SQL 与连接池
- 知道 Spark/Flink 直连 Metastore 的兼容性坑（版本差异）

**减分项：**
- 说不清 derby 与 MySQL 模式的区别，只知道"报错就重启"
- 高可用方案只有"MySQL 加个主从"，忽略 Metastore 进程本身的多活
- 不知道 `hive.metastore.uris` 配置与 failover 机制
- 升级 Hive 版本时直接跑，不知道 schema 要 `schematool -upgradeSchema`
- 不关注元数据备份，库表损坏只能重建

**解答：**
根因是部署形态错误：derby 是嵌入式、单写者的 Java 库，同一目录只能被一个 JVM 打开，Metastore 服务进程 + 多个客户端直连（Spark/Hive CLI 默认走嵌入式）互相抢文件锁必然报错。正确架构：Metastore 作为独立 thrift 服务（`hive.metastore.uris=thrift://host:9083`），后端元数据库用 MySQL——客户端只连 Metastore 进程，不再直连 derby。高可用分两层：进程层——部署多个 Metastore 实例（一般 2-3 个，无状态，共享同一后端库），`hive.metastore.uris` 配置多个地址，客户端遇到连接失败会自动切下一个；数据库层——MySQL 用主从同步 + 高可用组件（或直接用云 RDS），主库故障由 MySQL 层切换，Metastore 无感知。运维清单：① 升级 Hive 时先 `schematool -dbType mysql -upgradeSchema` 升级元数据库 schema，不升级直接连会报版本不匹配；② 定期备份 metastore 库（`mysqldump`），出问题可恢复——元数据是比数据本身更难重建的资产；③ 元数据量大后（几十万表、百万分区）Metastore 查询变慢，监控后端慢 SQL，必要时给常用查询（table/partition 查询）加索引，或用缓存层；④ Spark/Flink 版本与 Metastore 不匹配时，用 `metastore.schema.verification` 校验并保持 hive 版本一致，避免 jar 冲突。实践中的坑：多实例部署时别共用 hive-site 里的 derby 配置残留；Metastore 实例数不是越多越好——后端库连接数是瓶颈；`hive.metastore.uris` 的 failover 是"连接级"的，客户端要配超时重试。

**延伸考点：** Spark 直连 Metastore 与走 thrift 服务的区别与坑？Metastore 出现慢查询怎么定位（连接池、慢 SQL、表分区元数据膨胀）？

---

### Q12. 生产队列被大作业占满，小作业排队几小时，怎么用调度器解决？

**问题：** 同一队列里，一个 2 小时的超大作业提交后，后续几十个 5 分钟的小作业全部排队，线上报表延迟。你在不牺牲大作业的前提下怎么调度？Capacity 和 Fair 调度器分别怎么配？

**期望加分项：**
- 先定位是调度问题还是资源问题：确认队列是否已满、是否有超额申请（容器没跑满）
- 讲清 Capacity 与 Fair 的机制差异：capacity 保证队列资源下限、fair 按缺额公平分配，短作业天然占优
- 场景化方案：把大小作业拆不同队列（生产/批处理/交互）隔离，配 capacity 上限避免互相拖累
- 能说抢占（preemption）的配置与代价：`yarn.resourcemanager.scheduler.monitor.enable` + 阈值，抢占粒度是 container，代价是杀任务
- 有优先级意识：队列内 priority 只影响同一队列的排队顺序，跨队列要靠容量配比
- 知道默认不开启抢占，只配 capacity 数字不会自动生效

**减分项：**
- 只会说"上 Fair 调度器"，讲不清机制差异
- 以为配了队列容量就有了隔离，不知道默认无抢占时容量是软保证
- 不分析为什么大作业占满：可能是并行度过高或参数配置问题
- 开抢占时阈值设太低，生产作业被频繁杀，雪上加霜
- 把所有作业塞一个队列，不按业务线/重要性划分

**解答：**
先判断"占满"的性质：看 RM UI 队列详情，如果大作业实际占用的容器远小于申请量（申请了 2000 vcore 只用了 1000），是它自身并行度虚高，先在作业侧收敛（reduce 数、executor 数）；若确实吃满，才是调度策略问题。方案按两层配：第一层队列隔离——按业务重要性拆队列：`root.production`（日报/核心链路，容量 60%）、`root.batch`（大作业，40%）、`root.adhoc`（临时分析，弹性 0-20%），容量总和 100%，大作业挪到 batch 队列后不再挤压 production 的小报表；Capacity 调度器里给每个队列配 `capacity`（下限保证）与 `maximum-capacity`（上限，避免空转队列被借走后占不回来）。第二层解决"同队列内小作业排队"——Fair 调度器按"缺额"（当前用量与 fair share 之差）优先分配，短作业自然优先；Capacity 队列内则是 FIFO，可通过 `yarn.scheduler.capacity.<queue>.user-limit-factor` 与队列内多用户配额缓解。若队列高峰期仍需弹性，再考虑抢占：`yarn.resourcemanager.scheduler.monitor.enable=true` 配 `yarn.resourcemanager.monitor.capacity.preemption.*` 阈值（如超过 target 的 10% 才触发），抢占粒度是整容器杀 task，代价是重跑成本——只给"高优先级队列 vs 低优先级队列"的场景开，同一重要性队列间别开。实践中的坑：① 默认配置里抢占关闭，写了 capacity 不等于高峰期能抢回来；② 抢占阈值设太小（如 2%）会在大作业启动瞬间频繁杀小作业；③ 队列分层后要同步监控各队列利用率与排队时长，用数据调容量配比，别拍脑袋。

**延伸考点：** Fair 调度器的 fair share 怎么计算，权重（weight）起什么作用？抢占与"等待重试"的取舍，什么时候该用队列弹性伸缩替代？

---

### Q13. 公司要新上一个 500 人数据团队，YARN 队列怎么规划、资源怎么估算？

**问题：** 组织要给 500 人数据团队搭一套新的离线计算集群，业务分：核心报表（SLA 高）、数据研发日常跑批、分析师 ad-hoc 查询。你怎么做集群容量估算和队列规划，依据什么？

**期望加分项：**
- 先算总账：按并发作业数与单作业资源估算总需求（峰值并发 × 平均作业资源），再折算节点规模
- 队列规划：按业务重要性分队列 + 容量配比 + 弹性上限，SLA 队列资源保障
- 会说估算的关键假设：不是 500 人同时提交，通常活跃并发是总数的 10-20%；作业平均资源按数据量推算
- 能谈弹性与超卖：预留 20-30% 空闲应对峰值，配 maximum-capacity 允许低优先级借用
- 有治理闭环：配额/利用率监控、队列容量定期复盘调整、大作业前置审批
- 主动提成本意识：先粗算（单节点规格 × 节点数 × 使用率）再精算，留增长空间

**减分项：**
- 500 人就按 500 并发算资源，集群规格虚高几倍
- 队列只按部门拆，不按作业重要性拆，SLA 保障无从谈起
- 不监控不复盘，容量配比一次定死
- 忽略 ad-hoc 与批处理的资源争抢特性
- 给不出任何量化口径（并发数、单作业资源、使用率），全凭感觉

**解答：**
先做估算再规划。第一步量化需求：500 人团队的真实并发假设——分析师白天提交多、研发凌晨跑批多，峰值并发通常是总人数的 10-20%（约 50-100 个作业同时在跑）；单作业资源按数据量推：单作业 1TB 输入、200 vcore 集群下约需 50-100 vcore 与对应内存，粗算峰值并发 80 × 平均 60 vcore ≈ 4800 vcore，按 60-70% 利用率（预留弹性）折算需 7000-8000 vcore，配 2:1 的 vcore:内存比（如每节点 32 vcore / 128GB，取整 250 台量级）。第二步队列规划（Capacity 调度器）按"重要性 × 作业类型"二维划分：

```xml
<queue name="root">
  <queue name="production">      <!-- 核心报表：SLA 保障 -->
    <capacity>30</capacity><maximum-capacity>60</maximum-capacity>
  </queue>
  <queue name="batch">           <!-- 研发跑批：容量稳定 -->
    <capacity>50</capacity><maximum-capacity>100</maximum-capacity>
  </queue>
  <queue name="adhoc">           <!-- 分析师临时查询：弹性 -->
    <capacity>20</capacity><maximum-capacity>40</maximum-capacity>
  </queue>
</queue>
```

production 保底 30% 且峰值最高 60%（防止报表被借走太多）；adhoc 只给 20% 保底但白天借 batch 空闲；`user-limit-factor` 限制单用户占用上限，防分析师一人跑满 adhoc。第三步治理闭环：按队列监控利用率与排队时长，每月复盘容量配比；大作业（如超过队列 50% 资源）前置审批或自动限并发；ad-hoc 用户强制提交前先看队列占用。实践中的坑：① 别按"人数 × 默认资源"估算，那会超配 3-5 倍；② 队列容量总和必须 100%，弹性靠 maximum-capacity 而非 capacity；③ 分析师的交互式查询（点查引擎）与批处理混在 YARN 里要区分优先级，必要时独立引擎；④ 预留的弹性余量在云上可以是自动伸缩，自建机房就多备 15%-20% 的冗余节点，防止估算偏差。

**延伸考点：** 集群使用率目标定多少合适，怎么结合 SLA 与成本权衡？队列弹性（弹性队列）和静态队列配比的适用场景？

---

### Q14. 新表选 Parquet 还是 ORC，纯靠公司内部推荐，你怎么结合场景定？

**问题：** 平台同时支持 Parquet 和 ORC，团队内部两种声音都有。你要给一张"宽表 + 频繁多条件过滤 + 少量点查"的新表定存储格式，按什么决策？

**期望加分项：**
- 能讲清两者本质：都是列式存储，Parquet 基于 Dremel 嵌套模型（复杂结构支持好）、ORC 是 Hive 原生（ACID、轻量索引、压缩优化更贴近 Hive 生态）
- 按场景取舍：引擎生态（Spark/Druid/其他读引擎支持度）、查询模式（列裁剪、谓词下推）、写入成本（列式写放大）
- 知道 ORC 的 min/max 索引与布隆过滤器对"多条件过滤"的收益，Parquet 也有页级统计但实现与效果不同
- 结合"宽表点查"说坑：纯列式对点查不友好，需要索引/过滤条件下推，或加维度表/预聚合兜底
- 有量化意识：用同数据量实测压缩比、扫描耗时、写入耗时对比，而不是跟风
- 能提 schema 演进与兼容性：跨引擎读（Trino/Spark/Flink）时 Parquet 生态更通用

**减分项：**
- 只背"ORC 压缩好、Parquet 是 Spark 标配"，说不出机制
- 不考虑读取引擎的支持矩阵，选了格式后发现某个引擎读不了
- 忽略"点查"诉求，只按扫描场景定格式
- 不实测，凭印象拍板
- 不知道 ACID（Hive 事务表）依赖 ORC，或把 ACID 需求硬套到 Parquet 上

**解答：**
先明确两个事实：Parquet 用 Dremel 嵌套编码（record shredding + assembly），对嵌套复杂结构（数组、多层 map）支持天然更好，且被 Spark、Druid、Trino、Flink 等几乎所有引擎原生支持，是事实上的开放标准；ORC 为 Hive 深度优化：每个 stripe 带列级统计（min/max、sum）、布隆过滤器、支持 Hive ACID 事务表，压缩比在同数据下通常略优。决策按三条线走：① 引擎生态——如果这条链路里有 Spark、Druid 等多引擎，选 Parquet 通用性最强；如果主要是 Hive/Tez 跑批且未来可能开事务表，ORC 更贴；② 查询模式——"宽表 + 多条件过滤"：ORC 的列级 min/max + bloom filter 在过滤条件下能直接跳过大量 stripe/row group，扫描量下降明显；Parquet 的页级统计（dictionary encoding + page index）在过滤列位于前列时也有不错效果，但不如 ORC 激进。对"少量点查"：两种纯列式都不够——点查要走预聚合表/维度表或引入索引层，靠存储格式本身解决不了；③ 实测决策：取线上代表性数据，同 schema 分别落 ORC/Parquet，比压缩比、全表扫描耗时、带过滤查询耗时、写入耗时四项，再结合引擎矩阵拍板。实践中的坑：① 列式格式配小文件是双重灾难——每个小文件都要建元数据，扫描放大，先保证单文件 ≥ 128MB；② Parquet 的 schema 演进（加列/改类型）在部分引擎下要重写文件，ORC 的 ACID 演进更平滑；③ 同一张表在 Hive 里写 ORC、Spark 里读没问题，但反过来 Spark 写 Parquet、Hive 老版本读要注意兼容版本；④ 决定后统一规范（默认格式 + 例外审批），避免团队各写各的。

**延伸考点：** Parquet 与 ORC 的谓词下推机制分别怎么实现（page index vs stripe-level stats）？数据倾斜的列作为过滤条件时，布隆过滤器在哪种格式下更有效？

---

### Q15. 集群 CPU 不高但磁盘/网络 IO 高，压缩格式怎么选才划算？

**问题：** 集群 CPU 闲置 30%，但磁盘和网络 IO 经常打满，离线作业 shuffle 和落盘的数据量都很大。有人提议全面换成高压缩比格式（gzip），你怎么评估和决策？

**期望加分项：**
- 能分清三个场景：中间数据（shuffle/落盘临时）、最终存储（表文件）、传输链路，压缩策略不同
- 能讲透取舍维度：压缩比 vs CPU 开销 vs 解压速度 vs 是否可切分（splittable）
- 针对"CPU 有富余、IO 是瓶颈"给出量化判断：用 Snappy/LZ4（速度型）还是 ZSTD（平衡型）还是 Gzip（压比型），关键看 CPU 剩余是否覆盖压缩开销
- 知道可切分性坑：gzip 单文件不可切分 → 一个文件一个 map，并行度废掉；LZO/zstd 带索引可切分
- 能给出配置位点：map 输出压缩（中间）、reduce 输出/表存储（最终）、Hive 参数 `hive.exec.compress.intermediate` 等
- 用实测数据说话：同数据集对比不同 codec 的压缩比、压缩/解压吞吐，再定标准

**减分项：**
- 只背"snappy 快、gzip 压比高"，说不清衡量公式（压缩开销是否超 IO 收益）
- 不知道 gzip 不可切分导致 map 并行度塌陷
- 中间数据与最终存储混为一谈，一个 codec 走天下
- 不看 CPU 余量就上 gzip，结果压缩成了新瓶颈
- 忽略压缩对小文件的影响（压缩后文件更小更难合并）

**解答：**
先给判断框架：换压缩的收益 = 减少的 IO（磁盘/网络）× IO 单价成本，代价 = 增加的 CPU 开销；只有当"CPU 富余足以支付压缩开销"时高压缩比 codec 才划算。分三个场景定：① shuffle 中间数据——`mapreduce.map.output.compress=true` + Snappy（MR）或 Spark 的 `spark.shuffle.compress`：中间数据寿命短、读一次，压比收益低，Snappy/LZ4 速度优先；② 最终表存储——按查询频度分：热表用 Snappy/LZ4（解压快），冷表/归档用 ZSTD（压比高且速度尚可）或 Gzip；ZSTD 是近年的甜点位：压比接近 gzip、解压速度接近 snappy，CPU 富余时优先；③ 传输/导出链路（distcp、导出文件）用压比高的 codec 省带宽。可切分性是硬约束：gzip 不可切分——一个 2GB 的 gzip 文件只能一个 map 任务处理，并行度直接塌陷；LZO/zstd 可切分（zstd 新版带分块），大文件场景禁用无索引 gzip。量化建议：选 codec 前跑一次同数据集基准，记三个数——压缩比（压后大小/原始）、压缩吞吐 MB/s、解压吞吐 MB/s，再结合集群 CPU 闲置比例算账。实践中的坑：① 只配最终输出压缩不配中间压缩，shuffle 瓶颈依然在；② 压缩后文件变小，可能催生"数量不变但单个更小"的小文件，合并逻辑要跟上；③ 压缩格式要跟随表元数据（`STORED AS` 的 codec 属性）统一声明，换 codec 要重建表/重写数据，别在生产表上偷改；④ 冷热分层：同一张表按分区设不同压缩（`ALTER TABLE ... SET TBLPROPERTIES` 按分区覆盖 codec）在部分版本支持有限，优先用分层目录 + 独立表。

**延伸考点：** zstd 与 snappy 在 shuffle 场景的实测差异怎么测，什么指标决定取舍？压缩后小文件问题如何与压缩选型联动治理？

---

### Q16. 同一份数据既要做全表扫描分析，又要做按 ID 的秒级点查，存储怎么设计才不两头吃亏？

**问题：** 一张日增 500GB 的用户行为宽表，业务既要"全量扫描算指标"，又要"按 user_id 秒级查最近行为"。如果只落一份列式表，点查会扫全表；分两套存储又浪费。你怎么设计？

**期望加分项：**
- 先讲清读写放大概念：列式存储扫描型友好但点查要扫全表（读放大），行存/索引点查友好但扫描浪费 IO
- 能说出"一份数据 + 多级访问结构"的思路：列式主存储 + 轻量索引（min/max、bloom filter）+ 高频键的预聚合/点查表
- 结合具体组件：HBase/Kudu/Doris 这类支持点查的引擎做"近期热数据"层，列式冷数据做分析层，按时间/热度分层
- 有量化意识：点查 QPS × 单次查询数据量 vs 冷数据扫描吞吐，算出分层的阈值（如 7 天内的进点查层）
- 能谈数据湖表格式（Iceberg/Hudi）的隐藏分区/索引能力，或列存上的 row group 裁剪如何缓解
- 主动提兜底：点查走预聚合结果表/缓存，别指望一份原始表两头最优

**减分项：**
- 不知道列式存储点查的读放大机制，以为加了格式就解决一切
- 只会说"存两份"，说不清同步一致性与成本账
- 分层的热/冷边界没有量化依据，纯拍脑袋
- 忽略点查也要返回整行（宽表投影），不是只取一列
- 不评估列裁剪与谓词下推对点查的实际帮助边界

**解答：**
核心矛盾是访问模式的错配：列式存储（Parquet/ORC）把 IO 按列组织，全表扫描只读所需列，效率高；但按 user_id 点查时，因为数据按列分布而非按行连续，命中行散落在各文件/行组里，至少要读"包含该 user 的所有文件块"（配合 min/max 或 bloom filter 跳过一部分），本质上仍是扫表——这就是读放大。设计上分三步：① 主存储保证"扫描不吃亏"：列式 + 排序键（把 user_id 或时间作为 sort key，让同一 user 的数据聚簇，配合 row group 级 min/max 裁剪，点查可只扫少量行组）；② 热数据点查走专用引擎：近 N 天（按点查 QPS 与数据量算，一般 7-30 天）数据同步到支持点查的引擎（HBase/Kudu/Doris 明细表，按 user_id 做 key），冷数据留在列式分析层——两条路各读各的，互不拖累；③ 高频点查走预聚合：把"最近行为摘要"按 user 粒度聚合到小表/缓存，QPS 高的场景直接打缓存，连点查引擎都不用扫。分层阈值的量化口径：点查引擎成本 ≈ 热数据量 × 副本 × 存储单价，收益 ≈ 点查从"扫 500GB"降到"扫 10MB"的 IO/延迟节省——当热窗口数据量占分析层扫描量的 1% 以下时，收益显著。同步一致性的兜底：增量同步用时间戳/版本号，冷热分界处做双读（先热后冷 + 合并）保证不漏不重。实践中的坑：① 宽表点查返回多列时，列式读取要把多列拼回整行，IO 随列数上升，别低估；② 排序键选错（如按不常用列排序）会让点查裁剪完全失效；③ 两套存储的写放大：热层写入/compaction 开销要计入总成本；④ 别为"偶尔一次点查"专门建一套，先用排序 + 裁剪 + 缓存顶住，确认 QPS 后分层才划算。

**延伸考点：** 列式存储的 row group 大小对点查裁剪的影响（row group 越大裁剪越粗）？数据湖表格式的 metadata 索引（如 Iceberg 的 manifest）在点查中的实际作用？

---

### Q17. 双机房容灾要同步 HDFS 数据，日增量 5TB、带宽有限，怎么做增量同步与校验？

**问题：** 公司要做双机房容灾，主集群数据日增 5TB 要实时/准实时同步到备集群，机房间带宽只有 2Gbps。你用什么方案同步，怎么保证不漏数据、可校验、可切换？

**期望加分项：**
- 主方案选 distcp 并说清参数：`-update -delete`（增量）、`-bandwidth`（限速）、`-m`（并发 map 数）、`-p`（保留权限/时间戳）
- 有带宽账本意识：5TB/天 ÷ 86400s ≈ 58MB/s 均值，2Gbps（≈250MB/s）够用但要留余量，且要避开高峰
- 增量方案讲清两种：distcp 基于快照 diff（HDFS snapshot + `-diff`）做增量，或按目录/分区粒度（按天分区只同步新分区）
- 校验方案：`-update` 对比 size 与 checksum（`-D mapreduce...` 或 `-skipcrccheck` 的相反用法）、`hdfs dfs -checksum` 抽样核对
- 切换预案：备集群只读验证 + 元数据同步（Hive 建表/分区语句同步），切换时按时间点切流量
- 能提工具链之外的方案：底层存储复制（云厂商 replication）、数据同步平台（DBus/Canal 针对数据库，HDFS 用 distcp/自研）

**减分项：**
- 不知道 distcp 的增量参数，只会全量拷贝
- 不算带宽账：5TB 在 2Gbps 下能否在目标时间内传完，心里没数
- 同步完不做 checksum 校验，或盲目加 `-skipcrccheck`
- 只同步数据不同步元数据（Hive 表结构），备集群有数据没法查
- 不考虑切换演练，预案停留在纸面

**解答：**
先算带宽账：5TB/天，日同步窗口若 8 小时，需 5×1024×8 / 8 / 3600 ≈ 145MB/s，超过 2Gbps（≈250MB/s）但贴近上限；若 24 小时跑则均值约 58MB/s，2Gbps 余量充足——结论：优先拉长同步窗口 + 限速跑，别追求"实时"。方案用 distcp 增量：备集群先做一次全量基线，之后每次跑 `hdfs distcp -update -delete -bandwidth 100 -m 20 /data hdfs://backup:8020/data`——`-update` 只拷有变化的文件、`-delete` 删掉备端多余文件（保证一致性），`-bandwidth` 限速防挤占生产带宽，`-m` 控制并发。更精细的增量：主备都开 HDFS snapshot，用 `distcp -diff <snap1> <snap2>` 只传两个快照间的差异，避免按文件逐一遍历；或者对按天分区的表，直接用"只同步今天及以后的分区目录"，最简单可靠。校验三件套：① distcp 自身会在 job 结束时报告 skipped/copied/failed 数量，failed>0 必须重跑；② 抽样 `hdfs dfs -checksum` 对比关键目录；③ 数据量对比（`hdfs du` 主备对账）兜底。元数据同步：Hive 表结构用 `SHOW CREATE TABLE` 导出重建，分区用 `msck repair` 补；切换演练：定期把备集群拉起做只读查询验证，演练切换时间点与回切流程。实践中的坑：① 文件正在写时同步会拿到半截文件——先对"已完成分区/目录"同步，或用 snapshot 固定一致性视图；② `-bandwidth` 单位是 MB/s，别设成 100 以为很大；③ 容灾 ≠ 备份，如果业务要求"删库也能恢复"，还需要保留版本化快照而不是纯镜像；④ 切换后要验证主备数据校验和一致再切流量，防止"切过去才发现少数据"。

**延伸考点：** distcp 的 `-diff` 基于快照的增量与 `-update` 全量对比的适用场景差异？备集群承担读流量时，如何与主集群的写入延迟对冲？

---

### Q18. 每天凌晨上百个离线任务集中跑，经常互相拖累导致 SLA 违约，怎么系统治理？

**问题：** 数据团队凌晨 0-4 点集中跑批，上百个任务共用集群，高峰期经常出现任务排队、超时，日报在早上 8 点出不来。你怎么系统性治理这个"凌晨洪峰"？

**期望加分项：**
- 先做现状量化：按作业画"提交时间 × 资源占用"分布，找出洪峰与资源缺口，再谈方案
- 治理维度分层：削峰（任务错峰/分批提交）、调度优化（依赖编排、DAG 优先关键路径）、资源（队列弹性、SLA 队列保障）
- 讲依赖与 DAG 调度：用 Airflow/自研调度做分层依赖，把非关键任务挪出洪峰窗口，关键链路任务优先
- 能谈慢任务治理联动：洪峰期慢任务大多是资源争抢 + 自身低效叠加，先清理低效任务（倾斜、小文件、无谓大扫描）
- 有监控告警闭环：任务级 SLA 监控（开始时间、完成时间、耗时趋势）、违约自动告警与重跑预案
- 有推进节奏：先止血（错峰+队列隔离）再治本（任务瘦身、资源扩容决策）

**减分项：**
- 一上来就说"扩机器"，不先看是资源不够还是任务编排不合理
- 不知道集中跑批的根本问题是任务提交没有错峰与优先级
- 只加资源不治理慢任务，扩了机器还在排队
- 没有 SLA 监控体系，违约靠业务投诉才发现
- 不考虑重跑/补偿机制，失败一次当天报表就废

**解答：**
先量化再治：拉出 0-4 点各作业的提交时间、峰值资源占用、完成时间分布——通常会发现两类问题：提交全挤在 0 点（上一日数据就绪时间相同），以及少数"低效大户"占了大半资源（倾斜/小文件/无谓扫描），先处理低效大户往往比扩资源更见效。治理分三层：① 削峰错峰——调度平台按作业 SLA 分级：核心链路（日报关键路径）锁定时段先跑，非核心任务（周报预计算、实验数据）挪到白天或 4 点后；同一时间段的并发用"提交闸门"控制（最大同时运行数），避免上百个作业一拥而上；② 资源保障——按队列隔离：关键任务进 production 队列（容量保底 30%），非关键进 batch 队列，关键链路即使高峰期也能拿到资源；必要时给核心作业开 `schedulingPriority` 提高排队优先级；③ 慢任务瘦身——对占用 Top 的作业逐个治理：数据倾斜（Q6）、小文件（Q1）、无谓全表扫描（裁剪失效），把"每个作业的固有耗时"降下来，洪峰期才扛得住。配套监控：任务级 SLA 监控（计划开始/实际开始/耗时/完成时间四个指标），超时自动告警；失败任务自动重跑（幂等前提下），重跑仍失败则触发降级预案（如先出 T-2 数据）。实践中的坑：① 错峰要结合数据就绪时间，不是随便调晚——先把依赖关系梳理清楚（DAG），再排时间；② 别把"关键任务"定义成"所有任务"，否则分级失效；③ 重跑前确认任务幂等，否则重跑产生重复数据；④ 治理是持续过程，月度复盘洪峰曲线，看削峰效果是否还在（新任务又会悄悄挤进来）。

**延伸考点：** 调度平台的依赖编排怎么做才能既保证关键路径又不错峰（如关键路径优先 + 非关键任务弹性窗口）？任务级 SLA 监控的具体指标与告警阈值怎么定？

---

### Q19. 集群从 200 台扩到 600 台，扩完发现存储和计算比例失衡，扩容该怎么规划？（开放性）

**问题：** 业务增长要求把离线集群从 200 台扩到 600 台，你负责扩容方案。扩之前你要算清哪些账、按什么顺序落地，才能避免"扩完发现存储过剩/计算不足"？

**期望加分项：**
- 先拆需求：计算需求（并发作业 × 单作业资源）与存储需求（数据量 × 副本 × 压缩比）分开估算，再定节点规格与数量
- 有规格匹配意识：同规格节点保证"存储:计算"比例匹配 workload（分析型重计算、归档型重存储）
- 落地顺序：先算容量再采购/上架，节点配置机架感知、跑 balancer 搬数据、验证写入读取路径
- 考虑扩展后的运维影响：NameNode 元数据规模（文件数）、机架规模、网络拓扑（leaf/spine）
- 能谈缩容对称性：什么时候该缩计算、什么时候该缩存储，动态调整的触发条件
- 有成本/风险视角：分批扩容观察指标（利用率、队列排队时长）再决定是否继续

**减分项：**
- 只按"数据量涨了 3 倍"就说扩 3 倍，不区分计算与存储
- 不看现有节点利用率与瓶颈（CPU 打满还是磁盘快满），盲目按比例扩
- 扩容后不跑 balancer，新节点闲置、老节点继续 85%
- 机架感知/网络规划缺失，扩容后副本分布与跨机架流量失衡
- 不做分批验证，一次性上 400 台后发现问题难以回退

**解答：**
扩容前先拆账，这是开放题的核心。第一步分别估算：计算需求 = 峰值并发作业数 × 单作业平均资源（见 Q13 口径）；存储需求 = 日增量 × 保留周期 × 副本数 ÷ 压缩比 + 冗余水位。两者换算成节点时，规格必须匹配 workload——分析型集群重 CPU/内存（如 32 vcore / 128GB / 4×4TB），归档型集群重容量（16 vcore / 64GB / 12×8TB），混合型则按比例配，避免出现"扩完磁盘空一半、CPU 天天打满"的结构性错配。第二步看现状找短板：扩容前先看现有 200 台的瓶颈在 CPU、内存、磁盘还是网络（RM 利用率、DN 磁盘水位、机架间流量），瓶颈在存储就扩存储型节点，在计算就扩计算型——扩错方向是最大的浪费。第三步落地顺序：① 新节点先配置机架感知（`topology.script`），否则副本分配会打乱现有机架分布；② 节点上架后 `dfsadmin -refreshNodes` 注册，跑 balancer（限带宽、低峰期）把数据从 85% 老节点搬向新节点（见 Q4）；③ 用压测任务验证新节点写入、读取、shuffle 全链路正常；④ 分批执行：先扩 100 台观察队列排队时长与利用率变化，数据支撑再继续，避免一次扩完发现规格选错。第四步评估扩展后的运维面：文件数翻倍后 NameNode 堆内存与 GC（配合 Q1 治理）、机架数增加后的跨机架流量（副本 2 副本 3 的机架分散）、以及是否需要调整 HDFS 与 YARN 的资源配比。实践中的坑：① 扩容后最容易被忽视的是"数据没均衡、新节点空转"，balancer 必须排进计划；② 机架感知漏配会导致副本集中在同一机架，机架断电就全丢；③ 扩完要复测队列配置（容量百分比是按集群总量算的，扩完后百分比含义变了，可能要重新规划）；④ 预留 10-15% 余量应对突发，别按 100% 利用率设计。

**延伸考点：** 机架感知配置与副本放置策略如何联动，扩容后机架拓扑变化会引发什么问题？什么指标驱动"缩容"决策（利用率、存储水位、作业排队）？

---

### Q20. 湖仓一体选型：Delta Lake、Iceberg、Hudi 怎么选，ACID 和流批一体怎么保障？（开放性）

**问题：** 公司想把部分离线数仓迁到湖仓一体架构，支持增量更新、ACID 事务、Spark/Flink 都能读写。Delta Lake、Iceberg、Hudi 三个表格式你怎么选，选完后 ACID 与流批一体怎么落地？

**期望加分项：**
- 能讲清三者本质与差异：都是"表格式 + 元数据层"，核心是快照隔离与 ACID；Delta 靠 transaction log、Iceberg 靠 manifest 快照、Hudi 靠 timeline 提交历史，并发控制与 upsert 策略不同
- 按场景选型有依据：写多读少/高频 upsert 选 Hudi（MOR）、流批一体与 Spark 生态选 Delta、多引擎 + 演进治理选 Iceberg；结合公司现有引擎（Spark/Flink/Trino）给结论
- 能讲 ACID 实现细节：乐观并发（OCC，冲突检测失败重试）vs 悲观锁、快照隔离让读写互不阻塞、时间旅行
- 落地要点：compaction 策略（MOR 读时合并的查询劣化）、小文件合并、增量读取（CDC/变更流）与下游消费
- 迁移兼容：Hive 表原地升级（表格式间迁移）、分区发现与 schema 演进、与现有 Hive Metastore 的协作
- 有取舍与成本意识：三套都引入会造成分裂，选一个作为标准，特殊场景例外

**减分项：**
- 只会背"Delta 是 Databricks 的、Iceberg 是 Netflix 的"，说不出机制差异
- 不看现有引擎生态就选型，选完发现某引擎读写支持不全
- 不知道 MOR 的读放大要 compaction 兜底，查询性能劣化了才慌
- 把 ACID 想成"数据库级别"，忽略了湖上 ACID 的冲突处理与重试代价
- 不给迁移路径，只谈"我们应该上数据湖"

**解答：**
先给框架：三者都是"open table format"——在 HDFS/对象存储上组织数据文件 + 独立的元数据层，实现快照隔离、ACID 与时间旅行，差异在实现路径与生态绑定。Delta Lake：transaction log（JSON）记录每个 commit，乐观并发控制，与 Spark 深度绑定（读写最顺），多引擎支持靠 Delta Standalone/统一协议；Iceberg：manifest 树组织数据文件，快照不可变，查询规划时精准裁剪（支持 hidden partitioning、schema 演进好），Spark/Flink/Trino 支持都较完整，社区治理中立于厂商；Hudi：timeline 记录提交历史，内置 upsert（COW/MOR 两种表）、增量查询（incremental read）与 CDC 能力，适合高频更新与流批一体场景，Flink 写入生态成熟。选型决策按三条线：① 写入模式——高频小批量 upsert（如维表、订单更新）优先 Hudi MOR（写快读慢，配 compaction 兜底）；增量追加 + 大查询优先 Iceberg（查询规划最优）；Spark 栈为主且要流批一体，Delta 上手最快；② 引擎矩阵——把公司实际在用的 Spark/Flink/Trino/Hive 版本列出来，查各表格式的支持情况，这是选型最重要的输入；③ 治理中立性——避免被单一厂商绑定，Iceberg 最中立，Delta 强绑 Spark 但功能最顺手。ACID 落地的工程细节：并发写用乐观并发 + 重试（冲突时任务级重跑，不是数据库那种事务回滚），失败率高的场景评估是否真的需要并发写同一张表，必要时按分区写规避冲突；MOR 表要配 compaction 策略（按文件数/数据量阈值触发，定期压缩小文件与 base 文件），否则读放大导致查询越来越慢；流批一体落地：上游 Flink 写 Hudi/Iceberg 增量、下游 Spark 批读快照，消费侧用变更流（Hudi incremental read / Flink 流读）做增量，注意 checkpoint 与幂等。实践中的坑：① 别三套都上——选一个标准，特殊场景走例外审批，否则数据分散在三种表格式里治理成本爆炸；② 从 Hive 表迁移不是改个格式就行，要验证分区发现、schema 演进、查询引擎兼容（老 Trino/Hive 版本读不了新表格式）；③ 湖上 ACID 不等于数据库 ACID，回滚/并发冲突的语义要跟业务讲清楚，防止业务方按数据库预期使用。

**延伸考点：** MOR 表的 compaction 触发策略怎么定（文件数/数据量/时间），与查询延迟如何权衡？同一张表多个 Flink 作业并发写时，乐观并发冲突怎么处理才不丢数据？

---