# 数据 · Airflow（面试题库）

本文件聚焦数据工程师/平台工程师在 Airflow 上的真实工程能力，覆盖 DAG 设计最佳实践（幂等、原子任务、任务粒度）、调度时间语义（execution_date / start_date / cron 的坑）、重跑与回填（backfill/clear）、重试与告警（SLA、webhook、on_failure_callback）、Sensor 与 XCom 的正确用法与滥用、并行度控制（max_active_runs / pool / queue）、scheduler 与 Executor 选型（Sequential/Local/Celery/Kubernetes）、动态 DAG 取舍、调度延迟与性能调优（parsing 优化、metastore 清理）、跨 DAG 与跨系统依赖、Secret 与连接管理、CI/CD 部署、多租户权限与平台高可用等主题。题目均为线上可复现的场景化问题，重点考察排查链路、量化依据与取舍判断，不考概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

### Q1. 每天 8 点要出昨天的报表，上游数据 9 点才稳定，调度时间怎么设计？

**问题：** 报表 DAG 每天 8 点必须产出"昨天的报表"，但埋点数仓的明细数据要 9 点才稳定，后半夜还可能有补数。这个 DAG 的调度时间和依赖怎么设计？

**期望加分项：**
- 能先纠正概念：Airflow 里 `execution_date`（新版为 `data_interval_start`）是数据区间起点而非"运行时刻"，能说清 `schedule_interval='0 9 * * *'` 在 9 点触发、对应数据区间 [昨天 9:00, 今天 9:00)
- 能统计上游就绪时间分布（P95）再倒推调度窗口，而不是拍脑袋定 8 点
- 能提出"数据就绪标记 + Sensor"代替"时间假设"，并说明补数/迟到的兜底方案
- 有量化意识：调度时刻 = 数据就绪 P95 + 处理耗时 + 安全缓冲
- 主动提到"SLA 兜底"与"就绪校验（记录数/校验和）"

**减分项：**
- 把 execution_date 当成任务运行时间，或分不清"调度时间"与"数据时间"
- 只答"schedule 改成 10 点"，说不出依赖机制，上游再晚一点又挂
- 不考虑补数与数据回刷场景
- 不知道 Airflow 默认触发时刻与 execution_date 错位带来的取数坑（`{{ ds }}` 取哪一天）

**解答：**
先想清楚一个事实：Airflow 的 `schedule_interval` 定义的是"触发时刻"，而任务真正处理的数据区间是 `[execution_date, execution_date + schedule_interval)`。例如 `schedule_interval='0 8 * * *'` 每天 8 点运行，本次 run 的 `execution_date` 是昨天 8 点，数据区间为 [昨天 8:00, 今天 8:00)，恰好覆盖"昨天全天"，所以取昨天的数用 `{{ ds }}`（即 execution_date 的日期）是对的——但前提是数据那时已完整。8 点跑必然拿到 9 点才稳定的数据，所以正确做法分两步：第一，统计上游各分区就绪时间的分布，把调度时刻定为"就绪 P95 + 自身处理耗时 + 缓冲"之后；第二，不要只靠时间，加上显式依赖——数仓侧为分区写"就绪标记"（就绪表/文件），Airflow 用 Sensor（`HivePartitionSensor`、自定义 `SQLSensor` 或 `FileSensor`）等待标记，标记不出现就 Fail。补数兜底：约定报表基于"截至 X 时刻的数据快照"，或在任务内先做完整性校验（统计记录数/主键去重数与基线对比），不满足直接失败等重跑。实践中的坑：仅靠"调度时间往后挪"最省事但最脆，上游某天延迟半小时就出脏报表；排障时"execution_date 比运行时间早一个区间"是 90% 困惑的来源，日志里必须显式打印 `data_interval_start/end` 而不是打印当前时间。

**延伸考点：** `{{ ds }}`、`{{ data_interval_start }}`、`{{ ts }}` 在任务里分别取到什么？`@daily` 与 `0 8 * * *` 在跨时区（`default_timezone`）下的触发差异是什么？

---

### Q2. 一个 DAG 塞了 50 个任务、任务间用 XCom 传大量数据，出了什么问题？

**问题：** 某 DAG 里塞了 50 个任务，任务之间用 `xcom_push`/`xcom_pull` 传 DataFrame、文件路径甚至几百 MB 的对象，后来调度越来越慢、任务偶发失败。问题出在哪？

**期望加分项：**
- 能指出 XCom 存 metastore 数据库、有大小上限（默认 48KB，超过被截断/报错），大对象会撑爆 DB、拖慢 scheduler
- 能区分"任务粒度"问题与"XCom"问题：50 个任务串行依赖导致单点失败重跑放大、并行度受限
- 能给出替代方案：大数据量走文件/对象存储（S3/GCS/HDFS）+ 传路径，XCom 只传轻量元数据
- 能谈任务拆分：按业务阶段拆成多个 DAG 或用 TaskGroup 组织
- 主动提到 task_id 变更、历史 run 映射等维护成本

**减分项：**
- 认为 XCom 是任务间传数据的通用手段，说不出存储位置和大小限制
- 只怪 XCom 不分析任务粒度本身的问题
- 答不出"大对象应该传路径/引用"这一关键点
- 不知道 TaskFlow API（`return` 值自动走 XCom）与显式 xcom 的关系

**解答：**
两个问题要分开看。第一个是 XCom 滥用：XCom 默认存在 Airflow 元数据库的 `xcom` 表里，任务每次 push 都是一次 DB 写入、pull 是一次 DB 读取，几百 MB 的对象序列化（JSON 或 pickle）后写入 Postgres/SQLite，一是撞上 48KB 的截断上限（超过会静默截断或报错），二是把元数据库撑大、拖慢 scheduler 的查询，三是任务间强耦合——生产者必须先跑完，消费者才能拉取，完全丧失并行。正确姿势：只传"引用"不传"数据"，把大结果写 S3/HDFS，XCom 里只放路径、分区、行数这类轻量元数据；真需要跨任务共享计算结果，用外部存储 + 路径参数化。第二个是任务粒度：50 个任务塞一个 DAG，串行链上任何一个失败，其后所有任务不跑，重跑只能整链 clear；且这 50 个任务共享 DAG 的并发上限，互不相关的子链也被绑在一起。合理拆分：按"业务阶段"拆多个 DAG（如 staging → warehouse → report），阶段间用外部依赖或 `ExternalTaskSensor` 衔接；同一阶段内的小步骤用 `TaskGroup` 分组。实践中的坑：TaskFlow API 的 `return` 隐式走 XCom，很多人无意识传了大数据；跨 DAG 之间千万不要用 XCom（不同 DAG 的 task 之间 pull 不到是常见事故）。

**延伸考点：** XCom 的大小限制与存储后端能改吗？50 个任务的 DAG 在什么场景下反而是合理的（如严格串行的数据管道）？

---

### Q3. 凌晨重跑一个失败任务，为什么把 7 天前甚至一个月的任务全触发了一遍？

**问题：** 某 DAG 凌晨一个任务失败，你手动 clear 重跑，结果调度器把过去 7 天、甚至 catchup 区间内的所有 run 都重新执行了一遍，资源被打爆。发生了什么，怎么避免？

**期望加分项：**
- 能解释 `clear` 的语义：默认连带清除下游依赖（`-d`/递归），且按时间范围 `-s/-e` 清多个 run
- 能区分 backfill、clear、trigger 三者的适用场景与副作用
- 能指出 `catchup=False`（新版默认）的意义，以及旧 DAG 用默认值上线时的"补跑风暴"
- 重跑前有范围评估意识：先查看历史 run 与上下游，再决定 `-s/-e`
- 能提幂等设计让重跑可安全重复执行

**减分项：**
- 分不清 clear 与 backfill、trigger 的区别
- 不知道 clear 默认递归下游导致的"连带重跑"
- 说不上 catchup 默认值在新老版本的变化及其风险
- 重跑后不做结果确认，数据被重复写入产生脏数据

