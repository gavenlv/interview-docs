# 云 · AWS（面试题库）

本文件考察候选人在 AWS 平台工程落地上的真实能力：全球基础设施（Region/AZ/边缘节点）、核心计算 EC2、容器编排与 Serverless、存储与数据库选型、VPC 网络与负载均衡、IAM 权限与加密安全、监控告警、IaC 与 CI/CD、消息与边缘计算，以及成本优化、Well-Architected、迁移工具与认证知识面。题目全部为场景化提问、不考八股文——重点看候选人能否给出具体服务配置、CLI 命令与量化数据，说清"为什么选它、不选它"的取舍，引用线上实践与踩坑案例，主动覆盖边界条件与降级方案，难度从实践基础渐进到架构级设计。

---

### Q1. 新业务要上 AWS，Region 和 AZ 怎么选才不后悔？

**问题：** 你们要把一套面向东南亚用户的在线支付风控系统部署到 AWS，数据合规要求数据不能出境（新加坡监管）、核心延迟要求小于 100ms、还需要跨可用区容灾。你会如何选择 Region 和 AZ，并说明哪些服务具备区域性、哪些具备可用区级冗余？

**期望加分项：**
- 讲清三层基础设施模型：Region 是隔离的地理单元（可用区至少 3 个、相距 60 英里内），AZ 是独立电力/网络/机房的可用区，边缘节点（CloudFront/Global Accelerator）用于缓存与就近接入，不承载计算
- 给出去重选区域的具体考量清单：合规与数据驻留（如新加坡个人数据保护法）、延迟就近（东南亚用户）、服务可用性差异（部分新服务只在特定 Region 上线，如某些 AI 服务仅 us-east-1）、成本差异（同样配置在不同 Region 价差可达 20-30%）
- 强调"数据主权"是硬约束：先定 Region 再谈架构，例如新加坡业务强制用 ap-southeast-1，宁可牺牲一点成本也不能跨境
- 量化 AZ 设计：生产系统至少双 AZ、核心系统三 AZ，RDS Multi-AZ 与 EBS 快照等冗余机制都按 AZ 维度设计，单 AZ 是单点故障
- 有落地验证手段：用 Global Accelerator 或 CloudFront 测真实链路，用 `aws ec2 describe-regions` 查看开放区域，参考 AWS 官方区域服务表核对所需服务在该 Region 是否可用
- 提到边缘与回源的关系：边缘节点只做就近接入，真正容量规划按 Region 峰值计算，边缘回源带宽与延迟要实测

**减分项：**
- 只说"就近选区域"，不提合规驻留这一硬约束
- 把 Region/AZ/边缘节点混为一谈，说不清各自故障域
- 不知道部分服务仅少数 Region 提供，选完才发现目标服务在该区域没有
- 单 AZ 部署还宣称"高可用"，没有 AZ 维度冗余概念
- 没有延迟/成本/合规的权衡过程，直接拍脑袋

**解答：**

选 Region 遵循"合规 > 延迟 > 成本"的优先级。第一步先确认数据驻留要求：新加坡金融监管要求数据不出境，直接锁定 ap-southeast-1，这一步没有商量余地；第二步验证延迟，东南亚用户主要在新加坡、印尼、马来，ap-southeast-1 到三地 RTT 均在 30-60ms 内，满足 100ms 要求，再用 Global Accelerator 在边缘层做就近接入，把首包延迟再压一截；第三步核对服务清单，确认要用到的服务（如 Managed Streaming for Kafka、SageMaker）在 ap-southeast-1 全部可用——我遇到过需求方点名要用某个只在 us-east-1 开放的新 AI 服务，最后只能妥协为双区域部署。AZ 层面，风控系统要求跨 AZ 容灾：应用层在两个 AZ 各起一套 ASG，RDS 开 Multi-AZ（主备跨 AZ，切换 RTO 以分钟计），DynamoDB 默认三 AZ 复制，S3 是区域级服务天然多 AZ。这里最常见的坑是"单 AZ 起步、后来补容灾"——EBS 卷绑死 AZ，跨 AZ 迁移只能重建实例或做快照迁移，代价极大。我的经验是生产环境第一天就按双 AZ 设计子网与安全组，宁可前期多花一点实例费，也不让架构被单 AZ 锁死。Region 选错是"终身买单"，AZ 设计错了是"迁移痛苦"，决策顺序不能反。

**延伸考点：** 如果核心服务在目标 Region 未开放，有什么折中方案（邻近 Region、双区域异步复制、PrivateLink 跨区域访问）？各自代价如何？

---

### Q2. 电商大促峰值 10 倍流量，EC2 实例与购买模式怎么组合最划算？

**问题：** 你们一个电商业务日常 100 台 t3.medium 扛得住，大促峰值要 1000 台且持续 8 小时，用 3 个月。CTO 要求成本可控又不能影响性能，你会怎么选实例类型族和购买模式？安全组和 SSH 访问如何配套设计？

**期望加分项：**
- 讲清实例族定位：通用型（m/t 系）平衡、计算型（c 系）适合 CPU 密集、内存型（r/x 系）适合缓存与数据库、存储优化型（i/d 系）适合高 IO；结合负载特征选择而不是无脑用默认型号
- 购买模式组合策略：基线流量用按需或 Savings Plan 兜底，峰值波动用 Spot 填充——大促 8 小时峰值用 Spot 可省 60-90%，配合 ASG 的混合实例策略（on-demand 基线 + spot 扩容）与容量预留（Capacity Reservations）
- 给出量化算账：100 台基线 t3.medium 用 1 年 Savings Plan 比按需省约 30-40%；峰值 900 台 Spot 按现货价的 30-40% 算，大促整体算力成本能压到按需方案的 50% 以下
- 提到 Spot 的坑：Spot 会被回收（2 分钟告警），必须有"可中断"的任务设计（无状态应用、结果落 S3/SQS 重放），数据库等有状态服务绝不上 Spot；用 ASG 的 Mixed Instances 与 Capacity-Optimized 策略分散 Spot 中断风险
- SSH 与安全组最佳实践：不用默认 key 直连生产、用 Session Manager（SSM）替代 SSH、安全组最小化（只放 22/443 到堡垒机网段）、实例改用 IAM 角色而非在实例里放 AK/SK
- 有镜像意识：用 AMI 固化基线（含补丁与 Agent），配合 Launch Template，大促扩容 1000 台也能分钟级拉起

**减分项：**
- 只会按需购买，不知道 Savings Plan/Spot/预留实例三者的差别与适用场景
- 把有状态服务（MySQL、Redis）直接部署在 Spot 上，大促中途被回收
- 实例族选型无依据，CPU 密集任务买 m 系、内存敏感任务买 t 系
- 安全组配置过宽（0.0.0.0/0 全开 22 端口），SSH 密钥明文放在代码仓库
- 说不清 ASG 混合实例策略与 Spot 中断处理机制

**解答：**

我的组合思路是"三层供给"：基线流量（100 台）用 1 年期 Savings Plan 兜底，日常波动用按需补足，大促峰值全部用 Spot。算账逻辑：100 台 t3.medium 按需每月约 3000 美元，Savings Plan 能砍到 2000 出头；大促 8 小时峰值 900 台 Spot，按现货价约 35% 算总花费远低于按需拉满。技术前提是应用必须无状态化：会话放 ElastiCache、上传文件走 S3、订单消息落 SQS，实例只是"计算工蜂"，这样 Spot 回收（EC2 会提前 2 分钟发 Rebalance Recommendation）时 ASG 直接替换，不会有数据丢失。我会给 ASG 配 Mixed Instances：on-demand 设 15-20% 兜底、其余 Spot，策略用 Capacity-Optimized 并混入多个实例族（如 c5.large/c6i.large/m5.large），避免单一型号现货被抢空导致扩容失败。配套的安全基线：安全组只放 22 端口到堡垒机网段，生产禁用密码登录，运维统一走 SSM Session Manager（不暴露 22 端口、审计全程留痕），实例挂 IAM 角色拿 S3/SQS 权限，彻底去掉 AK/SK 落盘。最后用 Launch Template + 预置 AMI（装好监控 Agent 和依赖）保证 1000 台 5 分钟内全部就绪。这样大促成本约为全按需方案的 40%，且具备完整的可观测与可审计链路。

**延伸考点：** Spot 实例被回收时 ASG 如何优雅缩容（Rebalance Recommendation 与实例生命周期钩子）？异步任务如何保证不丢消息？

---

### Q3. 微服务上容器，ECS、EKS 和 Fargate 到底选哪个？

**问题：** 你们有一个 20 人团队、两套微服务系统：一套轻量 API（无状态、一天几个调用高峰）、一套核心交易链路（需要精细调度、GPU 批处理、跨账号共享集群）。团队没有专职 K8s 运维，CTO 问"ECS、EKS、Fargate 怎么选、和自建 K8s 比哪个合适"，你会怎么回答？

**期望加分项：**
- 给出决策框架：管理成本（托管 vs 自建 vs 托管控制面）、调度能力（K8s 生态）、计费模型（EC2 计费 vs Fargate 按 vCPU/内存计费）、团队运维能力四个维度综合判断
- 场景化结论：轻量 API 用 ECS on Fargate（无节点管理、按秒计费、秒级伸缩），核心交易链路且要 K8s 生态（Helm、Operator、网络策略）用 EKS，控制面托管但工作节点自管（EKS Managed Node Groups），GPU 批处理用 EKS + NodeGroup 挂 GPU 实例
- 讲清 Fargate 的边界：没有 DaemonSet/privileged 容器、hostNetwork 受限、弹性网卡配额限制（每任务一张 ENI、Pod 密度受限），选型前要核对特性兼容性
- 对比自建：自建 K8s 要自己维护 etcd 高可用、控制面升级、CNI/CSI 插件、节点打补丁，小团队通常不值；但超大规模（上万节点）或深度定制（自研调度器、专用 CNI）时自建仍有价值
- 有 EKS 落地经验：eksctl/terraform 建集群、IRSA（IAM Roles for Service Accounts）做 Pod 级权限、NodeGroup 的 AMI 升级与原地更新、集群升级（1.24→1.25 这类）的 API 兼容性检查
- 提到成本与容量：EKS 控制面免费（只收节点与负载均衡费）但加 Fargate 每 Pod 有约 20% 附加费；大流量下 ECS EC2 模式比 Fargate 便宜，Fargate 适合低流量高弹性的负载

**减分项：**
- 直接"无脑上 EKS"，不评估团队 K8s 运维能力
- 不知道 Fargate 的限制（无 DaemonSet、ENI 配额），选了之后才发现跑不了
- 说"自建 K8s 更省钱"，却不算 etcd/控制面/升级的人力维护成本
- 说不清 ECS 与 EKS 在调度模型、服务发现、Ingress/ALB 接入上的差异
- 没有计费量化：Fargate 的附加费、EKS 的节点费、自建的三倍人力成本

**解答：**

我先抛决策框架：看四个维度——团队 K8s 能力、调度需求、计费模式、生态依赖。轻量 API 那套，20 人团队没有专职 SRE，直接 ECS on Fargate：没有节点要管理、按 vCPU 和内存按秒计费、配合 App Runner/CodeDeploy 蓝绿发布，日均调用量低时成本比常驻 EC2 省一半以上。核心交易链路那套，需要 K8s 生态（Helm 部署、Istio/NetworkPolicy、GPU 批处理、跨命名空间 RBAC），选 EKS：控制面 AWS 托管（etcd 高可用、控制面升级免运维），工作节点用 Managed Node Groups 加 Spot 混合降低成本；Pod 权限用 IRSA 把 IAM 角色映射到 ServiceAccount，比在节点上堆权限干净得多。和自建 K8s 对比要算总账：自建意味着自己扛 etcd 三节点、API Server 高可用、CNI 升级、节点补丁与安全通告，这些运维工作量折合成人力远高于 EKS 的托管费；只有上万节点规模或深度定制（自研调度器/专用 CNI）才值得自建。一个常见的坑是"Fargate 看似全能"：它不支持 DaemonSet 和 privileged 容器，日志采集（Fluent Bit DaemonSet）在纯 Fargate 上跑不了，得改用 Fargate 的 sidecar 模式或 firelens；另外每 Pod 一张 ENI，单节点 Pod 密度受 ENI 配额限制。所以我的最终结论：轻量弹性负载用 Fargate，需要 K8s 生态用 EKS，避免为了"统一技术栈"强行把所有负载塞进同一套编排。

**延伸考点：** EKS 升级时如何评估 API 废弃与兼容性（kubectl convert、升级前审计清单）？IRSA 与节点级 IAM 权限各自的适用边界？

---

### Q4. 一个 API 半小时内从 0 打到 5000 QPS，Lambda 扛得住吗？

**问题：** 你们要做一个图片缩略图处理服务：用户上传原图到 S3，触发处理生成多尺寸缩略图，峰值时半小时内从 0 打到 5000 QPS。你打算用 Lambda + S3 事件触发实现，问 Lambda 的冷启动、并发配额怎么破？

**期望加分项：**
- 讲清触发链路：S3 事件 → Lambda（同步/异步）→ 处理结果回写 S3，S3 事件是异步投递，消息量大时要考虑事件风暴与幂等（事件至少一次投递、需要幂等处理）
- 冷启动优化三板斧：运行时选对（Node/Python 冷启动快，Java/.NET 慢）、预留并发（Provisioned Concurrency，预初始化容器消除冷启动）、依赖精简与层（Layer）复用、代码包尽量小
- 明确并发模型：Lambda 并发配额是"按账号分 Region 的软限制"，默认 1000，可通过配额工单提升；5000 QPS 若单次 100ms 处理时长需要 500 并发（5000×0.1），要预留并发或拆分函数
- 处理函数规模限制：内存 128MB-10GB（配 vCPU 比例）、超时最长 15 分钟（同步调用）、部署包 50MB（zip）/250MB（含层）、临时磁盘 /tmp 512MB-10GB——超过限制要换方案（容器镜像/Step Functions）
- 异步调用与重试：异步触发失败自动重试 2 次、死信队列接 SQS/DLQ；S3 事件触发是异步，Lambda 返回值不回传给上传方，需要回调或 WebSocket
- 结合其他服务：大图用 S3 事件直接触发可能跟不上，可先落 SQS 削峰（事件风暴保护），Lambda 从 SQS 批量拉取（BatchSize + 可见性超时），幂等去重

**减分项：**
- 不知道 Lambda 有并发配额和冷启动，拍胸脯保证 5000 QPS 没问题
- 无脑用 Java/.NET 实现低频冷启函数，P95 延迟被冷启动拖到几秒
- 处理超过 15 分钟的任务硬塞进 Lambda，不知道 Step Functions 或容器方案
- 不做幂等，S3 事件至少一次投递导致缩略图重复生成、费用翻倍
- 不评估同步调用在 API Gateway 场景的超时与重试语义

**解答：**

这个场景 Lambda 完全扛得住，但前提是把三个坑填平。第一是冷启动：图片处理函数用 Python 3.12 或 Node 20 运行时，依赖（Pillow 等）打进 Layer 并预编译，代码包控制在几 MB；对 P95 延迟敏感的核心链路开 Provisioned Concurrency 预留 100-200 并发，冷启动直接从几百 ms 压到近 0。第二是并发配额：并发不是"每秒调用数"，而是"同一时刻运行中的实例数"，5000 QPS 且单次处理 200ms 意味着需要约 1000 并发，默认配额 1000 可能直接触顶，提前通过 Service Quotas 工单提到 5000-10000，并给函数设并发上限（Reserved Concurrency）防止被别的函数挤占。第三是事件风暴：S3 一次上传触发多个对象事件、批量上传瞬间产生海量事件，直接触发 Lambda 会打满配额还重复消费——我在中间插一个 SQS 队列削峰，S3 事件进 SQS，Lambda 按 BatchSize=10、批窗口 5s 批量消费，配合消息去重（MessageDeduplicationId）保证幂等；处理失败进 DLQ，用死信队列做人工补偿。函数本身注意三个硬限制：内存最大 10GB、超时 15 分钟、/tmp 最大 10GB，缩略图这种秒级任务毫无压力，但如果未来要做视频转码（超 15 分钟）就得切 Step Functions 编排 + Batch/ECS 容器。最后用 CloudWatch 监控 Throttles、IteratorAge 和 DLQ 深度，缩略图场景端到端 P95 可以稳定在 1s 内。

**延伸考点：** Lambda 与 API Gateway 同步调用时，超时与重试如何配置才合理？Provisioned Concurrency 的计费与释放策略？

---

### Q5. 日志、备份、静态资源、大数据分析，四类数据分别放 S3 还是 EBS/EFS？

**问题：** 你们有这四类数据：①业务日志（每天 200GB、保留 90 天、偶尔要检索）；②数据库备份（每天全量 + 增量，保留 1 年）；③Web 静态资源（图片/JS，全球用户访问）；④K8s 集群多 Pod 共享的模型文件（几十 GB，读多写少）。你会怎么设计存储方案，S3/EBS/EFS/Glacier 怎么分工？

**期望加分项：**
- 讲清四类存储的本质差异：S3 是对象存储（区域级、11 个 9 持久性、海量扩展、通过 API 访问），EBS 是块存储（挂载单实例、低延迟、随 AZ），EFS 是文件存储（NFS 协议、多实例共享、随 VPC）
- 场景映射：①日志和③静态资源放 S3（加生命周期策略），②备份放 S3 后按时间递降存储类（S3 Standard → S3-IA → Glacier Instant Retrieval → Glacier Deep Archive），④K8s 共享模型用 EFS（CSI 驱动挂载）或 S3（只读时用 S3 CSI + 缓存）
- 生命周期策略要量化：日志 30 天后转 S3-IA（成本约为 Standard 一半）、90 天后删除；备份 30 天内保留 Standard（用于快速恢复）、之后转 Glacier Deep Archive（$0.00099/GB/月，约为 Standard 的 1/20），用生命周期规则自动流转而不是手动迁移
- 讲清版本控制与防护：S3 开版本控制防误删（配合生命周期清理旧版本）、开 MFA Delete、开启 S3 复制做跨区域容灾（CRR）——曾用版本控制救回被勒索软件覆盖的桶
- 有性能与成本的量化意识：S3 单对象 GET 性能足够（PUT/GET 可达数千 QPS，加前缀分散分区键），热数据不要放 Glacier（取回要等几分钟且有取回费）；EFS 有 Throughput Mode 与 Bursting 两种模式，读多场景用 Max I/O 或 Provisioned Throughput 避免突刺
- 提到 K8s 集成的坑：EFS 有 POSIX 锁与目录缓存问题，高并发小文件读写性能差，写密集场景应换 FSx 或改 S3 批量对象读写

**减分项：**
- 把 EBS 当"放数据的地方"，不知道 EBS 绑定 AZ 且不能多实例共享
- 不知道 S3 存储类分层的价格差异与取回时间，日志直接进 Glacier 导致检索等 12 小时
- 没开版本控制，被覆盖/删除的数据无法恢复
- 备份只有一份在同一区域，Region 级故障直接全没
- 说不清 S3 与 EFS 各自的访问模型（API vs NFS）与适用负载

**解答：**

我的原则是"按访问模式分桶，不搞一刀切"。①日志：直接灌 S3 标准存储，用 S3 生命周期策略 30 天转 S3-IA、90 天过期删除，检索走 Athena 直接查 S3（分区按年/月/日建表），省掉自建 Elasticsearch 的运维；②数据库备份：RDS 自动备份 + 手动快照导出 S3，生命周期 7 天 Standard（快速恢复窗口）、30 天转 S3-IA、90 天后转 Glacier Deep Archive 保留满 1 年——Deep Archive 取回要 12 小时，但合规审计场景可接受，单 GB 成本不到一美分，一年 1TB 备份成本才几美元；③静态资源：S3 挂 CloudFront，版本号命名（style.v2.css）配合缓存策略，命中率 90% 以上，S3 本身只承担回源流量；④K8s 共享模型：几十 GB 读多写少，用 EFS 走 CSI 驱动挂载最省事（多 Pod 并发读、无需改代码），但要注意 EFS 对小文件高并发写的性能陷阱，我会先把模型文件打成大 tar 包或改走 S3（Pod 启动时预热到本地卷）。几个必须讲的坑：EBS 绑定 AZ，跨 AZ 迁移要快照重建；S3 必须开版本控制 + MFA Delete，我曾遇到生产桶被脚本误删前缀，靠版本控制 5 分钟找回全部对象；备份别只留单 Region，重要数据用 S3 CRR 复制到第二个 Region。成本量化：用 Cost Explorer 的 Storage 维度看 Storage Class 占比，日志/备份这类低频数据占比通常超过 60%，分层流转后账单能降 40% 以上。

**延伸考点：** S3 生命周期策略的最小对象大小限制与零字节对象陷阱是什么？Athena 查日志时的分区策略与压缩格式（Parquet）如何选？

---

### Q6. 订单库要 99.99% 可用还要扛双十一，RDS、Aurora、DynamoDB、Redshift 怎么分？

**问题：** 你们有一个订单系统：写入峰值 2 万 TPS、需要强一致事务、按用户查最近订单（延迟 <50ms）、月度报表分析。现有工程师提议"全上 RDS MySQL 就行"，你怎么评估 RDS/Aurora/DynamoDB/Redshift 的分工与选型？

**期望加分项：**
- 明确四类服务定位：RDS 是托管关系库（MySQL/PostgreSQL）、Aurora 是云原生关系库（兼容 MySQL/PG、存储与计算分离）、DynamoDB 是 NoSQL 键值/文档库（毫秒级、自动扩缩）、Redshift 是列式分析数仓
- 场景映射：订单核心交易（强一致事务、SQL 生态）用 Aurora MySQL（写入吞吐比 RDS 高数倍、6 副本跨 3 AZ 存储）、用户最近订单查询用 DynamoDB（单表设计、主键 userId + 排序时间戳，getItem 稳定亚毫秒）、月度报表用 Redshift（从 Aurora 通过 DMS/CDC 同步或 Aurora Zero-ETL 集成）
- 讲清多 AZ 与高可用：RDS/Aurora 开 Multi-AZ（主备跨 AZ，故障自动切换 RTO 30-60s），Aurora 读副本最多 15 个可跨 Region、写走主节点；DynamoDB 三 AZ 自动复制，开 Global Tables 跨区域多活
- 只读副本与读写分离：Aurora 读副本做报表/搜索流量隔离，RDS Read Replica 最多 5 个、可跨 Region（用于容灾与就近读），但注意复制延迟（秒级）与主库压力
- 有量化容量规划：DynamoDB 用 On-Demand 模式免容量规划但贵，高峰期可切 Provisioned + Auto Scaling；Aurora 按 Aurora Capacity Units（ACU）算 Serverless v2 弹性，双十一用 Aurora Serverless v2 自动扩到峰值
- 提到选型取舍的代价：DynamoDB 不支持复杂 JOIN/事务范围有限（TransactWriteItems 最多 100 项），别硬塞报表需求；Redshift 不适合点查（列式、按列压缩），也别用 RDS 跑数 TB 级分析