**解答：**
先还原事故链：`airflow tasks clear <dag_id>` 默认是"递归清除该任务及其下游依赖"（`--downstream`），若再配合 `-s/-e` 时间范围，会把范围内多个 run 一起清掉重新调度；更常见的"补跑风暴"版本是：该 DAG 一直开着 `catchup=True`（Airflow 1.x 默认），某次配置变更或历史 run 状态异常后，scheduler 发现"某些 data interval 没有成功的 run"，于是自动为每个错过的区间创建 run，7 天、一个月的历史全部重跑。避免措施分三层：第一，新 DAG 一律显式 `catchup=False`（配合明确的 `start_date`，不给历史区间自动补跑的机会）；第二，重跑前明确目标——只重跑某天用 `airflow tasks clear <dag_id> -s <date> -e <date> --no-downstream`（或按需带 `--downstream/--upstream`）精确控制范围，要整条链路重算再带上下游；第三，所有任务做成幂等（按分区先删后写、upsert、写幂等键），保证"重跑多少次结果一致"，这是重跑安全的地基。实践中的坑：`backfill` 与 `catchup` 语义不同——backfill 是主动为历史区间创建 run（`airflow dags backfill -s -e`），catchup 是 scheduler 对"上线时已错过的区间"自动补跑；production 上对大 DAG 全量 backfill，先加 `--dry-run` 数区间数，必要时 `--limit` 分批跑，否则 DB 写入和 executor 都会被打爆。

**延伸考点：** `catchup=True` 时 `start_date` 设太早（如 3 年前）上线会发生什么？clear 时 `--downstream` 与 `--upstream` 的差别，重跑前如何确认影响范围？

---

### Q4. 上游失败时下游该不该继续跑？失败传播与幂等怎么设计？

**问题：** 一条数据链路：采集 → 清洗 → 聚合 → 出数，聚合任务失败时，下游"出数"任务默认不跑，但有次你手动把"出数"触发了一次，产出了错误报表。依赖关系和任务设计哪里出了问题？

**期望加分项：**
- 能讲清 `trigger_rule` 各取值（all_success / all_done / one_failed / none_failed）的业务含义与适用场景
- 有"失败即停"与"脏数据蔓延"的权衡意识：默认 all_success 是保护，放宽要谨慎
- 能给出任务原子性设计：一个任务只做一件事，可重入可恢复，不产生半成品
- 能讲失败处理分支：需要"失败也继续"的下游用 all_done + 分支，而不是绕过依赖
- 主动提"人工触发"要有条件：数据质量校验通过才能放行

**减分项：**
- 认为 trigger_rule 是"随便改的参数"，说不出每个取值的语义
- 只答"下游不要跑"，说不出如何用机制保证而不是靠人自觉
- 没有幂等/原子任务概念，重跑一次结果就变一次
- 不考虑"校验缺失"——没有数据质量关卡，错误数据静默流转

**解答：**
先定原则：默认 `trigger_rule='all_success'`（上游全成功下游才跑）是对脏数据的天然防火墙，不该轻易放宽。上面事故的根因不是"下游跑了"，而是"任务本身不可重入"——聚合任务把结果直接覆盖式写入报表表，但写入不是原子的（先删后插分两步、或未按业务日期隔离），第二次触发把错误中间态写进去了。所以正确设计分三层：第一，任务原子性：一个 task 只做"读入 → 计算 → 写出"一件事，写出用幂等模式（按业务日期/分区先删除后插入、upsert、或写临时表再原子 rename），保证同一输入重跑 N 次结果一致；第二，依赖语义：只有"失败后仍需执行补救/通知"的分支才用 `trigger_rule='all_done'` 配合 `BranchPythonOperator` 分流，常规数据处理链路保持 all_success；第三，人工介入兜底：生产上加"数据质量校验任务"（行数波动、空值率、主键唯一性对比基线），校验失败则任务 Fail，任何人工触发都要先过校验。实践中的坑：`trigger_rule` 放宽后，下游自己也要做输入保护（不能假设上游数据一定合法）；改 trigger_rule 必须评审，它是"脏数据静默蔓延"的头号来源。

**延伸考点：** `none_failed_min_one_success` 与 `one_failed` 的区别？出数任务依赖两个上游、一成一败时，各 trigger_rule 下分别怎么表现？

---

### Q5. 任务偶发失败，重试参数怎么设？重试太多会带来什么问题？

**问题：** 一个 Spark 提交任务每天偶发失败（网络抖动、YARN 资源紧张），你设了 `retries=5, retry_delay=30s`，结果失败时它 3 分钟内连打 5 次，把本来就紧张的队列打得更死。重试策略应该怎么设计？

**期望加分项：**
- 能区分"可重试错误"与"不可重试错误"（语法错、数据错重试无意义），并给出拦截手段
- 能说出 `retry_exponential_backoff`、`max_retry_delay`、`retry_delay` 的组合逻辑并给出合理取值（如 3 次、退避 5min→10min→20min）
- 能量化重试代价：每次重试占 executor slot、产生 DB 记录、挤压其他任务窗口
- 能按失败模式分流：偶发网络用重试，持续资源不足应告警而非重试
- 主动提到重试时间窗不能超过下游对该任务的等待上限（与 SLA 联动）

**减分项：**
- 只会说"retries 设 3"，说不出延迟与退避怎么配
- 认为重试越多越好，不考虑 slot 占用与排队效应
- 对所有错误一视同仁地重试，包括 SQL 语法错误等必败错误
- 不区分"任务内轻量重试"与"task 级重试"的取舍

**解答：**
先想清楚重试的本质：重试是在用"资源时间"换"成功概率"，必须对错误分类。必败错误（SQL 语法错、表不存在、数据格式错）重试只会反复浪费 slot，应该在任务内先做输入校验快速失败；偶发错误（网络抖动、连接被拒、资源排队超时）才值得重试。参数设计上：`retries=3`，`retry_delay=timedelta(minutes=5)`，开启 `retry_exponential_backoff=True` 并设 `max_retry_delay=timedelta(hours=1)`，即 5 分钟 → 10 分钟 → 20 分钟，总窗口约 35 分钟——既覆盖瞬时抖动，又不会 3 分钟内连打把资源打爆；窗口必须小于下游对它的等待上限，否则重试期间下游已在等数据。任务内轻量重试与 task 级重试要分工：对单次 API 调用可在 Python 里做几行退避重试（省得整个 task 重来），task 级重试用于"整个步骤需要重跑"的场景，避免两者叠加产生"重试里套重试"。实践中的坑：`retry_delay` 小于任务本身执行时长时，重试排队会层层堆积；重试次数过多让 `task_instance` 在 DB 里留下大量记录、scheduler 查询变慢；重试与告警联动——前几次重试不告警，重试全部耗尽才告警，避免告警轰炸。

**延伸考点：** 重试与 `max_active_runs_per_dag` 同时命中时任务怎么排队？`retry_exponential_backoff` 的 backoff factor 如何影响总重试时间？

---

### Q6. 任务失败要第一时间告警到钉钉/飞书，怎么实现最靠谱？

**问题：** 要求任务失败 1 分钟内把失败信息（DAG、task、失败原因、日志链接）推送到钉钉/飞书群，且重试中的失败不要刷屏。你会怎么实现？

**期望加分项：**
- 能说出 `on_failure_callback` / `on_retry_callback` / `on_success_callback` 的区别与注册层级（task 级 vs DAG 级）
- 能实现 webhook 推送：回调里 post JSON 到机器人接口，并处理网络失败（回调异常不能影响任务状态）
- 有防刷屏意识：只在"最终失败"（重试耗尽）时告警，重试中走 on_retry_callback 或静默
- 能提 SLA 告警：`sla` 参数 + `sla_miss_callback` 覆盖"任务没失败但超时未完成"的场景
- 能讲回调在哪个进程执行、要轻量、要带日志链接与 execution_date

**减分项：**
- 只会说"配 email 告警"，说不出自定义 webhook 怎么挂
- 回调里直接抛异常导致任务状态被污染
- 重试每次失败都告警，造成告警风暴
- 不知道 SLA 与失败告警是两类不同的事件
- 回调里做重活（无超时控制的长请求）拖慢调度

**解答：**
两条线：失败告警和 SLA 告警。失败告警：在 DAG 或 task 上挂 `on_failure_callback`，回调签名 `(context)`，从 context 取 `dag_id`、`task_id`、`execution_date`、`exception`，组一条消息 POST 到飞书/钉钉机器人 webhook。关键设计点：第一，只在"最终失败"时告警——回调里判断 `context['task_instance'].try_number >= context['task_instance'].max_tries`（重试已全部耗尽），否则只记录不推送，配合 `on_retry_callback` 处理"正在重试"的信息；第二，回调自身要健壮：包 try/except、设超时（`requests.post(..., timeout=5)`）、自身失败绝不回抛，防止 webhook 挂了把任务搞成 failed；第三，消息里带可跳转的日志链接（日志页面 URL + execution_date 参数）与失败摘要（异常类名+首行），群里可直接点进去排查。SLA 告警：任务"一直成功但太慢"用 `sla=timedelta(hours=...)` 声明在 task 上，超时由 scheduler 触发 `sla_miss_callback`，覆盖"没报错但数据晚了"的场景——注意 SLA 从 DAG run 的 scheduled 时刻起算，不是从任务开始算，跨天任务要算准。实践中的坑：`on_failure_callback` 在旧版本跑在 scheduler 进程，回调里做长耗时/网络阻塞会拖慢整个调度，务必轻量；告警要按 DAG 分组、可 @负责人，避免全群刷屏后被静音。

**延伸考点：** SLA 与"任务失败重试"在时间语义上的区别？`sla_miss_callback` 由哪个组件触发、多久检查一次？

---

### Q7. 用 Sensor 等上游数据，为什么任务堆积、调度变慢？Sensor 怎么用才对？

**问题：** 团队在 DAG 里用了一批 Sensor（如等 S3 文件、等 Hive 分区），每个 `timeout=24h`，后来任务大量堆积、scheduler 和 worker 都不够用。哪里出了问题？

**期望加分项：**
- 能指出 Sensor 是"占着一个执行 slot 的任务"，默认 `mode='poke'` 下会长期占用 worker slot
- 能说出 `timeout`（单次 sensor 实例最大等待）与 `poke_interval`（轮询间隔，默认 60s）的配合，及超时后自动 Fail 的机制
- 能推荐 `mode='reschedule'`：等待时释放 slot，由 scheduler 定期重新调度，避免长期占坑
- 能说清 sensor 的替代方案：数据就绪标记 + 事件驱动，或用 `ExternalTaskSensor` 等专用 sensor
- 有量化意识：sensor 数 × 平均等待时间 对 slot/DB 的占用估算
- 主动提到 sensor 超时后要能通知人，不能静默挂死

**减分项：**
- 不知道 poke 与 reschedule 两种模式的本质区别
- 认为 sensor 很"轻"，意识不到它占 slot
- 不设 timeout，让 sensor 无限等待
- 用轮询等 24 小时也不考虑事件/标记方案
- 把 sensor 当"万能等待器"随处用，不评估等待时长

**解答：**
先明确 Sensor 的本质：它也是一个 Task，默认 `mode='poke'` 时在 worker 上每 `poke_interval`（默认 60 秒）醒来探测一次，期间那个 executor slot 一直被占着——50 个 sensor 各等 24 小时，就是 50 个 slot 被锁死，worker 再大也会被占满，任务自然堆积；同时 scheduler 要为这些 sensor 反复心跳。正确姿势：第一，优先 `mode='reschedule'`——等待期间任务从 executor 退出、把 slot 释放回 pool，由 scheduler 按 `reschedule` 间隔重新放入队列，只是会增加调度次数与 DB 写入，但 slot 利用率天壤之别；第二，设 `timeout`：如 `timeout=3600`，超时后 sensor 以 Failed 结束并触发告警，绝不无限等；第三，控制 `poke_interval`：探测目标是外部系统 API 时，轮询太频繁会给对方压力，60s 起步、慢数据可到 5-10 分钟；第四，从根上减少等待：让上游在数据就绪时写"标记"（空文件/标记表），sensor 只等标记，或者用 `SQLSensor` 直接查就绪状态；能确定时序的用 `ExternalTaskSensor` 等上游 run 完成。实践中的坑：`mode='reschedule'` 依赖 scheduler 定期扫描，等待越久 DB 的 `task_instance` 更新越频繁；sensor 等待期间 DAG 被清掉/重跑会重复起 sensor；sensor 探测的表若在探测瞬间存在、随后被覆盖，sensor 通过了但数据还是坏的——就绪校验要看"稳定"，必要时探测两次间隔对比。

**延伸考点：** `ExternalTaskSensor` 与 `ExternalTaskMarker` 搭配时怎么避免"死锁/悬空依赖"？reschedule 模式对 scheduler 和 DB 的压力与 poke 模式怎么量化对比？

---

### Q8. 几十个业务线的 DAG 全在 0 点触发，任务排队、资源被打爆，怎么治理？

**问题：** 公司几十条业务线、上百个 DAG，很多 `schedule_interval` 都设成 0 点，每天 0 点后 executor 队列打满、任务排队 1-2 小时，但白天资源大量空闲。怎么系统性治理？

**期望加分项：**
- 能分层治理：①错峰（调度时间打散）②限流（pool）③扩容/弹性（executor）
- 能说清并行度相关的每一层参数：`max_active_runs_per_dag`、`max_active_tasks_per_dag`、executor 的 `parallelism`、worker 的 `concurrency`
- 会用 pool 按业务/资源类型划分并设容量，保证重要任务不被洪峰挤掉
- 会用量化方法：统计峰值并发任务数 vs 可用 slot 数，找到真正瓶颈是 executor、DB 还是上游
- 能提"调度窗口"治理：按业务 SLA 分级（T+1 早报 vs 一般报表）错开 schedule 时间
- 主动提到"排队但不告警"的监控盲区

**减分项：**
- 只会说"多加点 worker"，不分析为什么全挤在 0 点
- 分不清 DAG 级 concurrency、executor parallelism、worker concurrency 各自管什么
- 不知道 pool/queue 的作用，说不清 pool 满了任务的表现（queued 不执行）
- 无量化，凭感觉扩容
- 不管重要任务的优先级保障

**解答：**
先量化再动手。第一步看数据：统计"0 点后每小时的 queued 任务数 vs 实际 running 数"，确认瓶颈在 executor slot（running 打满、大量 queued）、DB（调度延迟）还是上游（数据没就绪）。第二步错峰：核心矛盾是"所有 DAG 同一时刻起跑"，把 schedule 时间按业务 SLA 分级打散——日报类从 0 点均匀铺到 2 点（0:00、0:15、0:30……），次晨数据类铺到 4-6 点，用 cron 错开而非全 0 点；跨 DAG 依赖中上游先跑下游后跑，本身就是天然错峰。第三步限流保重点：用 pool 把"关键链路任务"与"普通任务"隔离，pool 容量按峰值需求评估（如核心池 40、普通池 20），池满任务在 queued 排队而不是把关键任务挤掉；DAG 级设 `max_active_runs_per_dag`（默认 16 太大，按实际并行需求设 3-5），避免一个 DAG 的多个 run 同时抢占。第四步才是扩容：slot 数 = worker 数 × concurrency，用弹性扩缩容（K8s/celery autoscale）覆盖周期性洪峰。实践中的坑：`parallelism` 全局上限设小了，所有 pool 都饿着，排查时先查 executor 级参数再查 pool；pool 满了任务静默 queued 不告警，要加"queued 超时"监控；错峰不是平均主义，SLA 高的链路依然要排前面，用 pool 权重或队列（queue 分流到专用 worker）表达优先级。

**延伸考点：** pool 与 queue 的区别？"任务 queued 但 running 有空闲"这个状态一般是什么原因（优先级/池容量/weight_rule）？

---

### Q9. 上百个"结构相同、参数不同"的任务，用动态生成 DAG 还是写死？

**问题：** 有 120 张业务表，每张表每天要做同样的"抽取 → 清洗 → 入仓"，只是库名/表名/调度时间不同。你想循环生成 120 个 DAG，但老工程师劝你别这么做，你怎么判断？