**减分项：**
- 一句"MySQL 天下第一"，所有负载全塞 RDS，报表查询把主库拖垮
- 不知道 Multi-AZ 与只读副本的区别（一个管容灾、一个管读扩展）
- 把订单热数据与历史冷数据混在一个库，单表上亿行开始卡
- 用 DynamoDB 做复杂报表/联表查询，用 Redshift 做点查，选型错位
- 没有容量与成本量化：Aurora Serverless v2 与按需 RDS 的账算不清

**解答：**

正确的姿势是"分而治之"，单靠 RDS 撑不住 2 万 TPS 写入加报表双压。核心订单链路用 Aurora MySQL：相比 RDS，Aurora 存储与计算分离，写路径通过分布式存储层并行落盘，同等规格写吞吐明显更高；开 Multi-AZ 主备跨 AZ（默认同区域 3 AZ 6 副本），故障切换 RTO 约 30s，满足 99.99% 的 SLO。2 万 TPS 的写入不直接打主库：订单先落 SQS 异步消费 + 分库分表（按 userId hash 分 16 个库），或直接利用 Aurora 的写入能力加连接池（RDS Proxy 复用连接、消除冷连接风暴）。用户查"最近订单"这个高频只读点查切到 DynamoDB：单表设计（PK=userId、SK=orderTime），一份数据由应用双写或通过 CDC（DMS/Kinesis）同步过去，getItem 稳定在 1-3ms，扩容零运维——这才是 DynamoDB 的正确打开方式。月度报表走 Redshift：Aurora 到 Redshift 用 Aurora Zero-ETL（原生集成）或 DMS 持续同步，报表查询与交易库彻底隔离。高可用细节：Aurora 读副本跨 AZ 部署并接到报表/搜索只读流量，主库挂了读副本可提升为主（Promotion）；DynamoDB 本就三 AZ，无需操心。常见的坑有两个：一是把所有历史数据堆在主库不归档，我建议订单超 6 个月的数据定期归档到 S3 + Athena（或 Redshift），主库只保留热数据；二是报表 SQL 直接连 Aurora 主节点，双十一一跑分析查询把写路径拖垮——报表必须走只读副本或数仓。

**延伸考点：** Aurora Serverless v2 的扩缩容时机与冷启动对峰值写入的影响？DynamoDB 单表设计（PK/SK 模式）如何支撑"按用户查订单"与"按时间范围扫订单"两个查询模式？

---

### Q7. 双账号三环境，VPC 怎么划、安全组怎么管、怎么打通？

**问题：** 你们有 dev/staging/prod 三个环境，其中 prod 又要按业务拆多个账号做隔离，微服务之间要互通、还要访问共享的 Redis 与数据库。请设计一套 VPC/子网/安全组/IAM 的整体网络与权限方案，并说明 VPC Peering 与 Transit Gateway 怎么选？

**期望加分项：**
- 明确 VPC 规划：每个账号按环境一个 VPC（dev/staging 可共用账号但 VPC 隔离），VPC 内按 AZ 划分公有/私有子网（至少双 AZ），用独立 VPC 的 CIDR 大段规划避免重叠（如 dev 10.1.0.0/16、staging 10.2.0.0/16、prod 10.3.0.0/16）
- 讲清安全组设计：按服务角色建安全组（web-sg、app-sg、db-sg），跨安全组用 SG ID 引用（允许 web-sg 访问 app-sg 的 8080），比 IP 段更易维护；NACL 兜底做子网级粗粒度防线
- Transit Gateway 的选型理由：VPC 数 > 5 或跨账号互通时，VPC Peering 网状连接爆炸（N 个 VPC 要 N×(N-1)/2 个对等），TGW 是星型中心（attach 即连通），支持跨账号（Resource Access Manager 共享）、路由表隔离、集中 NAT/防火墙
- 讲清跨账号访问模式：共享资源（Redis/DB）放共享服务账号，用 PrivateLink（VPC Endpoint）暴露服务，消费方账号只建 Endpoint 不打通整个 VPC——最小暴露面，跨账号/跨 VPC 都可用
- 网络边界与出网：私有子网出公网走 NAT Gateway（每 AZ 一个高可用），生产禁止默认路由 0.0.0.0/0 直通公网出口；跨账号审计用 VPC Flow Logs 统一汇聚
- 有实际选型经验：Peering 无中转（不能跨 Peering 再 Peering）、无中心化管理；TGW 有每条 attach 的流量费（按 GB 计费）与路由表收敛复杂度，VPC 少且静态时 Peering 更简单省钱

**减分项：**
- VPC 网段规划无整体意识，三个环境 CIDR 全重叠，未来打通必冲突
- 3 个环境 20 个 VPC 全用 Peering 网状互连，运维爆炸
- 不知道 PrivateLink 能"只暴露服务端口"而不是打通整个 VPC，安全组开得又宽又大
- 单 NAT Gateway 挂所有私有子网，AZ 故障直接断网
- 说不清 TGW 路由表与 Peering 路由的传播机制，配置后流量不通

**解答：**

我的方案分三层：账号隔离、VPC 规划、互联选型。账号层：dev/staging 合用一个账号但 VPC 隔离，prod 按业务拆订单/支付/库存三个账号——账号是权限与成本的最小边界（IAM + 成本标签都按账号挂），生产事故不会波及测试。VPC 层：CIDR 统一登记，dev 10.1.0.0/16、staging 10.2.0.0/16、prod 10.3.0.0/16，每 VPC 内双 AZ 公有/私有子网各一套，数据库与 Redis 放私有子网且默认路由只指向 NAT，不出公网。互联层是重点：VPC 数在个位数时我可能用 Peering（配置简单、无流量附加费），但一旦跨账号 + 数量超过 5 个，直接上 Transit Gateway：prod 三个账号的 VPC 各自 attach 到 TGW，TGW 路由表按账号建"分段"——订单与支付 VPC 之间放行、对库存只开只读端口，用 TGW 的 route table 做收敛而不是逐条配 Peering 路由（Peering 网状要 3×(3-1)/2=3 对，10 个 VPC 就是 45 对，根本管不过来）。共享 Redis/DB 放共享服务账号，通过 PrivateLink 暴露：消费方在 VPC 里建 Interface Endpoint，安全组只放该 Endpoint 的 ENI 访问 6379/3306，两边 VPC 不需要任何对等关系，暴露面小到只有一个端口。出公网按 AZ 各建一个 NAT Gateway 并配置路由表 failover，避免单 NAT 成为可用性单点。最后所有账号开 VPC Flow Logs 汇聚到中心 S3 桶，做安全组与流量审计——我排过"某实例被挖矿"的案例，靠 Flow Logs 定位到被攻破实例只用了 10 分钟。安全组一律用 SG ID 引用（app-sg 允许来自 web-sg），后端扩缩容不用动任何规则。

**延伸考点：** Transit Gateway 的跨账号共享（RAM）与分段路由表（Route Table Association）如何实现"订单→支付可通、订单→库存只读"？私有子网内 EC2 如何安全地拉取软件源（VPC Endpoint for S3 vs NAT）？

---

### Q8. 网关、App、老系统三套流量，ALB、NLB、Route 53、Global Accelerator 怎么分工？

**问题：** 你们有一批微服务需要对外提供 HTTP API（WebSocket 长连接 + HTTPS REST），还要给数据中心老系统提供固定的 TCP 入口（回源 IP 白名单），同时国内用户访问延迟偏高。ALB/NLB/CLB 怎么选，Route 53 和 Global Accelerator 又各自解决什么问题？

**期望加分项：**
- 讲清三层负载的定位：ALB 是 L7 应用负载均衡（HTTP/HTTPS/WebSocket、基于路径/主机路由、配合 ECS/EKS Ingress）、NLB 是 L4 传输层（TCP/UDP、极低延迟、保留源 IP、适合海量连接）、CLB 是上一代传统 LB（无 L7 高级路由，新项目不建议）
- 场景映射：REST API 与 WebSocket 用 ALB（Listener 按路径分发给不同 Target Group，WebSocket 长连接靠 target group 的 connection draining 与 stickiness），老系统 TCP 固定入口用 NLB（静态 IP 可用 Elastic IP 绑定，配合 Global Accelerator 固定任播地址做白名单）
- 讲清跨可用区与故障转移：ALB/NLB 都是区域级服务自动跨 AZ，Target Group 健康检查（HTTP 200/自定义路径）决定摘流；NLB 不缓存连接、直通后端，特别适合对延迟敏感与长连接场景
- Route 53 的定位是 DNS 层：加权/延迟/故障转移/地理位置路由策略解决"请求先到哪"的问题；健康检查联动（DNS failover）在 LB 整区不可用时把流量切到备用区域
- Global Accelerator 是网络加速层：任播 IP（Anycast）就近接入边缘、内部走 AWS 全球骨干网回源，比公共互联网稳定，还能提供双静态 IP 供防火墙白名单；与 CloudFront 的区别是 GA 加速动态流量（TCP/UDP），CloudFront 主要加速 HTTP 静态/动态内容并做缓存
- 有选型边界：WebSocket 长连接量大时考虑 NLB + 后端直通（ALB 有 idle timeout 与连接数压力）；IP 白名单变更频繁选 GA（不用改白名单、IP 固定）而不是一直改安全组

**减分项：**
- 分不清 ALB/NLB/CLB，问"哪个更高级"而不是"哪个匹配协议与场景"
- 把 Route 53 当 LB 用，不知道 DNS 的 TTL 缓存会导致故障切换有分钟级延迟
- 不知道 NLB 保留源 IP、ALB 会改写 X-Forwarded-For，后端拿不到真实 IP 排查不了
- 选型只看功能不看协议：TCP 长连接硬塞 ALB，WebSocket 在 NLB 上裸奔
- 没有故障转移设计：单 LB 区域级故障没有 DNS/GA 层的兜底

**解答：**

我的分配原则是"协议决定 LB，地理决定 DNS/GA"。微服务的 HTTPS REST API 与 WebSocket 走 ALB：一个 ALB 上挂两个 Listener（443 做 REST、路径 /ws 走 WebSocket 转发的 Target Group），ALB 按路径/主机头把流量分给不同 ECS 服务，配好 target group 的 health check（/healthz 返回 200）与 connection draining（摘流等存量连接结束），发布时配合 CodeDeploy 蓝绿实现零中断。老系统 TCP 固定入口用 NLB：NLB 是 L4 直通、保留源 IP、延迟最低，且能给每个可用区绑定固定 EIP，数据中心侧防火墙直接白名单这些 EIP——注意这里我会叠加 Global Accelerator：GA 提供两个任播静态 IP，写入白名单后永远不用改，流量就近进入 AWS 骨干网再走内部网络回源到 NLB，跨洋/跨区访问延迟能降 30-50%，比直接走公网回源稳定。Route 53 在这套里的角色是"最上层的 DNS 门卫"：主区域用延迟路由策略把用户带到最近的 GA 入口或 ALB，健康检查联动做 DNS 故障转移——当 ap-southeast-1 整个不可用时，把流量切到备用区域（如 ap-northeast-1）的整套堆栈。必须讲的两个坑：一是 DNS TTL，Route 53 切换要等 TTL 过期（默认 300s），所以"秒级容灾"必须靠 GA（任播秒级收敛）而不是纯 DNS；二是 NLB 直通后没有 X-Forwarded-For 改写，后端要从源 IP 直接识别客户端，日志与限流逻辑要按 NLB 语义设计。CLB 我只在遗留迁移过渡期用，新架构一律 ALB/NLB。

**延伸考点：** ALB 的 Target Group 粘性会话（Sticky Session）与 WebSocket 长连接的兼容性怎么处理？Global Accelerator 与 CloudFront 双加速（动态+静态分离）如何组合？

---

### Q9. 一个开发要"只读访问所有 S3"，怎么把 IAM 权限设计到最小？

**问题：** 你们一个数据团队要"读全部 S3、能跑 Athena、不能删任何东西、不能动别的账号资源"，同时运维要能切换角色到生产账号做紧急修复。请设计一套 IAM 用户/组/角色/策略/权限边界/AssumeRole 的整体方案，并解释策略评估逻辑。

**期望加分项：**
- 讲清 IAM 实体模型：用户（长期凭证）、组（权限聚合）、角色（临时凭证、可跨账号/给服务用）、策略（托管/内联）、权限边界（Permission Boundary，角色最大权限天花板）
- 落地设计：数据团队建组 data-team，挂托管策略 AmazonS3ReadOnlyAccess + Athena 自定义策略（只允许 Query 相关操作），用户全部通过组拿权限、禁止给个人挂内联策略
- 角色与 AssumeRole：生产账号建 prod-admin-role（挂 AdministratorAccess 但加权限边界限制不能动 IAM），运维在开发账号用 STS AssumeRole 切过去（aws sts assume-role --role-arn ...），临时凭证默认 1 小时、可续期；跨账号先配信任策略（Trust Policy）
- 讲清策略评估逻辑：默认显式拒绝 → 权限边界（超过边界的权限直接无效）→ 组织 SCP（超过组织边界的直接无效）→ 身份策略/资源策略/会话策略显式允许 → 最后显式拒绝优先（Explicit Deny 永远赢）
- 权限边界与 SCP 的对比：权限边界是账号内"某角色最大能力"的天花板，SCP 是组织级"整个账号的最大能力"天花板，两者叠用做纵深——即使主账号管理员误授 AdministratorAccess，SCP/边界仍能兜住
- 有最小权限的实践手段：用 IAM Access Analyzer 检查策略是否过宽、用 `aws iam simulate-principal-policy` 模拟验证、给数据团队建专用 KMS 密钥且限制解密权限，防止"能读 S3 就能解密"

**减分项：**
- 权限设计混乱：直接给用户挂 AdministratorAccess，或者把 AK/SK 写在实例里/仓库里
- 不知道跨账号用 AssumeRole，而是给别的账号直接建 IAM 用户
- 说不清显式 Deny、权限边界、SCP 的优先级关系，被问就懵
- 没有最小权限意识：ReadOnlyAccess 一把梭，还能删、还能解密
- 不了解临时凭证与长凭证的区别，不知道 STS 过期与轮换

**解答：**

我先把"最小权限"拆成四个天花板：SCP（组织级）＞权限边界（账号内角色级）＞身份策略（用户/组/角色）＞资源策略（桶/队列），评估顺序是"任何一层显式拒绝即拒绝，允许需要所有相关层都放行"。数据团队的落地：建组 data-team，组上挂 AmazonS3ReadOnlyAccess 与一个自定义 Athena 策略（仅允许 StartQueryExecution/GetQueryResults，数据库只读），用户入组即获权限、出组即收回，禁止个人内联策略——我见过最乱的环境就是每个工程师自己贴一条 `s3:*` 内联策略，审计时根本查不清谁有权限。跨账号运维用 AssumeRole 而不是建用户：生产账号建 prod-admin-role，信任策略声明"仅允许开发账号的 ops-group 角色 AssumeRole"，运维先 `aws sts assume-role` 拿到 1 小时临时凭证（自动轮换、无 AK/SK 泄露风险），这比在多个账号复制用户要安全得多。关键坑是"ReadOnly 并不只读"：AmazonS3ReadOnlyAccess 只挡写/删，但如果桶里对象用 SSE-KMS 加密，读还需要 KMS 解密权限，数据团队要单独配 KMS 密钥的 Decrypt，否则要么读不了、要么误授了全部 KMS 权限；同时要防"读 S3 等于能拖走机密数据"，敏感桶用 S3 Object Ownership + 专门的受限角色访问。最后用 IAM Access Analyzer 定期审计"哪些角色权限过宽"，对 prod-admin-role 加权限边界（如禁止 iam:*、禁止删桶），这样即使策略写错，边界也能兜住。

**延伸考点：** 权限边界的典型误用（把边界设得比身份策略还宽导致形同虚设）？如何用 `aws iam simulate-principal-policy` 验证"某人到底能不能删某个桶"？

---

### Q10. 客户要求"数据必须加密、所有操作可审计、能防挖矿",安全基线怎么搭？

**问题：** 你们一个金融级客户要求：存储数据全部加密、管理员操作全程留痕可审计、能发现疑似入侵（挖矿/暴力破解）并在 5 分钟内告警。你会用 KMS/CloudTrail/GuardDuty/安全组搭一套怎样的安全基线？

**期望加分项：**
- 讲清加密分层：静态加密三件套——S3 SSE-KMS（对象）、EBS Encryption by default（块存储）、RDS 加密（库），全部用 KMS 客户托管密钥（CMK）而不是 AWS 托管密钥，密钥开启自动轮换（默认 1 年）与 key policy 限制使用者
- CloudTrail 审计链路：开多区域追踪（Multi-Region Trail）投递到中心 S3（启用 SSE-KMS 加密 + 对象锁定防篡改），管理事件默认记录，数据事件（S3 GetObject 等）按需开启但量大会烧钱（按百万事件计费）；配合 CloudWatch 告警监控"Root 登录、IAM 策略变更、安全组开放 0.0.0.0/0"
- GuardDuty 威胁检测：智能威胁检测服务（异常 API 调用、DNS 挖矿域名、暴力破解），喂入 CloudWatch 事件 → SNS 告警 → Lambda 自动隔离（如对实例贴 deny 安全组/停止实例），告警到处置全自动
- 安全组基线：生产安全组禁止 0.0.0.0/0 开放 22/3389（SSH 走 SSM Session Manager），只放业务端口到 LB/白名单网段；用 VPC Flow Logs + GuardDuty 的 Malware Protection 做可疑流量闭环
- 有量化与运维闭环：用 Security Hub 做统一安全态势（聚合 GuardDuty/Inspector 发现项，按 CIS 基线打分），定期（每周）跑一次 `aws config` 规则（如"所有 EBS 加密""无公网 SSH"）自动巡检不合规资源
- 提到事故处理实践：曾用 GuardDuty 的 CryptoCurrency 发现项定位挖矿实例，5 分钟内在 EC2 控制台对实例打隔离安全组 + 快照取证 + 终止实例，全程 CloudTrail 留痕

**减分项：**
- 只谈"用了 KMS"，说不清托管密钥 vs 客户托管密钥、轮换策略与 key policy 边界
- 不知道 CloudTrail 数据事件单独计费，开了之后账单翻几倍
- GuardDuty 告警没有处置动作，告警完事，入侵链路不闭环
- 22 端口对 0.0.0.0/0 开放还振振有词"有密钥就行"，不知道暴力破解与密钥爆破风险
- 审计日志不加密不锁，被入侵者直接删 CloudTrail 抹掉痕迹

**解答：**

安全基线我按"加密、审计、检测、隔离"四层搭。加密层：账号开启 EBS default encryption + 新建 S3 桶默认 SSE-KMS + RDS 加密存储，全部用客户托管 CMK，key policy 只授权指定角色与运维组，开启 1 年自动轮换；这里强调"默认加密"而不是"逐实例手工配"——我见过有人漏配一块 500GB 的 EBS 被合规打回。审计层：CloudTrail 开多区域追踪投递到中心审计桶，桶启用 SSE-KMS 加密 + S3 Object Lock（合规模式，防篡改防删），管理事件全量记录；数据事件（GetObject/PutObject）按关键桶选择性开启并接 S3 事件告警，避免全量开启导致日志费用失控；配套 CloudWatch 规则监控高危事件：root 登录、`iam:AttachUserPolicy`、安全组 `RevokeSecurityGroupEgress`/开 0.0.0.0/0。检测层：GuardDuty 常开，重点看 CryptoCurrency 与 BruteForce 发现项，事件通过 CloudWatch Events → SNS → Lambda 自动处置：对被挖矿实例先打"隔离安全组"（只出不进且出口 DNS 受限）保住现场，同时 `aws ec2 create-snapshot` 取证、再终止实例，全程 5 分钟内完成且每一步都留 CloudTrail 记录；Security Hub 聚合 GuardDuty/Inspector/Config 的发现项做月度安全评审。底线守则：生产环境 22/3389 不对 0.0.0.0/0 开放，运维走 SSM Session Manager（免暴露端口、全程可审计）；定期用 AWS Config 托管规则（ec2-volume-inuse-check、restricted-ssh）自动扫描不合规资源并触发整改工单。这套下来，安全评审的"加密覆盖 100%、审计可追溯、入侵 5 分钟告警"都能直接拿数据说话。

**延伸考点：** KMS 的 key policy 与 IAM 策略叠加后权限如何计算？S3 Object Lock 的合规模式与治理模式在审计留存上的差异？

---

### Q11. 线上接口突然 P99 延迟翻倍，怎么用 CloudWatch 和 X-Ray 定位？

**问题：** 你们一个支付回调接口 P99 延迟从 200ms 涨到 500ms，客户开始投诉。请说明你会怎么用 CloudWatch 指标/告警/日志和 X-Ray 做监控、定位与根因分析，并给出实际排查步骤。

**期望加分项：**
- 监控分层：基础设施层（EC2 CPU/内存/网络、EBS IOPS）、应用层（自定义指标：QPS、P99、错误率，用 CloudWatch Agent 或 StatsD）、依赖层（RDS 连接数/慢查询、Redis 延迟）、业务层（支付成功率、回调重试率）——四层都要有，缺一层定位就慢
- 告警设计：复合告警与阈值告警结合（如"P99>300ms 持续 5 分钟"触发，避免抖动误报）；用 CloudWatch Alarms 接 SNS；SLO 视角：用 `aws cloudwatch put-metric-alarm` 或开源 kube-prometheus 观测大盘
- 日志闭环：应用日志结构化输出到 CloudWatch Logs（JSON 格式、带 requestId），Logs Insights 用 PIPE 语法聚合（`fields @timestamp | stats pct(@duration, 99) by @message`），配合订阅过滤器（Subscription Filter）把慢日志转发到 Elasticsearch/Loki
- X-Ray 链路追踪：为 Lambda/ECS/API Gateway 开启 X-Ray tracing，Span 记录每个下游调用（DynamoDB/Redis/HTTP），用 Service Map 看延迟都花在哪；结合 CloudWatch 日志的 requestId 串起全链路（日志里打印 trace-id）
- 有实际排障套路：先看告警图缩小范围（是整体慢还是某实例慢、哪个 AZ），再看 X-Ray 服务图定位慢在下游哪个服务，最后用 Logs Insights 查该路径的慢日志找 SQL/GC/锁，形成"指标→链路→日志"的黄金链路
- 提到高频根因：慢 SQL（索引失效/大事务）、连接池打满（应用与 RDS 连接数）、GC 停顿（Java）、热点实例不均（ALB 会话粘连）、下游限流重试风暴

**减分项：**
- 只有 CPU/内存告警，没有应用层与业务层指标，P99 翻倍了都不知道
- 日志不结构化、不打印 requestId，X-Ray 与日志对不上
- 只会看 CloudWatch 控制台曲线，不会 Logs Insights 查询语法，也不开 X-Ray
- 告警阈值拍脑袋或阈值过宽，半夜被误报轰炸，最后没人看告警
- 没有根因闭环：定位到慢查询就完，不回溯为什么索引失效/为什么连接打满