**期望加分项：**
- 能客观列出动态 DAG 的代价：parsing 时执行生成逻辑拖慢 scheduler、DAG 数膨胀撑大元数据库、排查困难
- 能说出动态生成与静态写死的折中：同构任务优先"单 DAG + 配置驱动"，DAG 数膨胀要设上限
- 能提 Airflow 2.9+ 的动态任务映射（`expand`/dynamic task mapping）：任务数量由运行时数据决定
- 能谈配置驱动：表清单放配置/DB/Variables，代码不动只改配置
- 主动提到动态 DAG 在"改结构后历史 run 重新解析"上的坑
- 有规模量化：单个 scheduler 能稳定 parse 的 DAG 数有限，要监控 parse 耗时

**减分项：**
- 觉得"动态生成很酷"，无视调度性能与可维护性
- 说不清动态 DAG（生成结构）与动态 task（mapping）是两回事
- 没有配置驱动的概念，只会 for 循环硬编码
- 不考虑 120 个 DAG 对 DB 元数据与 UI 的污染

**解答：**
先分清楚两个层次：动态生成 DAG（在 .py 顶层循环 `globals()[name] = DAG(...)`）和动态任务（运行时 `expand` 出多个 task instance），前者是"结构由代码生成"，后者是"结构固定、实例数由数据决定"。对"120 张同构表"，按规模与变更频率选：方案一（推荐）：一个 DAG + 配置驱动——表清单放 YAML/DB/Variables，DAG 内用 `TaskGroup` 为每张表生成"抽清洗"子链，任务数 = 表数 × 步骤数，一次 run 全量处理，改表只改配置不改代码；方案二：动态生成 120 个 DAG——只有当"每张表调度时间不同、且确实需要独立历史 run 与告警"时才值得，代价是 120 个 DAG 都要被 scheduler parse（每个文件加载 Python 模块有开销）、DB 里 120 份 dag/run 元数据、UI 难管理；方案三（Airflow 2.9+）：`expand()` 动态任务映射——任务在 run 时按运行时数据（如当天要处理的表清单）展开成多个实例，结构固定、数量动态，适合"每日处理表数量会变"的场景。实践中的坑：动态生成 DAG 的代码若在模块顶层查 DB/读外部配置，每次 parse（默认每轮全量）都打一次外部系统，120 个 DAG 就是 120 次调用，scheduler 越来越慢；动态 DAG 上线后，生成逻辑的 bug 会一次性污染所有 DAG，且历史 run 重跑时会用"最新代码解析"，结构变化导致旧 run 任务映射错乱——所以动态生成要限制数量、固定生成逻辑、加单元测试锁定输出。

**延伸考点：** 动态 task mapping（expand）在什么场景会失控（如 expand 出 10 万实例）？`mapped_length` 的预估与调度器压力如何控制？

---

### Q10. 任务明明到点了却迟迟不开始跑，延迟十几分钟，怎么排查调度延迟？

**问题：** DAG 的 schedule 是 0 点，但每天任务实际 start 时间在 0:12-0:25 之间抖动，有时甚至 1 点多。scheduler 看着也活着，任务状态是 queued。怎么一步步定位？

**期望加分项：**
- 能建立排查链路：先区分"scheduler 没调度"还是"executor 没执行"——看任务 queued 的时间点与调度器日志
- 能检查 DAG parse 性能：`dagbag_import_timeout`（默认 30s）、parse 失败率、文件数，parse 慢是调度延迟头号原因
- 能检查 DB 层：Postgres 锁、连接数、慢查询（`sla_miss`、`task_instance` 表扫描）
- 能看资源层：pool 是否打满、executor slot 是否被占、worker 是否健康
- 有指标意识：scheduler 心跳时间、任务从 queued 到 running 的耗时监控
- 能提多 scheduler 场景下的协调问题（重复调度/互斥锁）

**减分项：**
- 上来就说"加 scheduler"，不先定位是哪一段慢
- 只盯 DAG 代码，不看 parse 与 DB
- 不知道 queued 与 scheduled 状态分别意味着什么
- 没有监控指标，全靠肉眼
- 忽略"任务延迟"与"任务时长"的区别（延迟指调度到开始）

**解答：**
先定义延迟：`实际开始时间 - 计划触发时间`，然后拆成两段：`计划触发 → scheduler 把 task 置为 scheduled/queued`，以及 `queued → running`。第一段看 scheduler 侧：最典型的元凶是 DAG parse 过慢——scheduler 每轮（默认每几分钟一次）重新加载所有 DAG 文件，若文件顶层有慢代码（查库、读大文件、动态生成），parse 超时（默认 `dagbag_import_timeout=30s`）后该批 DAG 直接跳过，任务只能等下一轮，于是出现"晚一个 parse 周期"的规律性延迟；查 `airflow dags list-import-errors`、看 scheduler 日志的 parse 耗时，把顶层 IO 全部移进任务内或加缓存。第二段看 executor 侧：queued 打转通常是 slot 不足（parallelism/pool 打满）、worker 心跳失效、或 broker（celery）积压；看 celery 的 flower worker 状态与队列长度，或 K8s 下 pod 创建延迟。第三层查 DB：`task_instance` 表锁、`sla_miss` 等表的慢查询、连接池耗尽，Postgres 的锁等待会直接卡住 scheduler 的调度查询。辅助手段：观察 scheduler 日志里的心跳间隔（`scheduler_heartbeat_sec` 默认 5s），任务级打 `execution_date` 与 `start_date` 差值的指标。实践中的坑：多 scheduler（HA）下若 DB 连接被占满或 advisory lock 争抢，会出现"任务被跳过一轮"；先看"延迟是否有周期性"——周期性说明是 parse 轮次问题，无规律说明是资源或 DB 抖动。

**延伸考点：** `scheduler_zombie_task_threshold` 设太小会误杀长任务，怎么设？任务 stuck 在 queued 且 pool 有空闲，可能是哪些原因？

---

### Q11. DAG 文件越来越多、parse 越来越慢，怎么优化？元数据库要不要定期清理？

**问题：** 平台跑了一年，DAG 文件 300+，scheduler parse 一轮从 1 分钟涨到 8 分钟；同时 Postgres 里 `log`、`task_instance`、`xcom` 表已经几十 GB。你怎么系统优化？

**期望加分项：**
- 能区分"parse 变慢"与"DB 变慢"两个问题分别处理
- parse 优化有具体手段：import 阶段零 IO/零重逻辑、减少动态生成、按业务拆文件、用并行 parse 参数、`min_file_process_interval` 控制轮询频率
- 知道 DB 清理的正确姿势：`airflow db clean`（按表 + 保留天数）或维护脚本，先备份、低峰执行
- 能说清哪些表膨胀影响最大：`log`、`task_instance`（保留重跑能力）、`xcom`、`dag_run`、`sla_miss`
- 有索引与 VACUUM（Postgres）维护意识
- 量化评估：先按表大小排序定位大头，再决定清理策略

**减分项：**
- 只知道"清日志"，不清理真正拖慢调度的表
- 清理前不备份、业务高峰跑，清完锁表把调度搞挂
- 认为 parse 慢只能靠换机器，说不出现有优化手段
- 不先分析"哪张表大、为什么大"，一刀切
- 不知道 `task_instance` 的清理要配合"重跑能力"评估（清掉就没法回看/重跑历史）

**解答：**
两个问题分开治。parse 慢：scheduler 每轮会重新解析所有 DAG 文件，三个层面优化——① 代码层：保证 DAG 文件模块顶层"零副作用"，import 阶段不查 DB、不发请求、不做重计算（最容易被忽略），动态生成逻辑全部移到函数内或加缓存；② 文件组织层：按业务目录组织、减少单文件体积与相互 import，同构 DAG 合并（见 Q9）；③ 调度参数层：`min_file_process_interval` 调大减少无谓的重复 parse，Airflow 2.x 支持并行 parse（`max_parsing_threads` 等）按 CPU 核数配；最后用 `airflow dags list-import-errors` 持续监控 parse 失败。DB 清理：先按大小排序定位大头（`SELECT relname, pg_size_pretty(pg_total_relation_size(relname)) FROM pg_class ...`），通常是 `log`（任务日志行）、`task_instance`（每天数千 run × 365 天）、`xcom`。用 `airflow db clean --tables log --clean-before-timestamp 2025-08-10T00:00:00` 或维护脚本按保留窗口删（如 log 90 天、xcom 30 天、task_instance 180 天），务必先备份、低峰、分批删；清理后执行 `VACUUM ANALYZE` 防索引膨胀。实践中的坑：`task_instance` 别清太狠——它承载"历史 run 重跑与审计"能力，先问业务保留要求；`dag_run` 清理要谨慎，删除 run 会把对应历史调度记录一并抹掉；`airflow db clean` 在任务运行期间删 `log` 会短暂锁表，低峰执行；只清 DB 不优化 parse，过几周又涨回来，两者要同时做。

**延伸考点：** Postgres 连接数对 scheduler 的影响（`sql_alchemy_conn` 池配置）？`airflow db clean` 与直接 SQL 删除在保留语义上的区别？

---

### Q12. 生产用 CeleryExecutor 还是 KubernetesExecutor？怎么选？

**问题：** 公司 Airflow 要从 LocalExecutor 上生产，任务以 Python/Shell/SQL/Spark-submit 为主，峰值并发 200-500 个 task，白天低峰只有几十个。让你选执行器并给出理由。

**期望加分项：**
- 能讲清四种 executor 的适用边界：Sequential（调试）、Local（单机小规模）、Celery（传统主流生产）、Kubernetes（云原生按需）
- 按峰值并发、资源隔离需求、运维人力、是否上云四个维度给决策矩阵
- 能指出 Celery 的运维点：broker（Redis/RabbitMQ）高可用、worker 扩缩容、flower 监控、任务丢失/重复执行
- 能指出 K8s 的代价：pod 启动秒级延迟、镜像拉取、资源配额、成本核算，以及对调度延迟敏感任务的坑
- 有混合意识：KubernetesExecutor 可以只让部分 DAG 走 K8s（`executor` 参数 / queue 路由），与 Celery 共存
- 能给出量化的选型依据：峰值并发 vs 平均负载的比值决定"常驻池"还是"按需起 pod"

**减分项：**
- 只会背"K8s 更现代"，说不清两种方案的运维复杂度差异
- 不知道 Celery 的 broker 是独立故障点，也不提任务可靠性与重投机制
- 不考虑团队运维能力与成本，纯技术选型
- 说不出 K8s 下日志、XCom、代码同步怎么做
- 认为只能二选一，不知道可以混合

**解答：**
先给判断框架：选 executor 的本质是回答"任务进程在哪跑、怎么伸缩、怎么隔离"。四个维度：① 峰值并发 200-500 属于中大规模，Sequential/Local 直接排除；② 资源隔离：多个团队共用、有任务互相抢资源的历史 → K8s 按 pod 隔离更干净；③ 运维人力：Celery 需要自己管 broker（Redis）高可用、worker 池、flower、以及 worker 上的 DAG 代码同步（改代码要同步到所有 worker），K8s 则把"环境一致性"交给镜像/helm 与 git-sync，但引入 K8s 本身的学习成本；④ 成本与延迟：Celery 常驻 worker 池资源利用率高、任务秒级起跑，K8s 按需起 pod 在低峰省钱、但每个 task 冷启动要 10-60 秒（拉镜像 + 调度 pod），对"到点必须准点跑"的日报链路是硬伤。典型决策：若公司已有成熟 K8s、任务以 Python/SQL 短任务为主 → KubernetesExecutor，隔离与弹性最好；若平台还是自建机房的 VM、任务长且多、且已有 Redis → CeleryExecutor 更稳更省；折中做法是默认 Celery，把需要强隔离的 DAG 通过 `executor=KubernetesExecutor`（task 级或 DAG 级）路由过去。实践中的坑：K8s 下任务的日志与 XCom 读取依赖 pod 状态，pod 被回收后 `airflow logs` 会取不到日志，要配持久化日志后端；Celery 下改 DAG 代码后必须同步到所有 worker，忘了同步是最常见的"改了不生效/在部分 worker 上报错"事故源；K8s 下 `resources` 不设 requests/limits 会导致调度（binpacking）混乱，任务互相争抢。

**延伸考点：** K8s 下 pod 被 OOMKilled 时任务的表现？Celery 的 `acks_late` 与任务重复执行的关系，怎么保证幂等？

---

### Q13. 现成 Operator 不满足需求，要自己写一个，怎么写才规范？

**问题：** 公司内部有一个数据质量平台，用 HTTP API 提交校验任务。Airflow 现有 Operator 都不支持，你打算写一个 `QualityCheckOperator` 来调用它，怎么写才符合平台规范、可复用、可测试？

**期望加分项：**
- 能说出继承 `BaseOperator` 实现 `execute()`，参数走 `__init__`，业务逻辑与连接管理分离
- 能把"API 调用"封装成 Hook（继承 `BaseHook`），Operator 只描述"做什么"，Hook 管连接与凭据，其他人写别的 Operator 也能复用
- 能用 `template_fields` 让参数支持 Jinja 渲染（如日期、分区）
- 考虑失败语义：提交失败 vs 轮询校验失败要区分处理、支持重试，执行完要清理临时资源
- 有可测试性：直接构造 `TaskInstance` 调 `execute` 做单测，或用 mock 的 Hook
- 主动提到执行中的坑：不要阻塞式 sleep 轮询、用 `self.log` 记日志、参数校验放 `__init__` 或 execute 开头

**减分项：**
- 在 Operator 里直接写死连接串/凭据
- 不知道 Hook 与 Operator 的分层，所有逻辑塞进一个类
- 不声明 template_fields，参数无法模板渲染，每次改日期要改代码
- 不考虑失败与重试语义，execute 抛异常就完事
- 无法测试（没有 mock 意识、日志全用 print）

**解答：**
规范的扩展分三层：Operator（做什么）、Hook（怎么连）、Sensor（等结果）。第一步写 Hook：继承 `BaseHook`，在 `__init__` 里 `self.get_connection('quality_api')` 拿连接配置（host/port/密码），提供 `submit_check(payload)`、`get_result(job_id)` 方法，统一处理认证、超时、重试——这样连接凭据走平台 Connections 体系，不落代码。第二步写 Operator：继承 `BaseOperator`，`__init__` 里收业务参数（`table`、`rule` 等），声明 `template_fields = ("table", "execution_date_str")` 让参数支持 `{{ ds }}` 渲染；`execute(self, context)` 里通过 `QualityApiHook(self.conn_id)` 提交任务，然后轮询结果——轮询别用阻塞 sleep，可以在 execute 内做带超时的有限轮询（轮询本身占 slot 是可接受的，因为这是任务主体逻辑），超时抛 `AirflowException` 触发 task 重试；提交成功但校验失败要区分：抛出带业务信息的异常，让 `on_failure_callback` 能定位。第三步配 Sensor（可选）：如果"提交即返回、结果异步"，可拆成 Operator（提交）+ `QualitySensor`（继承 `BaseSensorOperator`，`mode='reschedule'` 等结果），把等待从执行中剥离。实践中的坑：Operator 里不要自己 new 一个 HTTP 客户端而不复用 Hook，否则连接配置散落各处无法统一管理；`template_fields` 里的字段在 render 前是字符串，类型转换（如转 int）要在 execute 内做；execute 抛出的异常建议包一层自己的异常类（如 `QualityCheckFailed`），便于告警分类与 `@task` 调用时的可读性；写完必须配套单测——`from airflow.models import TaskInstance; ti = TaskInstance(task=op, run_id='test'); op.execute({'ti': ti, ...})` 即可在本地跑通主链路。

**延伸考点：** Operator 与 TaskFlow `@task` 的取舍？`partial`/`expand` 与自定义 Operator 如何结合做批量校验？

---

### Q14. 上下游 DAG 跨系统依赖（表没就绪就不该跑），怎么做得可靠？