**解答：**

我的方法论是"指标→链路→日志"三层夹击。第一层指标：确认四层监控都在（EC2 CPU 与网络、应用 P99/QPS 自定义指标、RDS 连接数与慢查询、业务支付成功率），先看告警图——如果 P99 涨但 QPS 没涨，排除流量型问题；如果只有某一个 AZ 的实例 P99 高，怀疑单 AZ 问题；如果所有实例都慢，问题在下游依赖。第二层链路：开 X-Ray tracing，看 Service Map 上延迟分布——我遇到过典型案例：P99 翻倍但 X-Ray 显示业务代码耗时没变，慢在调用 DynamoDB 的 Retry 上，原因是打满 RCU 后 SDK 自动重试放大延迟，这是"指标层完全看不出、X-Ray 一眼定位"的场景。第三层日志：用 CloudWatch Logs Insights 查慢日志（`fields @timestamp, @message | filter @duration > 300 | stats count() by @message`），结合 X-Ray 的 trace id 串起 requestId 找到具体请求路径。告警配置上我坚持"SLO 驱动"：`aws cloudwatch put-metric-alarm --alarm-name pay-api-p99 --metric-name p99 --threshold 300 --period 300`，P99>300ms 持续 5 分钟才告警，接 SNS 到值班群；同时配 RDS 慢查询告警和连接数告警（80% 阈值），往往比应用指标更早暴露问题。高发根因清单要熟：慢 SQL（数据量大后索引失效）、Java GC 停顿（堆不足触发 Full GC 风暴）、连接池打满（应用连接数×实例数超 RDS max_connections）、下游 5xx 引发的重试风暴。定位只是第一步，根因闭环更重要——我通常会在排查后给团队留下"为什么 P99 会涨"的复盘文档与告警优化清单，防止同类问题再发。

**延伸考点：** CloudWatch 的指标分辨率（标准 1 分钟 vs 高精度 1 秒）与保留期如何影响 SLO 计算？X-Ray 的采样策略（Sampling Rule）怎么配置才能在省钱与可观测间平衡？

---

### Q12. 全部资源要"代码化可审计"，CloudFormation 还是 Terraform？

**问题：** 你们要把现有 AWS 环境全面 IaC 化：几百个资源要能重复部署、变更可审计、与 CI/CD 打通，团队同时用 AWS 服务与第三方服务（Datadog、Sentry）。选 CloudFormation 还是 Terraform？两者的堆栈/状态管理/变更集机制有什么差异？

**期望加分项：**
- 讲清核心差异：CloudFormation 是 AWS 原生声明式 IaC（模板 YAML/JSON、状态由 AWS 管理、天然可审计），Terraform 是跨云声明式 IaC（HCL、本地状态文件或远端 Backend、插件生态覆盖非 AWS 服务）
- 选型逻辑：纯 AWS 环境且重视"变更集审计 + 回滚"用 CloudFormation（Change Set 预览、Rollback on failure、Drift Detection 检测漂移）；有第三方服务（Datadog/Sentry）或多云（阿里云/腾讯云）用 Terraform（provider 生态、HCL 可读性强）
- 讲清机制对比：CloudFormation 用 Stack/ChangeSet（先评估后执行、失败自动回滚、支持嵌套栈），Terraform 用 State（tfstate 锁 + 远端存储 + plan/apply，状态文件是单点，要放 S3 加 DynamoDB 锁）；CloudFormation 的 Drift Detection vs Terraform 的 plan 都是防漂移手段
- 有实践细节：CloudFormation 参数用 SSM Parameter Store 注入敏感值、模板用 `Fn::ImportValue` 做跨栈引用、宏（Macros）/自定义资源做扩展；Terraform 用 module 组织、`terraform plan` 输出直接进 MR 评审、`terraform apply -auto-approve` 走 CI 门禁
- 讲清 IaC 的坑：手动改资源后 Terraform 的 state 漂移（先 `terraform import` 再纳管）、CloudFormation 不支持的资源要用 Custom Resource 或改用 CDK（CDK 是 CloudFormation 之上用代码生成模板）；删除保护（TerminationProtection）与删除策略（DeletionPolicy）防止误删
- 有 CI/CD 集成经验：CodePipeline 里 CloudFormation 原生 action（CreateChangeSet/ExecuteChangeSet）或 Terraform 在 CodeBuild 里跑 plan/apply，变更都留审计

**减分项：**
- 只报工具名，说不清两者状态管理与回滚机制的本质差异
- 不知道 Terraform 的 state 文件是核心资产，放本地导致多人并发冲突
- CloudFormation 模板没有参数化与跨栈引用，几百个资源全写死
- 不理解 Change Set 与 plan 的区别（一个在 AWS 侧托管、一个本地计算），审计要求说不清
- 没有防漂移与删除保护意识，生产资源被误删后才发现

**解答：**

选型我按"环境纯 AWS 度 + 生态需求"决策。如果目标环境 100% 在 AWS 且要强审计，我倾向 CloudFormation：模板即代码、状态由 AWS 管理，天然支持 Change Set 预览（`aws cloudformation create-change-set` 先看将要新增/修改/删除的资源再执行）、失败自动回滚（Rollback on failure 默认开）、Drift Detection 定期检测"控制台手改"造成的漂移——这对金融合规的"变更可审计"是刚需，每次部署的变更集都能留档。模板里用 `AWS::SSM::Parameter::Value` 注入密钥、用 `Fn::ImportValue` 跨栈引用 VPC/安全组输出、复杂逻辑用嵌套栈与宏。但团队还有 Datadog/Sentry 这类第三方资源，Terraform 一个 provider 全搞定（cloudformation 只能 Custom Resource 硬写），且 HCL 的 module 复用比 CloudFormation 的嵌套栈更好维护，所以我现在的标准做法是"基础设施用 CloudFormation（或 CDK），跨云与第三方集成用 Terraform"，尽量不让两套工具管同一批资源（避免互相漂移打架）。Terraform 的关键纪律：state 必须放远端（S3 Backend + DynamoDB 锁防并发），`terraform plan` 输出进 MR 评审，apply 走 CI 门禁；接手旧环境先 `terraform import` 纳管已有资源，千万别"删了重建"。必须强调的坑：CloudFormation 有些资源属性变更会触发"替换"（Replacement）——比如修改 EBS 卷大小属性导致实例重建，Change Set 里会标红，评审时务必看清；另外给关键栈开 TerminationProtection，我曾见过误删生产 VPC 栈导致全环境瘫痪的案例。最后无论选哪个，部署都挂到 CodePipeline/CodeBuild 里，保证"一切变更走代码、留审计"。

**延伸考点：** CloudFormation 的 Change Set 与 Terraform plan 在"删除检测"上的差异（资源被手动删后：Drift 报告 vs state 漂移）？CDK 相比纯模板的优势与适用边界？

---

### Q13. 代码推到 GitHub 就要自动测试、构建、上 staging，发布还要能回滚，怎么做？

**问题：** 你们团队代码托管在 GitHub，希望"推 main 分支 → 自动跑测试 → 构建 Docker 镜像 → 部署到 ECS staging → 人工确认后发布生产，且发布能一键回滚"。请设计一套基于 CodePipeline/CodeBuild/CodeDeploy 的方案，并说明与 GitHub Actions 的集成方式。

**期望加分项：**
- 讲清三件套分工：CodePipeline 是编排（Source→Build→Deploy 阶段、支持手动审批门禁）、CodeBuild 是构建（Docker 镜像、单元测试、SonarQube 扫描）、CodeDeploy 是部署（ECS/EC2/Lambda 的滚动/蓝绿策略、自动回滚）
- 完整流水线设计：Source（GitHub 通过 CodeStar Connections 接入 webhook 触发）→ Build（CodeBuild 跑 test + `docker build` 推 ECR）→ Deploy-staging（CodeDeploy 到 ECS Fargate，Auto Scaling 蓝绿或 Rolling）→ Approval（手动审批门禁）→ Deploy-prod（蓝绿发布 + 自动回滚）
- 蓝绿与回滚细节：ECS 蓝绿发布用 CodeDeploy + 目标组切换（先起新版本验证、健康检查通过再切流量），失败自动回滚到旧目标组；EC2 用 CodeDeploy 的 AppSpec（hooks: BeforeInstall/AfterInstall/ValidateService）做优雅部署与健康验证；Lambda 用 CodeDeploy 金丝雀（Canary10% 5 分钟）
- 与 GitHub Actions 集成：两种路径——用 CodePipeline 原生 GitHub 源（简单、AWS 侧统一审计），或 GitHub Actions 负责构建推 ECR、再用 AWS 的 OIDC（Identity Provider）直接调 CodeDeploy/Lambda 部署（免密钥、免 AccessKey）；重点讲 OIDC 而不是把 AK/SK 塞进 Secrets
- 有质量门禁意识：Build 阶段跑单元测试 + 覆盖率门槛（<80% 失败）、静态扫描（Trivy/Snyk 扫镜像漏洞）、Schema 变更用迁移任务而非手改；staging 验证加冒烟测试步骤
- 提到回滚的量化指标：回滚时间目标（如 10 分钟内）、部署失败自动回滚（CodeDeploy 默认）、数据兼容性（发布前先跑向后兼容的 DB migration，回滚不回退 schema）

**减分项：**
- 只会说"用 GitHub Actions"，不知道 CodePipeline/CodeDeploy 的部署策略与回滚机制
- 把 AK/SK 直接放 GitHub Secrets，不使用 OIDC 或 CodeStar Connections
- 没有审批门禁，main 一推直接上生产
- 回滚只回滚代码不回滚数据，schema 变更后旧版本代码直接报错
- 部署不验证：没有健康检查/冒烟测试，流量切完才发现挂

**解答：**

我设计为五阶段流水线。Source：GitHub 仓库通过 CodeStar Connections 建立连接（替代旧的 GitHub OAuth App，权限更细），main 分支 push 触发 CodePipeline。Build：CodeBuild 跑单元测试与覆盖率门槛（如 JaCoCo/coverage 不足 80% 直接失败），通过后 `docker build -t $REPO:$COMMIT_SHA` 推 ECR，镜像标签用 commit SHA 保证可追溯；再跑 Trivy 镜像漏洞扫描，高危漏洞阻断。Deploy-staging：CodeDeploy 用蓝绿策略部署到 ECS Fargate——CodeDeploy 先创建新目标组、起新任务集、等健康检查（ALB /healthz 连续 200）通过才切 100% 流量，失败自动回滚到旧目标组，全程自动。Approval：staging 验证完成后进入人工审批门禁（手动批准才继续，回滚按钮就在旁边）。Deploy-prod：同样蓝绿 + 自动回滚，加上 CloudWatch 告警联动——新版本 P99 恶化或错误率飙升时自动触发回滚，回滚目标 10 分钟内完成。与 GitHub Actions 的集成我会给出双路径：简单场景直接用 CodePipeline 的 GitHub 源；团队已经重度用 Actions 时，用 GitHub OIDC 集成——在 GitHub 配置 AWS IAM Identity Provider（`actions/configure-aws-credentials` with role-to-assume），Actions 里构建推 ECR 后直接 `aws deploy` 触发 CodeDeploy，全程无任何 AccessKey 落盘。两个必讲的坑：一是回滚的"数据兼容性"——DB schema 变更必须向前兼容（加列不加删），旧版本代码才能安全接管流量，schema 回滚基本不做，靠兼容性设计；二是部署验证不能只看"任务起来了"，要在 AfterInstall/ValidateService hook 里跑冒烟用例（如核心 API 健康断言），否则流量切过去才发现配置缺失。