**问题：** DAG-A 负责把数据算好写进 Hive/数仓表，DAG-B 依赖这批表。现在 B 每天按固定时间开跑，偶尔 A 延迟导致 B 用半成品数据出报表。把 B 的时间往后挪能解决吗？怎么做才可靠？

**期望加分项：**
- 能指出"任务成功 ≠ 数据就绪"，按时间硬等是脆的，要用"数据就绪语义"表达依赖
- 能说清两类方案：任务级（`ExternalTaskSensor`/`ExternalTaskMarker` 等 A 的 run 成功）与数据级（`SQLSensor`/`HivePartitionSensor` 查分区存在、校验记录数）
- 能讲 `ExternalTaskSensor` 的 execution_date 对齐问题：跨 DAG 要传 `execution_date_fn` 或 `execution_delta`，否则等错 run
- 能说"数据级就绪优于任务级就绪"：上游 run 成功但数据可能还是不完整，校验记录数/分区大小更稳
- 能提超时与失败处理：sensor 设 timeout，超时告警而非无限等
- 主动提到"上游删掉/改名"导致传感器悬空的问题

**减分项：**
- 只答"把 B 的 schedule 往后挪 1 小时"，不解决上游更晚时怎么办
- 说不出 ExternalTaskSensor 与 SQLSensor 各自的适用边界
- 不知道 execution_date 对齐的坑（等错 run、跨天错位）
- 不考虑超时与告警，sensor 挂死没人发现
- 认为"A 跑成功"就等于"B 可以安全跑"

**解答：**
时间挪位只能提高概率，不能保证正确，真正的解法是把依赖从"时间假设"换成"状态检查"。分两层：如果 A、B 都是自己平台的 Airflow DAG，用任务级依赖——B 里放 `ExternalTaskSensor(external_dag_id='A', external_task_id='write_ok', ...)`，等 A 对应 run 的指定任务成功后 B 才继续；这里最容易踩的坑是 execution_date 对齐：A 与 B 的 data interval 通常一致（都是跑昨天的数），直接等即可，但若调度时间不同（A 每小时、B 每天），必须用 `execution_date_fn` 算出 B 要等的 A 的 run 是哪个，否则会等错 run 或永远等不到；加 `timeout`，超时 Fail 并告警。如果 A 在别的调度平台/或依赖的是"外部系统数据到位"，用数据级依赖更稳：`SQLSensor` 查询目标表分区是否已写、或用 `HivePartitionSensor` 查分区存在性，甚至自定义 Sensor 校验"记录数 >= 基线 && 最近分区时间戳在窗口内"——因为"上游 run 成功"不代表"数据质量达标"，数据级校验才是最终防线。实践中的坑：`ExternalTaskSensor` 对"上游 run 被清掉/重跑"敏感，上游历史 run 被 clear 后 sensor 可能一直等；上游 DAG 改名/删除会留下永远等不到的 sensor（加 timeout 兜底）；两套调度平台之间没有任务状态可查时，只能用数据标记（写就绪文件/标记表），并在就绪标记里带上数据版本号，避免"旧标记被读到"。

**延伸考点：** `ExternalTaskSensor` 与 `ExternalTaskMarker` 组合时如何避免跨 DAG 循环依赖（死锁）？数据就绪校验除了记录数还能验什么（校验和、空值率、主键唯一性）？

---

### Q15. 一次大版本重构，改了 DAG 的任务结构，怎么安全发布上线？

**问题：** 要把某核心 DAG 从"10 个任务的串行链"重构为"分阶段并行"，改了不少 `task_id`、拆了依赖。现在担心：改了 `task_id` 会不会把历史 run 弄乱？正在运行的任务怎么办？怎么灰度发布？

**期望加分项：**
- 能指出核心风险：Airflow 用**最新版 DAG 文件**解析历史 run，改 `task_id`/删任务会让历史 run 的"重跑/查看"行为与当时不一致
- 能讲清楚 `task_id` 改名的影响：历史 task_instance 按旧 id 存储，改名后历史 run 无法精确重跑旧任务（新解析下任务集合变了）
- 有发布流程意识：代码评审、语法校验（`airflow dags list-import-errors`）、CI 单测、pre-commit 渲染
- 有灰度手段：先部署到测试环境跑通，生产用新 DAG id 并行跑对账，或分批切流量
- 能谈代码同步问题：Celery 下所有 worker 必须同步新代码，否则任务在新解析下找不到 operator
- 主动提回滚方案：保留旧版文件可快速回退，DAG 版本与运行实例解耦

**减分项：**
- 觉得"改 task_id 无所谓"，不了解历史 run 的解析机制
- 只做人工"肉眼检查"就上生产，没有 CI 校验
- 不知道 Celery worker 与 scheduler 的代码必须一致
- 不考虑正在运行的 run 在发布瞬间的行为（解析到新旧哪个版本）
- 没有回滚预案

**解答：**
先理解 Airflow 的机制：任务执行时用"当时解析的 DAG 结构"，而历史 run 的**重跑**用的是"当前最新代码"——所以重构后，历史 run 在 UI 上能看，但一旦 clear 重跑，会按新结构执行，可能找不到旧 `task_id`、或跑出和当时完全不同的任务集合，这是最隐蔽的坑。发布流程分四步：第一，CI 把关：MR 里跑 `python -m compileall`/pytest 渲染 DAG（用 `airflow dags list-import-errors` 校验无 parse 错），并对新 DAG 做"任务连通性/依赖完整性"单测；第二，明确"是否真的需要改 task_id"：单纯调整依赖顺序或加任务不影响历史，改名会破坏历史 run 的重跑映射——如果只是参数变化，保留 task_id 用 Variables 控制；第三，灰度：测试环境先跑一个完整调度周期，生产用**新 DAG id**（如 `report_v2`）并行跑 1-2 天做结果对账（行数/金额一致），确认后再切换下游依赖并停旧 DAG——新旧并存期间注意"同一逻辑跑两份"对上游的重复读取压力；第四，同步与回滚：Celery 下所有 worker 先于/同时于 scheduler 更新代码（顺序错会出"调度器已用新结构、worker 还是旧代码"的诡异报错）；发布后保留旧版文件若干天（或放 git 历史可回退），出问题回退代码并 clear 受影响的 run 重跑。实践中的坑：发布瞬间正在运行的 run 会按"新代码解析"，若新代码删了它当前所在任务的 `task_id`，该 run 会被标记为失败/异常——所以大改结构尽量在低峰发布；UI 上"重跑历史 run"前必须让业务确认按新逻辑跑是可接受的。

**延伸考点：** DAG 的 `max_active_runs` 与并行发布两个版本 DAG 时的资源冲突？如何用 `airflow dags test` 在发布前本地/测试环境完整演练一个 run？

---

### Q16. 数据库密码被写死在 DAG 里扫出来，连接与密钥怎么管？

**问题：** 安全扫描发现，有同事把数仓的账号密码直接写在 DAG 文件的常量里，甚至提交到了 Git 仓库。除了"批评一顿"，作为平台负责人你怎么从机制上杜绝这类事？

**期望加分项：**
- 能说清 Airflow 的凭据体系：Connections（连接）与 Variables（变量）默认加密存 DB（Fernet key），并提供"Secrets Backend"对接外部密钥系统
- 能指出 Fernet key 的坑：存在 airflow.cfg，丢失后 DB 里所有加密凭据都无法解密，必须备份；轮换要平滑
- 能给出生产最佳实践：接 Vault / AWS Secrets Manager / GCP Secret Manager，连接/变量不在 DB 明文存储，而是运行时拉取
- 能讲规范：密码不进代码、不进 git、不进镜像；凭据统一走 Connections；敏感变量用 Secret 后端
- 有整改与预防意识：git 历史清理（filter-repo）、扫描 CI 卡点（gitleaks 等）、密钥泄露应急预案
- 主动提 Variables 误用：有人用 Variable 存密码/大 JSON，Variable 有 DB 读取开销且可能在日志/UI 泄露

**减分项：**
- 只会说"改用 Environment Variable"，不知道 Airflow 有 Connections 与 Secrets Backend 机制
- 不知道 Fernet key 丢失的后果
- 没有"凭据不进代码"的工程化约束（CI 扫描、code review）
- 用 Variable 存敏感信息且说不出风险
- 不知道 Vault 后端拉取失败的降级与缓存策略

**解答：**
机制上分三层堵。第一层：凭据入库不落代码——把连接信息（host/user/password 等）配在 **Connections**（`airflow connections add my_db --conn-type postgres ...`），DAG 里只写 `conn_id`；Connections 在数据库中以 Fernet 加密存储，密钥在 `airflow.cfg` 的 `fernet_key`。这里第一个大坑：`fernet_key` 必须备份，丢失/更换会导致所有已存凭据解密失败（数据库能连上但连接取不出来），平台直接瘫痪；轮换时要用 `airflow rotate-fernet-key` 平滑过渡，同时旧密钥要保留。第二层：生产上更彻底的做法是接 **Secrets Backend**——配置 `backend=airflow.providers.hashicorp.secrets.vault.VaultBackend`（或 AWS/GCP 对应实现），Connections/Variables 在运行时从 Vault/云密钥服务拉取，DB 里不存明文，密钥生命周期（轮换、吊销）交给专业系统管理；注意要处理"拉取失败"的降级与本地缓存，避免 Vault 抖动导致全平台任务挂。第三层：工程化约束——CI 里跑密钥扫描（gitleaks/trufflehog）作为 MR 卡点，git 历史里已泄露的用 `filter-repo` 清理并轮换相关凭据；Code Review 约定"DAG 里不允许出现任何凭据字面量"。实践中的坑：有人图省事用 **Variables** 存密码——Variables 存在 DB 里、默认每次引用都可能查库（有缓存但语义不清）、且 UI/API 可读，容易在日志里被打印出来，敏感信息一律走 Connections + Secret Backend；不要给所有业务线共用一把 Fernet key 或一个 admin 连接账号，按团队拆分连接并最小授权。

**延伸考点：** Fernet key 轮换的具体步骤与风险点？Vault 后端与本地缓存的配置项（如 `auth_type`、`mount_point`）怎么设才能兼顾安全与可用？

---

### Q17. 每天 0 点后大批任务失败，且集中在"数据库连接"类任务，怎么定位？

**问题：** 上线新 DAG 后，每天 0 点开始有一大批任务失败，报错集中在"连接超时/连接被拒"，涉及不同业务线；白天手动重跑却大多能成功。你怎么排查根因？

**期望加分项：**
- 有"洪峰 + 共享资源"的意识：0 点是全局调度高峰，先怀疑"并发连接数被击穿"而非单个 DAG 的 bug
- 能先量化：0 点前后连接数曲线 vs 数据库 `max_connections` 对比，确认是连接数耗尽还是网络抖动
- 能查共享组件：DB 连接池、连接白名单/防火墙、VPN、代理的连接上限
- 能分析"为什么白天重跑能成功"：白天并发低，连接空闲释放，佐证"连接耗尽"假设
- 能给长效解法：任务内连接复用/用完即关（with 语句）、限制 DAG 并发、错峰、扩连接配额、连接池化
- 能提可观测性：任务失败率按时间段/连接名聚合的监控，出事时能 5 分钟定位

**减分项：**
- 只修单个任务（重试），不查共享瓶颈
- 不知道 Hook 默认每次 get_conn 新建连接、用完不关会累积
- 只盯着报错文字，不看时间与并发维度
- 没有监控，靠出事后再看日志
- 直接调大数据库 max_connections 了事，不治理连接使用方式

**解答：**
先立假设：0 点 + 大批 + 跨业务线 + 白天重跑成功，四个特征指向"共享资源洪峰"，最典型的是**数据库连接耗尽**。定位三步：第一，拉 0 点前后 1 小时的连接曲线——数据库侧看 `pg_stat_activity`（Postgres）/`SHOW PROCESSLIST`（MySQL）的活跃连接数与 `max_connections` 对比，若触顶且大量 `idle in transaction`/`waiting`，基本实锤；第二，确认连接从哪来——按客户端/账号聚合，通常发现是某个新 DAG 用了大并发（如动态生成了 100 个任务，每个都开一个连接），或某任务没关连接（Python 里 `conn.cursor()` 未 close、`hooks.get_conn()` 用完没释放，Airflow 的 `DbApiHook.get_conn` 默认每次新建，池化仅在 provider 内部）；第三，验证重跑能成功的原因：白天并发低、连接空闲释放，连接数回落。修复分两层：治标——0 点错峰、`max_active_runs` 限流、任务内 `with conn:` 用完即关，连接数立刻下降；治本——数据库连接配额按业务线拆分（云 RDS 的 connection pooler，如 PgBouncer）、关键 DAG 用连接池（`hook.get_conn()` 的池化配置）、把"同时打开的连接数"打点上报。实践中的坑：只调大 `max_connections` 会掩盖问题，连接耗尽时新请求排队，任务重试又把并发推高，形成"重试风暴"；排查时要区分"连接超时"（网络层）与"连接被拒"（已达上限）两类报错，前者还要查防火墙/白名单是否在 0 点有变更；这类问题必须提前有"失败率 × 时间段 × 错误类型"的聚合监控，否则每次都要等告警了才翻日志。

**延伸考点：** Airflow 的 Hook 连接池在哪层配置（`sql_alchemy_conn` vs provider 的 pool）？"重试风暴"怎么避免（重试上限与退避、错误类型分流）？

---

### Q18. 两个部门共用一套 Airflow，互相影响、权限混乱，怎么治理？

**问题：** 公司一套 Airflow 同时服务数仓和算法两个部门，算法那边开大并发任务时数仓的日报经常被挤掉；两边还能互相看到/修改对方的 DAG、连接和变量。你作为平台负责人怎么治理？

**期望加分项：**
- 能分层治理：资源隔离（pool/queue/独立 worker）+ 权限隔离（RBAC + DAG 级权限）+ 配置隔离（连接/变量按团队命名或独立）
- 能说清 Airflow 的 RBAC 模型：默认 `auth_backend` 下的 roles 与 DAG 级权限（`dag.can_edit`/`can_read`），按团队建 role 授权
- 能讲 queue 与 pool 的配合：queue 路由到独立 worker 池（物理隔离），pool 限流保证关键 DAG 有保底 slot
- 有量化：每个部门评估峰值并发，按配额分配 pool 容量，超配额排队而非互相抢占
- 能提审计与治理：DAG 变更审计、连接/变量访问最小化、敏感 DAG 只有 owner 可改
- 主动提到"共享 vs 独立"的成本权衡：完全独立部署的运维成本 vs 共享平台的治理成本

**减分项：**
- 只会说"分开部署两套"，不考虑成本与运维负担
- 不知道 RBAC 与 DAG 级权限的具体配置方式
- 只有 pool 概念没有 queue/worker 物理隔离的概念
- 不量化配额，pool 容量拍脑袋
- 不管连接与变量的权限，任何人能看明文/改配置

**解答：**
先明确治理目标：互不饿死（资源）、互不可见（权限）、变更可控（审计）。第一层资源隔离：给两个部门分配独立 **queue**（如 `queue='dw'` / `queue='algo'`），分别路由到独立的 celery worker 池（或 K8s 独立 namespace），物理隔离保证算法开大并发时挤不到数仓的 worker；同一 worker 池内再用 **pool** 限流——为"数仓日报链路"设核心池并配足容量（按历史峰值 × 1.5 估算），关键 DAG 的 `pool='dw_core'`，普通任务放共享池，池满排队而不是抢占；这样"保底 + 弹性"都有了。第二层权限隔离：启用 RBAC（`[webserver] rbac = True`，用 `auth_manager`/`security_manager` 配置），创建部门 role（如 `DW-Admin`、`ALGO-Viewer`），按 DAG 粒度授权——通过 `airflow roles` 命令或 DAG 的 `access_control` 参数声明"谁能编辑/阅读哪些 DAG"；connections 与 variables 的权限按 role 收敛（Airflow 2.x 可控制谁读写），避免算法看到数仓的连接明文。第三层治理机制：DAG 的变更走 review + CI；开启审计日志（`[logging]` 审计插件或 UI 操作日志），对"谁改了什么 DAG/连接"留痕。实践中的坑：只做 pool 不做 queue，资源隔离不彻底——pool 只是调度排队，任务还是跑在同一个 worker 上，大并发任务照样把 worker 的 CPU/内存吃满；RBAC 默认只分 Admin/User/Op/Viewer 四个粗粒度 role，必须自定义 role 并按 DAG `access_control` 收紧，否则"User"能看所有 DAG；共享平台必然有治理成本，要定期评审配额与权限，别等出事再排查。