**延伸考点：** ECS 蓝绿部署的 target group 权重切换（Weighted）与 CodeDeploy 的自动回滚触发条件如何配置？GitHub OIDC 的 subject claim 如何按 repo/branch 收敛权限（避免任何仓库都能部署）？

---

### Q14. 对外 API 要限流、鉴权、按版本灰度，API Gateway 怎么搭？

**问题：** 你们要发布一个对外公共 API：第三方调用要 API Key 鉴权、每个密钥限流 1000 RPM、同时支持 JWT 登录态、还要能按版本（v1/v2）灰度切流量。用 API Gateway 的 REST API 还是 HTTP API？限流、鉴权、与 Lambda/后端集成怎么做？

**期望加分项：**
- 讲清 REST API 与 HTTP API 的差异：HTTP API 是新一代轻量网关（成本约为 REST API 的 70%、延迟更低、支持 JWT 原生鉴权与私有集成），REST API 功能全（API Key、Usage Plan、请求/响应转换映射、WAF 集成、自定义域名）——功能要求高选 REST，只要路由+JWT+限流选 HTTP
- 鉴权三层：API Key（第三方应用级，配 Usage Plan 限流）+ IAM 鉴权（内部服务用 SigV4）+ Cognito User Pools / JWT Authorizer（终端用户登录态，验证 token 的 issuer 与 audience）
- 限流设计：Usage Plan 按 Key 限流（1000 RPM/Key）+ 按阶段（Stage）限流（如 10000 RPM）+ 后端 Lambda 预留并发兜底；限流返回 429 与 `x-ratelimit-*` 响应头，客户端要按 Retry-After 退避
- 灰度与版本：Stage 做 v1/v2 环境（或自定义域名映射 `api.example.com/v1`、`/v2`），用 Stage Variable + Lambda 别名切换后端；流量按权重灰度（如 v1 80% / v2 20%）用 Route 53 加权或 ALB 加权目标组在网关后面做
- 与 Lambda 集成：代理集成（Proxy Integration）直接把 event 传给 Lambda（映射简单、少一层转换），需要改造响应格式或做协议转换（如 XML）时用映射模板（Mapping Template，REST API 才有）
- 有落地细节：CloudWatch 指标（4XX/5XX、Latency、ThrottleCount）与告警、请求日志（Execution Logs）脱敏、自定义域名走 ACM 证书 + Route 53 别名、WebSocket 场景用 WebSocket API 类型

**减分项：**
- 分不清 REST/HTTP/WebSocket 三类 API 的适用场景与能力差异
- 只做"网关转发"，鉴权/限流/日志全没有，裸奔暴露 Lambda
- 限流只配网关层，不知道后端与数据库也要兜底（网关穿透后被打挂）
- 灰度方案没说清：直接改域名/改路径给所有用户，没法小流量验证
- 不知道 429 语义与客户端退避，也不监控 ThrottleCount

**解答：**

选型：如果只需要"路由 + JWT + 基础限流"，我用 HTTP API（便宜、低延迟、原生 JWT Authorizer）；但需求里有 API Key 配 Usage Plan、按 Key 限流、WAF 与自定义域名这些，我会用 REST API——它具备 Usage Plan（按 API Key 分配配额与限流）、Mapping Template、更全的 CloudWatch 集成。整体方案：第三方走 API Key 鉴权，在 Usage Plan 里给每个客户 Key 配 1000 RPM 限流并关联到 v1 与 v2 两个 Stage；终端用户走 JWT：API Gateway 的 Authorizer 配置为 Cognito User Pool（或自定义 JWT 验证，校验 iss/aud/exp，返回 principalId 供后端做用户维度限流）。限流是三层：Usage Plan 按 Key 限 1000 RPM、Stage 级限流 10000 RPM 防单 Stage 被打爆、后端 Lambda 设 Reserved Concurrency（如 500）兜底——网关限流只是第一道闸，穿透流量后端必须能扛；超限返回 429 + `x-ratelimit-limit/remaining/reset` 响应头，我还会在 SDK 侧让客户端按 Retry-After 指数退避。灰度：v1/v2 用两个 Stage + 自定义域名路径映射（`/v1`、`/v2` 各指一个 Stage），后端切换用 Stage Variable 指向 Lambda 别名（v2 阶段先在别名上跑金丝雀），流量按权重灰度用网关后面 ALB 的加权目标组（v1 80%、v2 20%）或 Route 53 加权记录——注意网关本身无原生权重分流，要靠后端或 DNS 层做。与 Lambda 用代理集成：事件直接透传、响应原样返回，省掉映射模板这层维护；只有对接 XML/旧协议才用 Mapping Template。最后监控 4XX/5XX 与 ThrottleCount 告警，打开 Execution Logs 但脱敏 Authorization 头（防止日志里泄露密钥）。

**延伸考点：** API Gateway 的缓存（Cache）与后端缓存如何配置避免热点 Key 打穿？WebSocket API 的 $connect/$disconnect 路由与连接管理（ConnectionId 存 DynamoDB）如何做？

---

### Q15. 订单创建要发通知、触发积分、同步到数仓，消息怎么编排？

**问题：** 你们有个订单创建流程：下单后要发短信/邮件通知、触发积分计算、把订单同步到数仓、还要对失败做补偿。有人建议"全用 SQS 队列 + Worker 轮询"，有人建议"上 Kafka"。请给出 SQS/SNS/EventBridge 的编排方案，并说明何时才值得用 Kafka？

**期望加分项：**
- 讲清三个服务定位：SQS 是点对点消息队列（生产者→消费者、至少一次/精确一次投递、死信队列）、SNS 是发布订阅（一条消息广播给多个订阅者：SQS/Lambda/邮件/HTTP）、EventBridge 是事件总线（基于事件模式的过滤路由、支持 AWS 服务事件/自定义事件/第三方 SaaS 事件、可做计划任务）
- 场景编排：订单创建事件发布到 EventBridge（自定义事件 bus），规则按事件类型路由——"订单创建"事件广播给通知 SQS（短信）、积分 SQS（Lambda 计算）、数仓 SQS（DMS/同步任务）；EventBridge 的 Input Transformer 做字段裁剪，避免各消费者都收到全量订单字段
- SNS-SQS 直连或 EventBridge 的选择：需求简单、只要广播用 SNS→SQS 订阅；需要事件过滤、重放（Replay）、跨账号事件路由、与 AWS 服务事件（如 CodeBuild 状态）联动时用 EventBridge
- 可靠性与幂等：消费者必须幂等（同一订单重复消费不重复加积分），SQS 可见性超时（Visibility Timeout）与死信队列（DLQ 接告警）保证不丢不重；消息体设计带 version、idempotency key
- 与 Kafka 的对比与边界：SQS 是"消费即删除"、不支持回溯与多消费者组并发消费同一条消息（FIFO 队列 300 TPS 限制）；Kafka 保留日志可重放、分区序、多消费组、高吞吐（单分区数万 TPS）——需要事件溯源/流处理（Flink/KSQL）、海量吞吐、多消费者组各取所需时才值得自管 Kafka（MSK）；简单消息编排用 SQS/SNS/EventBridge 成本与运维都低得多
- 有补偿设计：失败进 DLQ 后接 Lambda 重试 + 人工补偿面板；跨服务一致性用 Saga 模式（订单状态机 + 各步骤补偿动作）而不是分布式事务

**减分项：**
- SQS/SNS/EventBridge 三个服务混为一谈，不知道点对点、发布订阅、事件总线的区别
- 无脑上 Kafka，理由是"别人都在用"，却说不清 Kafka 的高运维成本与场景匹配
- 不做幂等设计，SQS 至少一次投递导致积分重复计算、通知重复发送
- 没有 DLQ 与补偿，消息丢了就丢了，业务对不上账
- 用 SQS 做广播、用 SNS 做点对点，服务模型用反

**解答：**

编排思路：事件总线（EventBridge）做中枢 + 队列（SQS）做缓冲 + 消费者幂等。订单创建时业务代码 `PutEvents` 到自定义事件总线，事件体带 `orderId`、`type`、`timestamp`；EventBridge 规则按 event type 路由：规则 A 把事件投递到通知 SQS（短信服务消费）、规则 B 投递到积分 SQS（Lambda 计算积分）、规则 C 投递到数仓 SQS（DMS 或数据同步任务消费）。为什么用 EventBridge 而不是 SNS：EventBridge 支持按事件模式过滤（只有"订单已支付"才触发积分，而"订单已创建"不触发）、Input Transformer 裁剪字段（消费者只拿自己需要的字段，减少敏感字段暴露）、还能统一接入 AWS 服务事件（如 CodeBuild 失败自动告警）与定时任务（cron 触发日报），未来加新消费者只需加规则不用改生产者。可靠性三件套：SQS 队列开 DLQ（红/黄两条，重试 3 次进 DLQ），DLQ 深度监控告警 + 接 Lambda 自动重试、人工补偿面板兜底；消费者处理用幂等键（orderId + 事件类型做去重，写 DynamoDB 或 Redis SETNX），因为 SQS 是至少一次投递，重试时不能重复加积分；可见性超时设成"处理耗时的 2-3 倍"，防止处理中超时被并发消费。什么时候值得上 Kafka：需要事件日志保留与回溯（审计/事件溯源）、多消费组各自维护 offset、流处理（Flink/KSQL 做实时聚合）、单主题吞吐超 SQS 上限时，用 MSK（托管 Kafka）；但对订单通知这种"广播 + 消费即删"的业务，Kafka 完全是过度设计——多 3 个组件的运维面、topic/consumer group 管理、分区与重平衡的心智负担，换取的是用不到的容量。我会先问"要保留历史可重放吗？要流处理吗？"，两个都否就用 SQS/SNS/EventBridge 组合，成本不到 Kafka 的零头。

**延伸考点：** SQS FIFO 队列与标准队列的选择（顺序性、去重、300 TPS 限制）？EventBridge 的 Pipes（源→过滤→目标直连）与规则 + Lambda 中转的性能与成本差异？

---

### Q16. 全球用户访问官网图片与 API，CloudFront 怎么配才又快又省？

**问题：** 你们官网面向全球用户：静态资源（图片/JS/CSS）在 S3，动态 API 在 ap-southeast-1，希望全球访问延迟低、回源带宽成本可控、还能在边缘做个性化响应。请设计 CloudFront 的缓存策略、与 S3 的集成方式，并说明何时需要 Lambda@Edge/CloudFront Functions。

**期望加分项：**
- 讲清 CloudFront 缓存机制：缓存键（Cache Key）默认含 Host/URL/Query String，按需配置"忽略 Query String 或只缓存白名单参数"，TTL 由 Cache-Control 响应头控制（max-age/s-maxage），命中率提升的关键是"缓存键尽量少"与"TTL 尽量长"
- 静态资源策略：S3 作源 + 版本号命名（`style.v2.js`）实现"永久缓存 + 内容可更新"（文件名变 → URL 变 → 重新回源），Cache-Control: public, max-age=31536000, immutable；图片走 WebP/AVIF 自动转码（S3 预转或 Lambda@Edge 按 User-Agent/Accept 头响应不同格式）
- 动态 API 策略：默认不缓存或按身份缓存（Cache Key 加 Authorization/Cookie），可用"分区域 + 就近源"（Origin Shield 合并回源请求减少回源次数、多个源站做 failover）；动态内容不强制缓存，靠 TCP 优化（CloudFront 边缘加速 TLS 握手）也能降延迟
- 成本量化：回源成本 vs 边缘流量——命中率 90% 以上时回源流量只占 10%，S3 出流量费与 CloudFront 传输费对比要算清；Origin Shield 在中心区域再聚合一层缓存，多边缘回源变单点回源，回源量再降约 20-30%
- Lambda@Edge vs CloudFront Functions：CloudFront Functions 是轻量边缘函数（仅 Viewer Request/Response、JS、亚毫秒级、$0.10/百万次，适合改 header/URL 重写/简单鉴权）；Lambda@Edge 功能全（可读 S3/DynamoDB、可改请求与响应、四个触发点，适合 A/B 测试、个性化、响应动态拼接）——能在 Functions 做绝不上 Lambda@Edge（冷启动与成本差异巨大）
- 安全与集成：S3 桶私有 + OAC（Origin Access Control）授权 CloudFront 回源（替代旧的 OAI）、WAF 挂 CloudFront 做边缘 WAF 防护、Signed URL/Cookie 做付费内容鉴权、配合 Global Accelerator 的边界（静态用 CF、动态 TCP 用 GA）