**延伸考点：** DAG 的 `access_control` 参数如何与 role 结合实现"只读/可编辑"两级权限？K8s 部署下用 namespace + ResourceQuota 做物理隔离与 queue 路由怎么配合？

---

### Q19. 老板要求 Airflow 平台高可用、故障分钟级恢复，架构怎么搭？

**问题：** 调度平台已成为公司数据主链路，老板要求"单组件故障不影响主链路、恢复目标 RTO 30 分钟内"。现有单机部署（一个 scheduler + 一个 webserver + SQLite）。你给出改造方案。

**期望加分项：**
- 能先指出 SQLite 单机是最大单点，第一步必须换 Postgres 并做主备/云 RDS
- 能讲清 Airflow 2.x 的 HA 架构：多个 scheduler（依赖 DB 锁协调）、多 webserver + 负载均衡、executor 侧 worker 池
- 能列出所有组件与对应 HA 手段：DB（主备/托管）、broker（Redis 主从/集群）、日志（集中存储）、DAG 代码（Git/共享存储/镜像）
- 有故障演练与验证意识：kill 一个 scheduler/worker 验证任务不中断、重跑机制兜底
- 能讲幂等与"任务重投"语义：celery 的 `acks_late`、任务幂等，保证故障重跑不产生脏数据
- 有可观测性：健康检查、指标、告警，故障时能快速定位是哪一层的组件挂了

**减分项：**
- 不先解决 SQLite，直接加机器
- 不知道多 scheduler 依赖数据库协调（advisory lock），以为无脑多开
- 遗漏 broker/日志/DAG 代码等"隐藏单点"
- 没有故障演练，方案只停留在 PPT
- 不考虑故障期间"正在运行的任务"与"重试/重投"的语义

**解答：**
先分层列组件，把每个单点都消灭掉。第一层 DB：SQLite 换 **Postgres**（这是硬前提），生产用云 RDS 主备（自动故障切换）或自建 Patroni 集群；这是所有 scheduler/webserver 共享的状态源，DB 挂则全平台挂。第二层 scheduler：Airflow 2.x 支持**多 scheduler 并行**，它们靠 Postgres 的 advisory lock 抢"谁是 leader"（调度主循环）与行锁协调任务分发，开 2-3 个 scheduler 进程，单个挂掉其余接管；注意 `scheduler_heartbeat_sec`、`max_threads` 等参数要与 DB 能力匹配，避免争抢锁拖慢调度。第三层 executor：Celery 起 worker 池（多节点），broker 用 Redis 主从/哨兵（或云托管），worker 挂掉任务会重投（配 `acks_late=True` 避免任务丢失，配合任务幂等防止重复执行产生脏数据）；K8s 部署则天然多副本。第四层 webserver：无状态，多实例 + 负载均衡即可。第五层"隐形单点"：日志默认在 worker 本地文件，改配集中日志（S3/ES/Loki）；DAG 代码放 Git，部署用共享文件/镜像/git-sync 同步到所有 scheduler 与 worker，避免"部分节点代码不一致"这个最隐蔽的故障源。最后必须做**故障演练**：定期 kill 一个 scheduler、一个 worker、模拟 DB 切换，验证主链路任务不中断、告警能定位；给 scheduler/webserver 配健康检查（`/health` 端点），接监控与告警。实践中的坑：多 scheduler 不等于多"调度能力翻倍"，调度瓶颈常在 DB（锁、连接、索引），先保证 Postgres 规格与索引（task_instance/dag_run 表）够用再谈加 scheduler；任务幂等是 HA 的地基——任何一次组件故障都会触发重投/重跑，任务不可重入的话，HA 做得越好脏数据越严重。

**延伸考点：** 多 scheduler 下"任务被两个 scheduler 同时调度"怎么避免（行锁 + `most_recent_dag_run` 语义）？DB 主备切换瞬间正在运行的 run 会怎样，如何让 RTO 达标？

---

### Q20. 公司要做统一调度平台，用 Airflow、DolphinScheduler 还是自研？开放性题

**问题：** 你们平台团队被要求建设公司的"统一任务调度平台"，承载数仓批处理、算法训练、业务定时任务共三类负载，日调度量预计从几百增长到数万。技术选型你怎么做？如果最终选了 Airflow，最大的风险是什么、你怎么补？

**期望加分项：**
- 先列需求与约束再选型，而不是先入为主：任务类型（SQL/Python/Shell/分布式计算）、规模量级、SLA、团队运维能力、是否上云
- 能对比主流方案：Airflow（Python 可编程、生态大、UI/运维弱）、DolphinScheduler（可视化、中文社区、自带告警，但生态与灵活性有限）、自研（成本极高，除非需求特殊）
- 有量化：数万日调度 = 每分钟约 7 个 run，要考虑 scheduler 吞吐与 DB 写入，评估"是否超出 Airflow 舒适区"
- 能指出 Airflow 的已知边界：调度延迟（秒级到分钟级）、scheduler 吞吐上限、元数据库压力、多租户弱，并给出缓解（HA、清理、分层）
- 有平台化思维：抽象统一"任务描述/触发/执行"模型，执行层可替换，避免被单一引擎绑架
- 主动提"渐进式"：先小规模试点再全量迁移，双跑验证

**减分项：**
- 不分析需求直接说"用 X"，或反过来无脑自研
- 只背 Airflow 优点，说不出它的性能与运维痛点
- 没有量级概念，把"数万日调度"当小菜
- 不考虑团队人力与长期维护成本
- 没有迁移与双跑意识，一上来就全量替换

**解答：**
先立方法论：选型是"需求 → 约束 → 打分"的过程，不是信仰问题。先量化三类负载：数仓批处理（T+1，分钟级调度延迟可接受，量最大）、算法训练（长任务、资源隔离要求高、量小）、业务定时任务（要求准点、可能要求秒级触发）。约束：团队是数据团队（Python 熟）还是运维团队、是否上云、运维人力几人。对比：Airflow——Python 定义 DAG 灵活、算子生态全、社区大，但调度延迟秒级到分钟级、scheduler 吞吐有上限（单 scheduler 稳定支撑的并发任务有限）、元数据库要精心维护、多租户与权限弱；DolphinScheduler——DAG 可视化拖拽、自带告警与多租户，适合"运维/非 Python 团队 + 强可视化诉求"，但扩展生态、编程灵活性弱于 Airflow；自研——只在"毫秒级调度、百万级任务、强定制"等 Airflow 无法覆盖时才考虑，成本是 2-3 年持续投入。综合来看，绝大多数公司选 Airflow 是对的，但要接受并补齐它的三个风险：① **吞吐与延迟**：数万日调度下，用 HA 多 scheduler、按业务拆独立 Airflow 实例（隔离 + 分治）、优化 parse 与 DB，把每实例负载控制在舒适区；② **运维与多租户**：用 RBAC + queue/pool 做租户隔离，搭好监控告警、日志集中、DB 清理等平台配套，缺了这些 Airflow 就是"能用但难养"；③ **执行形态单一**：如果业务任务类型差异极大（既有 SQL 又有 GPU 训练），在 Airflow 之上抽象统一的任务描述模型（任务元数据 + 触发器），Airflow 只做其中一类执行器，为将来演进留接口。最后，落地走"小规模试点 → 双跑对账 → 分业务线迁移"，任何选型结论都要用试点数据验证，而不是 PPT 拍板。

**延伸考点：** 如果选型落在 DolphinScheduler，你会在什么条件下反转选择？"数万日调度"这个量级下，Airflow 的 scheduler 吞吐瓶颈具体体现在哪（parse、DB 写、task 扫描），怎么量化评估？