**减分项：**
- 不知道缓存键概念，Query String 全缓存导致同一 URL 缓存成百上千份、命中率暴跌
- 静态资源不按版本号命名，改版后 CDN 缓存不失效，用户看到旧资源
- 用 Lambda@Edge 做改 header 这种 Functions 就能干的活，成本与冷启动翻几倍
- 回源链路没配 Origin Shield/缓存，全球流量全打回 S3，回源带宽费用爆炸
- S3 桶公开或只用旧 OAI，回源鉴权与安全基线说不清

**解答：**

静态资源这条链路是"版本号 + 永久缓存 + OAC 私有源"。S3 桶保持私有，配置 CloudFront 的 OAC（Origin Access Control）让 CDN 用 IAM 凭证回源；资源按版本号命名（`/static/js/app.8f3c2a.js`），响应头 `Cache-Control: public, max-age=31536000, immutable`，浏览器与边缘都永久缓存，更新 = 改文件名，命中率能做到 95%+；CSS/JS 在 Build 阶段由 Webpack/Vite 生成带 hash 文件名，这层做好了，改版不发版、不发版本号就是死局。图片场景：大图在 S3 预生成多尺寸，CloudFront 用 Lambda@Edge 的 Origin Request 钩子按 User-Agent 选设备尺寸、按 Accept 头选 WebP/AVIF，回源一次缓存多份变体。动态 API 这条链路别硬缓存：按请求路径 + 必要参数配缓存键，需要登录态的响应加 `Cache-Control: private` 或按 Cookie 维度缓存，同时给源站配 Origin Shield（新加坡区域做聚合缓存层）——全球 20 个边缘的回源请求先在中心区域合并，回源量可降 20-30%，S3 出流量和源站压力都明显下降。边缘函数选型我坚持"能 Functions 不用 Lambda@Edge"：CloudFront Functions 在 Viewer Request 改 header、做 URL 重写、简单 IP 白名单，亚毫秒、几乎零成本；Lambda@Edge 只在需要读写外部数据（A/B 测试查 DynamoDB、个性化推荐、动态拼接响应）时才用，且注意四个触发点（Viewer/Origin × Request/Response）的语义差异与超时限制（Viewer 侧 5s、Origin 侧 30s）。最后安全收口：WAF 挂 CloudFront（边缘拦截 SQLi/XSS/CC）、付费内容用 Signed URL（`aws cloudfront sign` 生成带过期时间的签名 URL），整个方案的回源鉴权、缓存命中率（监控 CacheHitRate）、回源流量成本（Cost Explorer 看 DataTransfer 维度）都有数据可查。

**延伸考点：** 缓存失效的两难——版本号命名 vs 动态 URL 内容更新的取舍；Origin Shield 与多源站 failover 的配置细节？CloudFront 的 TTL 与浏览器 Cache-Control 的配合（s-maxage vs max-age）如何避免"边缘缓存了、浏览器没缓存"？

---

### Q17. 月账单 8 万美元，CFO 要砍 30%，从哪下手？

**问题：** 你们 AWS 月账单 8 万美元，CFO 要求三个月内降 30% 且不能牺牲可用性。你会怎么用 Cost Explorer、Savings Plan、Spot、预算告警和成本分摊定位浪费、制定优化路径？

**期望加分项：**
- 先归因再优化：给所有资源打成本标签（环境/部门/应用），用 Cost Explorer 按标签分组看 TOP 成本项，通常 80% 账单集中在 20% 资源（EC2 与 RDS 常占 50-60%）；用 Compute Optimizer 找低利用率实例（CPU 常年 <10% 直接降配/关闭）
- 购买模式优化是最大杠杆：基线常驻资源（EC2/RDS/Fargate）买 Savings Plan（1 年 30-40%、3 年最高 60% 折扣），Spot 填充弹性负载；先分析"利用率是否稳定"再决定买 1 年还是 3 年，避免过度承诺
- 资源生命周期治理：非生产环境定时关机（Instance Scheduler 夜间/周末停）、开发/测试实例用 `aws ec2 stop-instances` + 标签识别、未挂载 EBS 卷与未使用 EIP 清理（EIP 空闲要收费）、RDS 按需实例评估是否转 Savings Plan
- 存储与网络优化：S3 生命周期分层（日志/备份转 IA/Glacier 通常能省 40-60%）、数据传出流量（Data Transfer）大时评估 CloudFront/直连、EBS gp3 替代 io1（gp3 默认 3000 IOPS 含在基价里）、快照只保留必要份数
- 预算与告警闭环：Budgets 按月/按服务设预算（超 80% 告警、100% 阻断通知）、Cost Anomaly Detection 检测异常突增（如某夜 EC2 被误扩到 500 台）、账单按标签分摊到部门（Chargeback），形成"预算—告警—问责"循环
- 有落地顺序与量化：先做购买模式（省 25%+），再清闲置（省 5-10%），最后优化存储与流量（省 5%），用 Cost Explorer 月环比验证效果，90 天目标拆成"第 30 天 10%、第 60 天 20%、第 90 天 30%"的里程碑

**减分项：**
- 没有成本归因就乱优化，不知道钱花在哪就买 Savings Plan
- 只盯着实例单价砍，忽略闲置资源、存储分层、数据流量这些"看不见的浪费"
- 不分析利用率就买 3 年 Savings Plan，资源下线后承诺费白交
- 没有预算与异常检测，某次误操作产生的 2 万美元费用月底才发现
- 优化牺牲可用性：把生产实例全关或全换 Spot，大促时没容量

**解答：**

先做归因，再动刀，顺序是"购买模式 → 闲置清理 → 存储与流量"。第一步归因：全账号资源打标签（env=prod/staging/dev、team、app），Cost Explorer 按标签与服务分组——我遇到的大部分账单里 EC2 + RDS 占 50% 以上、S3 存储次之、Data Transfer 是隐藏大头。第二步最大杠杆是购买模式：用 Compute Optimizer 统计所有实例与 RDS 的 CPU 利用率曲线，常驻基线资源（7×24 稳定运行的 40 台实例 + RDS）买 1 年期 Compute Savings Plan（约 30-40% 折扣），弹性部分继续按需 + Spot 混合；这里的关键是不拍脑袋买 3 年——利用率低于 50% 的资源说明会下线，承诺费就白交了。第三步闲置清理：Instance Scheduler 让 dev/staging 夜间与周末自动关机（光这一项测试环境通常省 20-30%）、清理未挂载 EBS 卷与空闲 EIP（每个 EIP 每月约 3.6 美元）、RDS 快照只留最近 7 份、io1 卷转 gp3（gp3 基础 3000 IOPS 免费，同性能便宜 20%）。第四步存储与流量：S3 生命周期把日志 30 天转 IA、90 天转 Glacier，备份转 Deep Archive；Data Transfer 大的服务评估 CloudFront 或检查是否走了跨 Region 同步（跨区传输费按 GB 计）。第五步是防回潮：Budgets 设三档（月预算 80% 告警、100% 通知）、Cost Anomaly Detection 盯异常突增——我经历过一次事故：测试账号误开 500 台 GPU 实例，当天 Cost Anomaly 就报了 4 万美元异常，靠这个在 24 小时内止损。落地节奏按里程碑：第 30 天购买模式落地（预计省 20-25%）、第 60 天闲置清理 + 非生产关机（累计 25-30%）、第 90 天存储流量优化（30%+），每次月报用 Cost Explorer 环比验证，同时把账单按标签摊回各部门做 Chargeback，让"成本是别人的事"变成"成本是各团队的事"。

**延伸考点：** Savings Plan 的 Compute 与 EC2 Instance 类型差异（后者绑定实例族，资源变更会失效）？Cost Anomaly Detection 的监视器（Monitor）如何按服务/账号收敛异常范围？

---

### Q18. 安全团队要你们过 Well-Architected 评审，六个支柱怎么逐项自证？

**问题：** 客户审计要求你们的生产系统通过 AWS Well-Architected 框架评审（六大支柱），安全团队列了几十条检查项。你会如何按六支柱逐项自检，哪些是高频翻车点，怎么用 AWS 官方工具（Well-Architected Tool / Trusted Advisor / Config）落地？

**期望加分项：**
- 讲清六支柱：卓越运营（Operations）、安全（Security）、可靠性（Reliability）、性能效率（Performance Efficiency）、成本优化（Cost）、可持续性（Sustainability，第七支柱）；给出每支柱的核心问题集
- 高频翻车点清单：可靠性——单 AZ 部署、无备份/恢复演练、无 ASG；安全——IAM 权限过宽、22 端口公网开放、未加密、CloudTrail 未开；成本——无预算、无标签；卓越运营——无告警、无变更流程；性能——无 Auto Scaling、无缓存
- 用官方工具落地：Well-Architected Tool（建 workload、跑 Review 问卷、生成改进项与优先级）、Trusted Advisor（账户级检查：安全组开放端口、EBS 快照、配额、成本建议）、AWS Config（托管规则持续合规：encrypted-volumes、restricted-ssh 等 200+ 条规则）三件套组合使用
- 有量化改进闭环：Review 打分 → 按"高影响低投入"排序（如先修 restricted-ssh、再开 CloudTrail、再补备份）→ 每季度复评，用 WAFR 的里程碑追踪；给出自证材料：备份策略文档、恢复演练报告（每半年一次 `aws rds restore-db-instance` 演练）、容量测试报告
- 提到组织视角：Reliability Pillar 的 RPO/RTO 目标先定（如 RPO 15min / RTO 1h），再反推备份频率、多 AZ、跨 Region 复制设计；Safety of changes 用 IaC + 蓝绿发布佐证
- 实践案例：曾把"22 端口开放 0.0.0.0/0"这类高危项用 Config rule + 自动修复（remediation）闭环，评审翻车项从 12 项降到 2 项

**减分项：**
- 背不出六支柱名称，或把"评审"理解为"考试背题"，没有落地的检查项
- 只说"我们很稳"，拿不出 RPO/RTO、备份演练、容量测试任何一项量化证据
- 不知道 Trusted Advisor / Config / WAFR 这些官方工具，全凭人工自查
- 修复不分优先级，一次性改几十项导致生产变更失控
- 评审完就完事，没有持续合规机制，三个月后又打回原形

**解答：**

先澄清：Well-Architected 不是考试，是"用六支柱问自己系统是否站得住"的自省框架。我的做法是四步：定基线、跑工具、按优先级修、持续复评。第一步定基线：可靠性支柱先定 RPO/RTO（业务目标：RPO 15 分钟、RTO 1 小时），所有设计反推——数据库 RDS 自动备份 15 分钟间隔 + Multi-AZ、应用多 AZ 部署 + ASG、关键状态落 S3 跨区域复制，这样评审时"备份和容灾"是拿数据说话而不是口号。第二步跑工具三件套：Well-Architected Tool 建 workload 跑 Review，Trusted Advisor 查账户级问题（安全组开放端口、闲置资源、配额），AWS Config 开 200+ 条托管规则做持续合规（重点：`ec2-volume-inuse-check`、`restricted-ssh`、`encrypted-volumes`、`cloudtrail-enabled`）。第三步按"影响 × 投入"排优先级，我通常的修法：先收安全口子（22 端口改 SSM、IAM 权限收敛、开 CloudTrail + 加密），再补可靠性（多 AZ、备份、恢复演练——每半年真跑一次 `aws rds restore-db-instance-from-db-snapshot` 恢复演练并留报告），再上成本与性能（标签、预算、ASG、缓存），一次 Review 会议不要超过 5 项变更，避免生产变更风暴。第四步持续化：Config 规则接自动修复（如检测到 EBS 未加密自动 remediate）、每月看 Security Hub 分数、每季度复评 WAFR。常见翻车点要主动提：单 AZ 部署、无恢复演练、IAM 全管理员、日志无保留策略、无成本预算——这些往往是评审被卡的重灾区。最后给审计的自证材料包：备份策略与演练报告、容量压测报告、变更审批流程（IaC + 蓝绿）、安全基线（Config 合规报表），这套下来评审通过率会高很多。

**延伸考点：** 可靠性支柱的"Backup and restore"与"Multi-AZ"两种恢复策略的 RTO/RPO 差异如何量化？Config 规则的自动修复（Remediation）用 SSM Automation 文档如何实现"发现未加密 EBS 自动加密"？

---

### Q19. 自建机房 200 台服务器迁 AWS，怎么迁才不踩雷？

**问题：** 你们 IDC 有 200 台物理机（Web、MySQL、Oracle、文件服务器），要迁到 AWS，停机窗口只有周末 24 小时，数据量约 10TB，Oracle 库是迁移难点。请给出迁移工具选型（SMS/MGN、DMS、S3 传输）与分波次迁移方案，如何保证数据一致性？

**期望加分项：**
- 讲清工具分工：Application Migration Service（MGN，原 CloudEndure Migration）做整机热迁移（持续复制 + 切换，支持 Windows/Linux、RTO 分钟级）；DMS（Database Migration Service）做数据库迁移（同构/异构、CDC 增量同步、零停机）；S3 Transfer Acceleration 或 AWS DataSync 做海量文件传输；离线大批量可用 Snowball/Storage Gateway
- 分波次迁移：第一批迁无状态 Web/应用（先验证网络、IAM、监控链路），第二批迁文件存储（S3/EFS + DataSync 增量同步），第三批迁数据库（DMS 全量 + CDC），第四批切 Oracle（重难点单独排期）——每波次都要有回退预案
- 数据库迁移细节：Oracle→RDS Oracle 用 DMS 同构迁移（SCT 评估兼容性），Oracle→Aurora PostgreSQL 是异构迁移（Schema Conversion Tool 先转 DDL、应用 SQL 兼容改造、DMS 同步）；大表用并行加载 + LOB 处理，源库归档日志保留要够 CDC 追平
- 数据一致性保障：DMS 迁移期间源库写不停（CDC 持续追），切换窗口做最后一轮同步（追平延迟到 0）→ 停写 → 验证校验和 → 切 DNS/流量；用 DMS 的 Task 校验（Table statistics 对比行数与校验）与 SCT 的 data validation
- 停机窗口拆解：24 小时窗口里数据库追平 + 切换通常占 8-10 小时，Web 层切换只占 1-2 小时，大头是验证与回退准备；务必先做一次"演练切换"（Test Cutover）再正式切换，回退策略=保留源环境至少 2 周
- 有上线后动作：切换后监控对比（延迟/错误率与源环境基线对比）、DNS TTL 提前调低（300s→60s）、新环境做快照基线、旧环境保留期后关机并清理

**减分项：**
- 想把 Oracle 一天迁完，不做 SCT 兼容性评估，应用 SQL 到新库直接报错
- 只做全量复制不做 CDC，切换前源库新数据全丢
- 没有演练直接正式切换，切换当天才发现权限/网络/字符集问题
- 回退预案缺失：源环境直接下线，出问题无路可退
- 文件迁移只做一次性拷贝，没有增量同步，切换窗口内文件不一致

**解答：**

我的原则是"分波次 + 工具各司其职 + 先演练再切换"。工具分工：Web/应用整机用 MGN 做持续块级复制（源端装 agent 后无需停机，切换时一键 cutover，RTO 分钟级）；文件服务器用 AWS DataSync（支持增量同步，切换前最后一轮同步只跑增量）；MySQL 用 DMS 全量 + CDC 增量；Oracle 是硬骨头单独排期：先用 AWS Schema Conversion Tool（SCT）评估兼容性（Oracle 特性、存储过程、隐式转换一票问题先暴露），决定走 RDS Oracle（同构，改动最小）还是 Aurora PostgreSQL（异构，要改应用 SQL），DMS 同步期间持续 CDC 追平，源库 archive log 保留够 3 天确保追得平。波次安排：W1 无状态 Web + 网络基线（VPC/安全组/监控），W2 文件（S3 + DataSync 增量），W3 数据库（DMS 全量 + CDC 持续同步），W4 正式切换周——这个周末窗口只做"追平 + 验证 + 切流量"三件事：周五晚开始最后一轮同步，周六白天追平到延迟 0、停写、SCT/DMS 校验行数与 checksum、切 DNS（TTL 提前从 300s 降到 60s），Web 层 MGN cutover 只占 1-2 小时，Oracle 切换给足 4 小时；切换后灰度放量（先 10% 流量观察 1 小时再全量）。关键纪律：正式切换前必做一次 Test Cutover 演练（发现权限缺失、字符集乱码、应用连接串没改完这类问题都在演练里暴露）；回退预案=源环境保持运行至少 2 周不销毁，切换失败一键回切 DNS 和源库；上线后前 3 天做延迟/错误率与源环境基线对比。这套流程我做过多次，能把"迁移事故"变成"平稳换血"。

**延伸考点：** MGN 的持续复制与 cutover 机制（test/recover/cutover 三步）与源端带宽要求怎么评估？DMS 的 Endpoint 类型（Source/Target）与 replication task 的 LOB 模式（full/limited）对大表迁移的影响？

---

### Q20. 备考 SAA 还是 SAP？认证背后的知识面与设计模式怎么系统化？

**问题：** 你团队要推 AWS 技术认证，一名有 2 年云经验的后端工程师想考证，纠结考 SAA（Solutions Architect Associate）还是直接冲 SAP（Professional）。你会怎么给他做知识面规划，SAA/SAP 的考察差异、核心设计模式与备考要点是什么？

**期望加分项：**
- 讲清两级差异：SAA 考"单项服务是什么、怎么搭、什么场景用什么"（知识面：计算/存储/数据库/网络/IAM 五大类，40-50 道场景题），SAP 考"复杂架构怎么设计、多服务怎么协同、权衡与成本"（涉及迁移、混合云、多区域、性能调优、成本优化，75 道题含大量"两难取舍"场景）
- 给出服务知识面地图：核心必考 20 个服务分层记——计算（EC2/Lambda/ECS/EKS/Fargate/ASG）、存储（S3/EBS/EFS/Glacier/Storage Gateway）、数据库（RDS/Aurora/DynamoDB/Redshift/ElastiCache/DMS）、网络（VPC/ALB/NLB/Route53/CloudFront/GA/VPN/DX/PrivateLink/TGW）、集成（SQS/SNS/EventBridge/API GW/Kinesis/Step Functions）、安全（IAM/KMS/CloudTrail/GuardDuty/WAF/Security Hub）
- SAA 备考要点：以"场景→选服务"为纲（如"无状态+突发→Lambda""强一致事务+SQL→Aurora""读多写少→ElastiCache"），刷题理解"为什么不是 B 而是 C"（通常是可用性、成本、一致性三选一权衡）；官方 Skill Builder + 模拟题 2-3 套，正确率 85%+ 再报考
- SAP 备考要点：重点练"权衡类"题（如：全球低延迟→Global Accelerator vs CloudFront 的区别；成本 vs 高可用→多 AZ vs 单 AZ + 恢复；S3 性能→加前缀 vs 分桶），掌握设计模式：事件驱动（S3→Lambda→SQS）、缓存侧（Cache-Aside/写穿）、消息削峰（SQS+ASG）、Serverless 组合（API GW+Lambda+DynamoDB）、迁移模式（7R：Rehost/Replatform/Refactor/Relocate/Retain/Retire/Repurchase）
- 经验分享：SAP 不是 SAA 的简单升级，需要真实架构经验支撑（笔试前先能独立画出自己的生产架构图并解释每个选择）；推荐路径 SAA（2-3 个月）→ 再 3 个月真实项目/实验 → SAP；用 Well-Architected 六支柱做答题框架（看到题目先想属于哪个支柱）
- 有实验与实践佐证：用 AWS Free Tier 搭一套"API GW + Lambda + DynamoDB + SQS"端到端、用 CloudFormation 重建环境，考试与实际工作互相印证

**减分项：**
- 一上来就冲 SAP，基础服务知识面不全，连 S3 存储类差异都答不清
- 只会背服务特性，不会"场景→方案"的推理（考试全场景题，背八股必挂）
- 刷题只记答案不分析选项，遇到换皮题就懵
- 没有实验环境，纯看文档备考，手生
- 不知道 SAA/SAP 考察的差异定位，备考资料与重点完全错位

**解答：**

定位先说清楚：SAA 考"知识面"，SAP 考"架构力"，两者差一个量级。2 年经验建议先 SAA：它考察"场景→正确服务"的映射，复习主线是把 20 个核心服务按"计算/存储/数据库/网络/集成/安全"六类建知识卡片，每个服务记三件事——是什么、典型场景、与竞品的差异（比如 Aurora vs RDS：存储计算分离 + 15 个读副本；DynamoDB vs Aurora：无 Schema 高性能点查 vs 强一致 SQL）。刷题要重点分析"为什么不选 B"：SAA 的题几乎都是"可用性、成本、一致性"三个维度里让你挑最优解，比如"全球用户访问低延迟"答案是 CloudFront/GA 而非把资源复制到每个区域（成本爆炸）。基础打牢后建议用 3 个月真实项目 + AWS Free Tier 实验搭一套完整架构（API GW + Lambda + DynamoDB + SQS + CloudWatch，用 CloudFormation 重建三遍），再去冲 SAP。SAP 的题是"两难取舍"：给你预算上限、可用性要求、迁移窗口，让你权衡——备考武器是 Well-Architected 六支柱答题框架（看到题先归类"这是考可靠性还是成本？"）与设计模式库：事件驱动、缓存侧（Cache-Aside）、削峰（SQS + ASG 按队列深度扩容）、Serverless 组合、7R 迁移模式（Rehost 用 MGN、Replatform 用 RDS/ElastiCache、Refactor 用 Lambda）——SAP 高频考点是混合云（DX/VPN/TGW 选型）、多区域（Global Tables/DMS 跨区复制）、成本（Savings Plan vs RI vs Spot 的账）。我见过的备考误区是"背一堆服务特性上考场"，SAA/SAP 场景题占 90% 以上，必须练"读题→识别约束→套设计模式→排除法"的肌肉记忆，模拟题正确率稳定 85%+ 再约考，才是稳过的节奏。

**延伸考点：** SAP 的迁移类题中，Rehost（MGN）与 Replatform（DMS + RDS）如何按停机窗口与改造成本权衡？事件驱动设计里"S3 事件直接触发 Lambda"与"经 SQS 削峰"在 SAP 场景题中如何识别最优解？
