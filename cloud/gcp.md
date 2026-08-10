# 云 · GCP（面试题库）

本文件考察候选人在 GCP 平台工程落地上的真实能力：组织层级、计算、容器 GKE、Serverless、存储、数据库、网络、IAM、数据与成本。题目全部为场景化提问、不考八股文——重点看候选人能否给出量化依据、说清取舍、引用真实上线中的 gcloud 配置与踩坑案例、主动覆盖边界条件，难度从实践基础渐进到架构级。每道题的加分项给出明确衡量标准，减分项提示常见翻车点，解答遵循"思路 → 关键方法/配置示例 → 实践中的坑"的递进逻辑。

---

### Q1. 一个多团队、多业务的公司，GCP 组织层级和项目怎么划分？

**问题：** 公司有 3 个业务线（支付、风控、数据平台），每个业务线有开发/测试/生产环境，安全团队还要求统一管控。你会怎么设计 Organization/Folder/Project/Resource 层级？IAM 和配额在层级间如何继承？

**期望加分项：**
- 说清四层结构的职责：Organization 是根（domain 自动创建），Folder 做多级分组（按业务线 → 按环境），Project 是资源边界和配额/IAM/计费的基本单元，Resource（VPC、GKE、实例等）挂在 Project 下
- 给出可落地的划分方案：Folder 顶层按业务线、二层按环境（dev/test/prod），如 `folding` 用 `gcloud resource-manager folders create --display-name=... --folder=...` 创建；付费支付等敏感业务单独 Project 并用 VPC Service Controls 隔离
- 讲清继承规则：IAM 权限向下继承但可被下层"拒绝"规则覆盖；组织级配额是共享池，项目级可查 `gcloud compute project-info describe`，组织级配额在 Organization 层限制
- 提到共享服务设计：网络、日志、审计等共享服务放独立 Folder/Project，避免重复建设；用 Folder 级 IAM 批量授权（如把 `roles/billing.user` 授给每个业务线 Folder）
- 有安全与合规视角：审计日志在 Organization 或 Folder 层聚合导出到专用日志 Project，实现"写入者不能删自己日志"
- 给出量化标准：能用一张树状图在 1 分钟内讲清楚谁在哪个层级能做什么

**减分项：**
- 把 Project 当文件夹用，或把资源直接堆在 Organization 根上，层级混乱
- 说不清 IAM 继承与拒绝规则（Deny）的优先级关系
- 不知道配额分组织级和项目级两层，或把配额理解成全局单一池
- 没有共享服务（网络/审计/日志）的独立规划，各 Project 各自为政
- 只背概念，举不出实际 `gcloud` 命令或真实落地案例

**解答：**

我的划分原则是"业务线隔离、环境分层、共享服务集中"。顶层 Organization 下建两个一级 Folder：`shared-services` 和 `business-lines`。`business-lines` 下按支付、风控、数据平台建二级 Folder，每个二级 Folder 下再按 dev/staging/prod 建三级 Folder，Project 全部建在三级 Folder 下，命名规范 `{biz}-{env}-{app}`。IAM 按层级最小化授权：Organization 层只给平台团队 `roles/resourcemanager.organizationAdmin` 和安全团队 `roles/securitycenter.admin`；Folder 层给业务线负责人 `roles/resourcemanager.folderEditor`；具体角色下放到 Project。这里要特别注意继承规则：上层授予的权限下层默认继承，但可以在下层用 Deny 规则（组织策略 `iam.denyPolicyGeneration`）或拒绝策略显式禁止，优先级上 Deny 高于 Allow。配额是"组织级共享池 + 项目级约束"两层结构，我会在 Organization 层设置全局配额上限（如总 vCPU 8000），各业务线在 Folder 层设子配额，防止某条线把共享池打满影响全公司。共享服务单独建一个 Folder 放网络（共享 VPC host project）、日志汇聚（Organization 级 sink 导出到独立 Project）和审计（Data Access 日志保留 400 天），核心诉求是日志 Project 的权限收敛到只有审计团队可读，业务方"删不掉自己干过的事"。实战中最大的坑是 Project 数量失控（每个小团队建几十个），所以我会定规则：必须通过 Folder 下的 Project Factory（模板 + Terraform）创建，命名和标签由组织策略强制。

**延伸考点：** 如果某业务线要求"支付数据任何 Google 员工都不可见"，除了 IAM 你还需要哪些组织层机制（VPC SC、组织策略限制）？

---

### Q2. 线上 CPU 密集服务经常跑满，怎么选机器类型并控制成本？

**问题：** 一个统计报表服务是 CPU 密集型的，长期稳定运行，目前用 n1-standard-8 但经常 CPU 打满且贵。你会怎么选机器类型？抢占式实例（Preemptible/Spot）、自定义机型（Custom Machine Type）和托管实例组（MIG）分别适合什么场景？

**期望加分项：**
- 讲清 2024 后新一代机型系：通用型 n2/n2d/e2、计算型 c2/c2d/c3、内存型 m2/m3、高性能 H3 等，并给出选择依据（CPU 核数 × 单价 × 预留性能 vs 实际压测数据）
- 说明自定义机型用法与边界：`gcloud compute instances create --custom-cpu=16 --custom-memory=32`（以 0.25GB 为增量），适合"标准档位浪费内存"的场景，如 CPU 密集但内存需求小的服务用高 CPU 低内存定制档可省 20%-30%
- 讲清 Spot（抢占式）的取舍：价格比按需低 60%-91%（具体按资源类型浮动），但 24 小时最多被抢占一次、提前 30 秒通知；适合容错批处理（Spark、渲染、CI），不适合有状态数据库；用 `--provisioning-model=SPOT --instance-termination-action=STOP`
- 讲清 MIG 价值：`gcloud compute instance-groups managed create`，支持自动扩缩（`--scale-based-on-cpu`）、滚动更新、健康检查自愈、区域级多可用区部署；配合 Spot 时用容量池化策略（capacity-pooled）摊薄抢占风险
- 有量化对比：给出同规格按需 vs Spot vs CUD 的年化成本差，如 c2-standard-8 按需月费约 $400+，1 年 CUD 打 70 折、3 年打 55 折左右
- 提到监控兜底：CPU 使用率、MIG 的 current_action 状态、抢占通知日志（`maintenance-event` 类型）都要接告警

**减分项：**
- 还在推荐已淘汰的 n1 系列或说不清 n2/c2/e2 的定位差异
- 把 Spot 用在有状态服务上，答不出"被抢占后流量怎么兜"的方案
- 自定义机型说不清配额与增量规则，或者把自定义机型当成万能省钱工具
- MIG 只答"自动扩缩"四个字，讲不出滚动更新、健康检查、区域冗余
- 无量化数据，给不出任何成本对比或压测依据

**解答：**

先看负载画像再选型：服务长期稳定、CPU 密集，说明它既不适合 Spot（会被抢占影响 SLA），也不适合盲目扩规格（贵）。我会先做压测拿到基线：当前 n1-standard-8 在峰值 CPU 95%，但内存只用 30%，说明"标准档位内存浪费"，此时第一选择是自定义机型 c2-standard CPU 核心按需定——比如用 c2 的 8 核 + 16GB 内存（`gcloud compute instances create --custom-cpu=8 --custom-memory=16 --custom-extensions`），单价按核数计费，相比 n1 标准档通常能省 15%-25%，且 c2 单核性能比 n1 强（Turbo 频率）。第二步：这是稳定负载，直接买 1 年或 3 年 Committed Use Discount（CUD），按区域和机型族购买，1 年约 30% 折扣、3 年约 45%，把"确定性的稳态负载"全部用 CUD 覆盖，能再压 30%-40% 成本。第三步：容量保障与弹性交给 MIG——把服务放进托管实例组，开 `--scale-based-on-cpu` 目标利用率 70%，配健康检查（TCP 探测 80 端口），MIG 会自动在多个 zone 分布实例；高峰期弹性扩容的增量实例用 Spot 容量池，反正报表是幂等任务、挂一台自动补一台。这里要分清角色：稳态核心实例用 CUD + 按需，弹性的边界实例用 Spot，两者混在同一 MIG 的不同实例模板或不同 MIG 中，通过区域级 MIG + capacity-pooled 策略摊薄抢占风险。实践中的坑：一是忘记开启 `--instance-termination-action`，Spot 被抢占后实例默认直接删、未落盘数据丢；二是 MIG 缩容策略默认删旧实例，配置 `--update-policy type=PROACTIVE --max-surge` 不当会导致滚动更新期间流量抖动；三是自定义机型要确认项目配额支持（vCPU 配额按区域核算，`gcloud compute regions describe` 可查）。

**延伸考点：** Spot 实例被抢占的通知通过什么机制到达（metadata + 事件），你会怎么用它做优雅退出？

---

### Q3. 新项目用 GKE，Autopilot 还是 Standard？集群和节点池怎么规划？

**问题：** 公司要上一个微服务项目（约 20 个服务，K8s 经验一般，SRE 人力少），想用 GKE。你会选 Autopilot 还是 Standard 模式？节点池、网络和升级策略怎么设计？GKE 相比自建 K8s 或者其它托管 K8s 的核心差异是什么？

**期望加分项：**
- 给出明确选型标准：Autopilot 适合"不想管节点、对成本敏感度中、SRE 人力少"（按 Pod 计费、自动节点池、强制最佳实践）；Standard 适合"需要定制节点（GPU、特殊 taint）、对 Pod 密度和成本精细控制、已有 K8s 运维能力"的团队，能说出 2024 年后 Autopilot 已支持 GPU 和容量上限等边界变化
- 说清 Autopilot 关键约束：按资源类目计费（CPU/内存/临时盘/GPU 分档定价）、Pod 请求必须显式声明、`system` 命名空间受限、不能访问节点、无法自定义 kubelet 参数——能举出反例场景（如 DaemonSet 依赖宿主网络/特权容器会受限）
- 讲清节点池设计：Standard 下多节点池按"业务类型 + 弹性策略"拆分（如 `default-pool` 用标准机型配 CUD，`spot-pool` 用 Spot + taint 只允许批处理调度），用 `gcloud container node-pools create --machine-type=c2-standard-8 --spot` 示例；GPU 节点池单独建并配 `--accelerator type=nvidia-l4`
- 提到 GKE 的托管增值：自动升级（Rapid/Stable/Static release channel）、节点自动修复、GKE Sandbox（gVisor）、Workload Identity（不用 long-lived service account key）、GKE Autopilot 的默认安全加固（如默认启用 workload identity 与容器优化 OS）
- 有版本与升级策略视角：`gcloud container clusters update --release-channel stable`，用维护窗口 + surge 节点（`--node-pool-soak`）做无损升级
- 有实践佐证：如"把某有状态 Redis 从 Standard 迁到 Autopilot 失败，因为需要 hostPath/特权"这类反例

**减分项：**
- 只背"Autopilot 免运维"，讲不出它限制了什么、什么场景不该用
- 说不清 GKE 与自建 K8s 的差异点（managed control plane、release channels、Sandbox、Workload Identity）
- 节点池规划只有"一个池到底"，没有弹性/成本/隔离的分池思路
- 不知道 Autopilot 按 Pod 计费与 Standard 按节点计费的成本模型差异
- 升级、维护窗口、surge 等生产化细节一片空白

**解答：**

我的判断框架是"运维人力 × 定制需求 × 成本模型"三维选型。该项目 20 个微服务、K8s 经验一般、SRE 人力少，我倾向 Autopilot：免管节点池，按 Pod 资源类目计费（CPU/内存分别按档定价），默认启用 Workload Identity、容器优化 OS、网络策略等安全基线，团队只需写 Deployment 和 HPA。但前提是业务不需要宿主级能力——如果要做 DaemonSet 采集宿主日志、特权容器、自定义 kubelet 参数，或者追求 Pod 密度最大化摊薄成本，就选 Standard。假设选 Standard，节点池我会拆三份：`default-pool`（按需 + CUD 覆盖稳态）、`spot-pool`（`--spot --node-taints=spot=true:NoSchedule`，只让批处理和可重试 Job 调度，DaemonSet 由 pod affinity 绑定）、`gpu-pool`（单独建，配 nvidia-l4，加 `--maintenance-policy` 与污点隔离）。网络用 VPC 原生集群（`--enable-ip-alias`，默认开启，Pod 直接拿 VPC 内 IP，便于和 Cloud SQL 等内网资源互通），节点池放多个 zone 保证可用性。升级策略：用 release channel 的 stable 通道 + 维护窗口（凌晨 2-4 点），滚动升级配 `--surge-upgrade --max-surge=1 --max-unavailable=0`，保证不丢 Pod 容量。生产坑提醒：一是 Standard 下忘记开 Workload Identity，团队图省事把 `GOOGLE_APPLICATION_CREDENTIALS` 的 JSON key 塞进 Secret，这是最常见的坏味道；二是 Autopilot 下 Pod 不声明 resources 会直接被拒绝，且 `spec.hostNetwork` 等字段会被 webhook 拒绝，迁移前要扫一遍清单；三是多租户集群没配 ResourceQuota/LimitRange，一个服务 OOM 把整池打满导致全部 Pending。最后补一句 GKE 相比自建的价值：control plane 托管 + 自动升级 + 节点自动修复 + Sandbox（gVisor）无感隔离 + 原生与 Google 生态（Cloud Monitoring/KMS/IAM）打通，省的是"造轮子"的运维工时。

**延伸考点：** Autopilot 的 Pod 计费模型下，哪些资源（如 GPU、临时盘、内部网络出口）是额外计费或需要显式声明的？

---

### Q4. 一个低流量内部工具和一个人脸识别 API，Serverless 三件套怎么选？

**问题：** 团队要上两个服务：一个是只有内部 10 个人用的管理后台（低频、偶发调用），一个是人脸识别 API（并发高、需要流控和长超时）。在 Cloud Run、Cloud Functions、App Engine 里怎么选？Cloud Run 的自动伸缩（min/max instances、并发度）到底怎么配？

**期望加分项：**
- 给出清晰选型矩阵：低频内部工具用 Cloud Functions（或 Cloud Run 默认 0→1）成本最优；长跑/流控/高并发 API 用 Cloud Run（容器化、支持请求级并发、可配 CPU 分配与超时上限）；App Engine 适合"不想管容器也不想管函数的传统 Web 应用"，说出 App Engine Standard vs Flexible 的差异
- 讲透 Cloud Run 自动伸缩：`--min-instances` 决定冷启动兜底、`--max-instances` 防止突发打爆预算，`--concurrency`（默认 80）决定单实例并发请求数，配 CPU 空闲分配（`--cpu-always` 或 request-based）平衡冷启动与成本
- 有冷启动优化实践：min-instances=1 + 预热请求、缩小镜像（distroless）、用 start-up probe，把冷启动从 2s 压到 300ms 量级
- 提到 Cloud Functions 的定位变化：2nd gen 基于 Cloud Run，适合事件驱动（Pub/Sub、GCS 触发）而非重请求处理；说出并发上限、超时上限（gen2 60 分钟 vs gen1 9 分钟）等硬性边界
- 有成本量化：Cloud Run 按 CPU/内存 × 秒计费、请求数另计，低流量场景月度成本可低至几美元；对比常驻 GKE/VM 的 idle 成本
- 提到限流与鉴权细节：Ingress 类型（internal/all）、`--no-allow-unauthenticated`、通过 Gateway/负载均衡统一入口

**减分项：**
- 把三件套当成"都是 serverless，随便选"，答不出各自的取舍和限制
- 说不清 concurrency、min/max instances 的语义，或把 max instances 当摆设
- 不知道冷启动是什么、怎么解决，对 min-instances 的成本含义无感
- 用 Cloud Functions 硬扛高并发请求流量（它面向事件，不适合长连接/流控场景）
- 说不出任何成本量级或配额边界

**解答：**

先区分"事件驱动"和"请求驱动"：管理后台是人偶尔点一下，属于低频请求，Cloud Functions 2nd gen 最合适——基于 Cloud Run 的 gen2 支持最长 60 分钟超时，代码即函数，按调用次数 + 资源秒计费，10 人内部工具月成本可以压到几美元，冷启动对这个场景无所谓。人脸识别 API 则不同：它要承受并发、需要控制实例数防止打爆下游费用、可能有 30s+ 的推理超时，这属于请求驱动服务，用 Cloud Run 容器化部署（Dockerfile + `gcloud run deploy face-api --image=... --region=asia-east1 --concurrency=20 --max-instances=50 --cpu=2 --memory=4Gi --cpu-always --no-allow-unauthenticated`）。自动伸缩的关键是理解三个参数的关系：`concurrency` 是单实例能同时处理的请求数，决定"一个实例能顶多少 QPS"；`max-instances` 是成本安全阀（50 个实例 × 4Gi 内存就是硬顶）；`min-instances` 是冷启动保底——我会把 min 设 2 并接 Cloud Scheduler 每 5 分钟发一次预热请求，把推理冷启动（模型加载可能要 2-5 秒）摊掉，用户无感。App Engine 在这套方案里用不上：它适合"传统 Web 应用不想容器化"的场景，Standard 环境运行时受限、Flexible 可以带 Docker，但现在 Cloud Run 能覆盖它 90% 的需求，我一般只在既有 App Engine 存量业务迁移时遇到。实践中的坑：一是 concurrency 配成默认 80 但函数是 CPU 密集推理，单实例并发 80 会让推理排队、P95 恶化，正确做法是按压测把 concurrency 调到 20；二是忘了 `--cpu-always`，请求间隙 CPU 被回收，导致偶发请求又要冷启动；三是 Cloud Run 每个实例有 1000 个并发连接上限（及 QPS 配额），别把 max-instances 当无限容量。管理后台和 API 我还会统一挂到 Gateway（API Gateway 或 GLB）做鉴权与限流，避免直接暴露公网端点。

**延伸考点：** Cloud Run 的 `--cpu-always` 与默认 request-based 模式对计费的影响是什么？低延迟服务为什么必须开它？

---

### Q5. 数据湖用 GCS 还是 S3？桶设计、存储类与生命周期怎么配？

**问题：** 团队要把数据湖从 AWS S3 迁到 GCS（或新搭），数据分热（近 7 天查询）、温（月度报表）、冷（归档审计）三档。你会怎么设计桶结构、存储类（Standard/Nearline/Coldline/Archive）、生命周期规则和对象版本？GCS 与 S3 在概念和用法上有什么差异？

**期望加分项：**
- 讲清 GCS 与 S3 的核心差异：统一命名空间（单桶、无 region 前缀的全局命名空间 vs S3 的 bucket+region）、无"前缀即目录"（GCS 是扁平对象但控制台按 `/` 模拟目录）、强一致读写（所有操作 read-after-write 强一致，S3 早期是最终一致）、对象版本是显式开关（`gcloud storage buckets update gs://xxx --versioning`）、无 S3 的 bucket policy 但用 IAM + 对象 ACL 双轨
- 给出存储类选择：近 7 天热数据 Standard（min storage duration 无/低），月度报表 Nearline（30 天最小存储期，单价更低），归档审计 Archive（365 天最小存储期，最便宜）；说清最小存储期（min storage duration）对成本的影响——提前删除要付满最小存储期的钱
- 生命周期规则示例：`gcloud storage buckets update --lifecycle-file=lifecycle.json`，JSON 规则把 `STANDARD → NEARLINE`（30 天后）→ `COLDLINE`（90 天）→ `ARCHIVE`（365 天），超 730 天删；并说明用 `Age` 与 `daysSinceCustomTime` 做条件
- 桶设计：按"业务域/环境/数据层"分桶而非一个大桶，如 `gs://dl-pay-raw`、`gs://dl-pay-curated`；用标签（labels）做成本分摊，统一加 `--public-access-prevention=enforced`
- 有迁移实践：`gcloud storage cp -r --recursive`（替代 gsutil 的新 CLI）、`storage transfer service` 做 S3→GCS 迁移、`gsutil rsync` 增量同步；提到加密默认用 Google 托管密钥，可用 CMEK 自管
- 有量化视角：能给出各存储类的大致单价量级与不同访问频率下 Total Cost of Ownership（TCO）对比

**减分项：**
- 把 GCS 当成"换皮 S3"，说不出强一致、扁平命名空间、bucket policy 缺失这些本质差异
- 存储类只背名字，说不清最小存储期、取回费（retrieval cost）与提前删除的代价
- 生命周期规则讲不出转冷/删除的具体配置和条件字段（Age、StorageClass）
- 桶设计没有分层或成本分摊意识，一桶打天下
- 不会用现代 CLI（还在说 gsutil 的旧命令）或说不出迁移工具

**解答：**

桶结构我会按"数据域 + 环境"建三层：`gs://{domain}-raw`（源数据落盘）、`gs://{domain}-curated`（清洗后）、`gs://{domain}-gold`（供查询服务），跨环境再分 `-dev/-prod`，所有桶开启 `--public-access-prevention=enforced` 并加 labels（team/cost-center）。存储类三档对应数据生命周期：raw 落盘先用 Standard 支持近 7 天热查询，7 天（按 Age 规则）自动转 Nearline 供月度报表，90 天转 Coldline，365 天转 Archive 做审计归档，730 天删除。关键成本认知是**最小存储期**：Nearline 最小存储期 30 天、Coldline 90 天、Archive 365 天，如果数据在期限前删除，仍要付满期限的钱——所以生命周期规则要配"转冷时间 ≥ 最小存储期"，否则出现"数据删了钱照扣"。生命周期 JSON 示例：`{"lifecycle":{"rule":[{"action":{"type":"SetStorageClass","storageClass":"NEARLINE"},"condition":{"age":30}},{"action":{"type":"Delete"},"condition":{"age":730}}]}}`。与 S3 的差异要答到点子上：GCS 所有读写都是强一致（S3 历史上 GET 可能读到旧值）；对象版本需显式开启且删除时记得挂 `--version` 保留历史；GCS 没有 bucket policy，权限靠 IAM（推荐）和对象 ACL（老工具/公共读才用）；命名空间全局唯一且桶名不能带大写。迁移执行我会用 `gcloud storage` 新 CLI（`gcloud storage cp --recursive` 支持并行分片）或 Storage Transfer Service 做 S3→GCS 迁移（支持增量、可保留对象元数据），小批数据用 `gcloud storage rsync` 增量同步。实战坑：一是 gsutil 老命令对超大文件要手动调 parallelism，新 CLI 自动；二是忘了开版本控制，被误删/被勒索覆盖后无法恢复；三是单桶存不同生命周期的对象导致冷热混存，规则条件（Age 从上传日算）会把热数据提前转冷。

**延伸考点：** 什么是 gzip 内容编码与 `--gzip` 标志的区别？对象以压缩形式存储时元数据里 `content-encoding` 怎么影响下载端解压？

---

### Q6. 报表、订单、时序、会话四类数据，GCP 数据库怎么选？

**问题：** 公司有四种数据：①订单/支付（强一致事务）、②100 亿行日志指标（低延迟读写、单 key 查询）、③用户会话/实时状态（全球多活）、④月度报表（分析型聚合）。Cloud SQL、Spanner、Bigtable、Firestore（和 BigQuery）分别对应哪个？选型依据和迁移陷阱是什么？

**期望加分项：**
- 给出正确映射：订单/支付 → Cloud SQL（事务型 OLTP）或需要水平扩展时 Spanner；海量时序/日志 → Bigtable（宽列、单行低延迟、吞吐高）；会话/多活 → Firestore（无模式文档、全球多区域强一致/最终一致可选）；报表分析 → BigQuery（列式、MPP）
- 讲清"为什么"而非只贴映射：Bigtable 是宽列存储，适合"行 key 前缀扫描 + 单行读写"，不适合 JOIN 和聚合；Spanner 是全球分布式 SQL + 强一致事务（TrueTime），代价是贵且 schema 要按 interleave 设计；Firestore 的强一致模式跨区域时性能开销大，多活场景要评估一致性与成本
- 有量化依据：单条 Bigtable 读延迟 5-10ms、写吞吐按节点（每节点约 10k QPS 量级，按实际配置）线性扩展；Spanner 按节点/区域计费，小业务成本是 Cloud SQL 的 5-10 倍，所以"不是所有事务都要 Spanner"
- 提到托管迁移要点：Cloud SQL 用 Database Migration Service 从 MySQL/PostgreSQL 迁移（CDC 增量，源端只读秒级切换）；从自建到 Cloud SQL 的坑（字符集、时区、最大连接数、高可用切换只读副本）；Bigtable 从 HBase 迁移用 `cbt`/HBase API 兼容层
- 有选型流程：先按"事务强度 × 规模 × 一致性 × 查询模式"打分，再给每个场景一个备选方案（如订单也可用 Spanner + Bigtable 混合、会话也可用 Redis，但要说明取舍）
- 提到成本与运维视角：Cloud SQL 支持 CUD、Spanner 按节点计费、Bigtable 按节点计费（最小 3 节点做 HA）、Firestore 按读写次数计费——这些是选型的硬约束

**减分项：**
- 四类数据对应四家数据库直接贴标签，说不出选型依据和反例
- 把 Spanner 当"Cloud SQL 升级版"无脑推，忽略成本量级
- Bigtable 说不清宽列模型与行 key 设计对性能的决定性影响（hotspot 问题）
- 迁移方案只说"官方工具"三个字，讲不出停机窗口、增量、回滚
- 不考虑一致性需求（强 vs 最终）对选型的影响

**解答：**

我先按"事务性 × 规模 × 查询模式"拆：①订单/支付是强一致 OLTP，默认 Cloud SQL（PostgreSQL）——单实例 + 高可用（跨 zone 主备 + 只读副本扩展读），事务完整支持；只有当订单量到"单库扛不住写入"且业务必须 SQL 事务时，才升级 Spanner（全球强一致事务，但节点计费贵、schema 要按主键 interleave 设计父子表）。②100 亿行日志指标是典型宽列场景，用 Bigtable：单行 key（`{device}#{ts}`）读写 5-10ms、按节点横向扩展吞吐，key 设计避免热点（时间戳做 key 前缀会把热写入打到一个节点，正确做法是设备 ID 前缀 + 时间戳），它不支持 SQL 聚合，分析侧用导出到 BigQuery 或 Dataflow 管道。③会话/多活用 Firestore：无模式文档 + 自动多区域复制，选 strong 一致性或 eventual 模式，天然支持移动端/Web 实时监听；注意它的事务是文档级（跨文档事务有限制），且强一致跨区域延迟高，不是订单系统的替代品。④报表分析归 BigQuery，和 ①是"读写分离"关系：Cloud SQL 的变更用 CDC（如 Striim/Dataflow/DMS）同步进 BigQuery，报表查询打 BigQuery 不打业务库。迁移执行：Cloud SQL 用 Database Migration Service 一键迁移（先全量后 CDC 增量，源端只需开放复制账号），切换时注意把 max_connections、时区（`time_zone=+08:00`）、字符集对齐；Bigtable 从 HBase 迁用 HBase API 兼容层 + `cbt` 工具，行 key 规则若原先设计差，迁移时要重设计而非硬搬。最容易翻车的判断：把"全球部署"直接等于 Spanner——其实 Firestore 多区域 + 最终一致性在大多数会话场景更便宜；把"需要横向扩展"直接等于 Bigtable——其实 Cloud SQL 只读副本 + 分区也能撑到相当量级。

**延伸考点：** 订单表以后要按用户维度做审计查询，Bigtable 的行 key 怎么设计避免热点且支持范围扫描？

---

### Q7. VPC 网络怎么规划？私有子网、防火墙、Cloud NAT 和共享 VPC 各解决什么问题？

**问题：** 公司要建 GCP 私有网络：应用跑在 GKE 里需要出公网访问外部 API，但不想给每个节点绑公网 IP；同时多个业务 Project 想复用一套网络。VPC 网络、子网、防火墙规则、Cloud NAT、共享 VPC 怎么组合设计？

**期望加分项：**
- 讲清 VPC 基础模型：VPC 是全局资源（跨 region 存在），子网是区域资源；auto mode 子网自动分配（不推荐生产）vs custom mode 手动划段（推荐，CIDR 可规划）；防火墙规则是"有状态 + 隐式 deny all"模型，默认允许 egress、拒绝 ingress，规则按 priority 生效
- 说清 Cloud NAT 用途与配置：给无公网 IP 的实例提供出网，`gcloud compute routers create` + `nat create`，绑定 router 和子网，可配端口分配（每 VM 默认 64 端口，SNAT 耗尽报 429 的坑）；Cloud NAT 不是入站方案，入站用负载均衡 + 公网 IP
- 讲透共享 VPC：host project 建 VPC/子网，service project 以"子网使用者"身份接入（`--subnets` 授权），好处是统一出网/防火墙/审计、集中 NAT；要点是防火墙规则定义在 host project、GKE 集群可部署在共享子网（需 `shared VPC` 支持）
- 有安全视角：防火墙最小开放（仅内网 3306 对特定网段、仅 LB 的 health check 源段 130.211.0.0/22 和 35.191.0.0/16 放行 80）、VPC Flow Logs 开启做审计、Private Google Access 让无外网实例访问 Google API（`--no-address` + `googleapis.com` 走内网）
- 有实践坑：Cloud NAT 无 IPv6 支持（需单独方案）、NAT 端口耗尽导致间歇性超时、auto mode 子网无法按需扩容、防火墙规则数量超 500 条（或组织策略限制）等
- 能给出 CIDR 规划示例：`10.10.0.0/16` 拆 `10.10.0.0/20`（prod）、`10.10.16.0/20`（staging）等，预留 /16 扩展位，避免 172.17.x.x 与 Docker 冲突

**减分项：**
- 说不清 VPC 全局与子网区域的关系，或把 VPC 当 region 资源
- 不知道 auto vs custom 模式差异，生产还推荐 auto mode
- 把 Cloud NAT 当"负载均衡"或"入站方案"，或说不清 NAT 端口耗尽
- 共享 VPC 只知道"共用网络"，讲不出 host/service project 权限模型
- 防火墙只有"开 80/443"，没有默认拒绝、源 IP 限制、隐式规则意识

**解答：**

设计顺序是"先划段、再隔离、后出网"。网络层：一个 custom mode VPC `corp-vpc`，CIDR 规划 `10.0.0.0/8` 内按业务域切子网：`10.10.0.0/20` 给 prod GKE、`10.10.16.0/20` 给 staging、`10.10.32.0/20` 给数据库专属子网（只允许应用网段访问 3306），每子网绑定具体 region（子网是区域资源，跨 region 部署要分别建）。VPC 本身全局，因此防火墙规则、路由跨 region 一致，这是和 AWS VPC（区域级）的本质差异。出网：所有工作负载不绑公网 IP（`--no-address`），用 Cloud NAT 统一出网——`gcloud compute routers create nat-router --region=asia-east1 --network=corp-vpc`，然后 `nat` 绑定要出网的子网并开启 minPortsPerVm（默认 64，高并发服务调到 512 避免 SNAT 端口耗尽），NAT 出口 IP 固定为保留 IP 便于下游白名单；同时开 Private Google Access 让实例直连 Google API 内网地址，出网流量不绕公网。多 Project 复用：用共享 VPC——host project 放 `corp-vpc`，各业务 Project 作为 service project 申请子网使用权（`gcloud compute shared-vpc associated-projects add` + 在子网级别授权），业务方在自己的 Project 里建 GKE/VM，但网络、防火墙、NAT 统一归平台团队管，审计流量集中。防火墙按"默认拒绝 + 最小开放"写：放行 `10.10.0.0/20` → 数据库子网 3306、LB health check 源段（130.211.0.0/22、35.191.0.0/16）到各应用 80/443，其余 ingress 全 deny。实战坑：一是 auto mode 新建子网会被自动分 172.17.x.x 段，和本地 Docker 网段冲突导致 VPN 路由打架，所以生产必须 custom mode；二是共享 VPC 下防火墙规则只能建在 host project，service project 建规则不生效，新人常在这卡壳；三是 Cloud NAT 每个 VM 默认端口配额，短连接高并发会先耗尽 SNAT 端口（现象是间歇性 connect timeout），要么提 minPortsPerVm 要么改长连接。

**延伸考点：** 在共享 VPC 场景下，防火墙规则应该建在 host project 还是 service project？不同租户的安全隔离靠什么实现？

---

### Q8. 电商促销活动，全球用户访问，负载均衡怎么设计？

**问题：** 电商要做全球促销：页面静态资源走 CDN，API 是后端微服务（GKE 内），另外还有两个 UDP 游戏服务。用 GLB（HTTP(S)）、ILB、Network LB（TCP/UDP）怎么分工？健康检查、跨区域流量调度（anycast + 就近路由）怎么配？

**期望加分项：**
- 讲清三种负载均衡定位：HTTP(S) Load Balancer（GLB）是 L7、全球 anycast 单 IP、支持 URL 路由/SSL/内容缓存，适合 API 和 Web；Internal Load Balancer（ILB）是 L4 内网流量（微服务间、数据库前）；External TCP/UDP Network LB 是 L4 公网、支持 UDP/长连接、后端必须用 forwarding rule + backend service
- 说清 GLB 全球架构：single anycast IP（IPv4/IPv6），流量经 Google 边缘进入就近 region 的后端（`--global` forwarding rule + `--load-balancing-scheme=EXTERNAL_MANAGED`），后端 NEG（GKE 用 `--network-endpoint-groups` 的 GCE_VM_IP_PORT 或 POD 类型）而非 VM 实例组，实现 Pod 级负载
- 健康检查配置要点：`gcloud compute health-checks create http hc-api --port=80 --request-path=/healthz --check-interval=5 --timeout=5 --healthy-threshold=2 --unhealthy-threshold=3`，后端配置 `--max-rate-per-endpoint` 限流，会话保持（cookie）按需开启
- 有量化与容量视角：GLB 自动弹性、无预置容量，可承受 TB 级突发；配合 Cloud CDN 缓存静态资源把源站 QPS 降 80%-90%；跨区域 fallback 靠后端在多 region 部署 + 就近路由（自动）
- 有实践坑：UDP 服务必须用 Network LB（GLB 不支持 UDP），health check 对 UDP 后端用 TCP health check 兜底；NEG 与 MIG 后端的选择（Pod 直连 vs VM 轮询）、连接耗尽（client IP affinity 影响哈希）、证书管理（`gcloud compute ssl-certificates create` 配 Let's Encrypt/Google CA）
- 提到内部流量与东西向：ILB 做内部服务发现 + 统一鉴权入口，配合 Cloud Service Mesh 可做灰度

**减分项：**
- 分不清 L4/L7 与内外网负载均衡的适用场景，把 UDP 硬塞给 GLB
- 不知道 GLB 是全球 anycast 单 IP、边缘就近路由这些核心特性
- 健康检查参数（interval/timeout/threshold）讲不出合理值或含义
- 后端组是"实例组"还是"NEG"分不清，说不出 Pod 级负载的实现方式
- 无容量/成本视角，答不出 CDN 卸载、突发流量下 LB 是否要预置

**解答：**

促销场景我会拆成三条流量线。第一条：静态资源（图片/前端 bundle）→ Cloud CDN 挂在 GLB 后端，缓存 TTL 按资源类型设（immutable 资源 30 天、HTML 5 分钟），源站命中率压到 10% 以下，QPS 从峰值的 50 万压到 5 万。第二条：API 请求 → HTTP(S) Load Balancer，`--global` forwarding rule 得到一个全球 anycast IP，用户就近接入 Google 边缘、路由到最近 region 的 GKE 后端——后端配 NEG（`--network-endpoint-groups` POD 类型）而不是实例组，这样流量直达 Pod，避免多一跳；后端服务设 `--max-rate-per-endpoint=100`（QPS 级限流）和会话亲和（`--session-affinity=GENERATED_COOKIE`，促销购物车场景需要）；健康检查用 `GET /healthz`，间隔 5s、3 次失败摘除、2 次成功恢复，GKE 里的 Pod 必须实现 liveness/readiness 与之配合。第三条：游戏 UDP 服务 → External TCP/UDP Network LB，L4 层、支持 UDP 和长连接，按 IP/端口转发，后端可以是实例组，health check 用 TCP（UDP 没有健康探测语义，用端口连通性兜底）。跨区域高可用：核心 API 在 asia-east1 和 us-central1 双活部署，GLB 就近路由 + 健康检查自动摘除故障 region，促销前做容量压测确认单 region 能扛全量流量（防止"一个 region 挂了全部流量打到另一个 region 直接击穿"）。实践中的坑：一是 GLB 后端如果是 MIG 实例组，流量是"LB→VM"，Pod 分布不均时会有热点，必须用 NEG 直达 Pod；二是 UDP 服务器集群用 Network LB 时后端要开启 `--session-affinity=CLIENT_IP`（UDP 无 cookie），否则同一用户会话漂移；三是促销前忘了预热 CDN，缓存全空、源站被击穿——提前用脚本批量预热热 URL；四是证书要提前申请并做好续期自动化（`gcloud compute ssl-certificates` 支持 Google 托管 CA 自动续期）。最后，内部微服务间调用用 ILB 做统一入口 + Service Mesh 灰度，公网入口与内网入口分开，避免内网服务暴露公网。

**延伸考点：** GLB 的 Global external vs Regional external 有什么区别？什么场景必须用 regional 版本？

---

### Q9. 权限治理：primitive、predefined、custom 三种角色怎么用？服务账号密钥怎么管？

**问题：** 团队权限混乱：有人有 Project Owner、有人用个人账号跑生产脚本、服务账号 JSON key 散落在代码仓库里。你要怎么重构 IAM？primitive/predefined/custom 角色的区别和使用场景是什么？条件角色绑定（conditional IAM）怎么用？

**期望加分项：**
- 讲清三类角色本质：primitive（Owner/Editor/Viewer）是项目级、宽泛、易过度授权，Google 不推荐新系统使用（保留兼容）；predefined 是按服务设计的粒度角色（如 `roles/cloudsql.admin`、`roles/storage.objectViewer`）；custom 是在权限级别（permission 如 `storage.objects.create`）自定义组合，适合"predefined 过宽但权限组合独特"的场景，且仅支持项目级创建
- 说清服务账号（Service Account）最佳实践：一个服务一个 SA、最小权限、用 Workload Identity（GKE）或 `--impersonate` 代替下载 JSON key，短期凭证（`gcloud auth application-default login`、metadata server 取 token）替代长期 key；key 要定期轮换、放 Secret Manager 而非仓库
- 有条件 IAM 实战：`--condition='resource.name.startsWith("projects/_/buckets/prod-")'` 或基于时间窗口的临时授权（如值班账号只在 9-18 点有权限），用 condition 做资源级、时间级、来源级（IP）的限制
- 有审计与最小权限流程：`gcloud projects get-iam-policy` 导出做定期 review，用 Policy Analyzer（`gcloud policy-intelligence`）发现过度授权；组织策略禁止 primitive 角色（`constraints/iam.allowedPolicyMemberDomains` 等）
- 有事故案例佐证：如"共享 JSON key 泄露导致 GCS 数据被拉取"、"长期不轮换的 SA key 被用于横向移动"
- 能说清 IAM 的评估模型：Allow 优先 + 下层覆盖，Deny 最高优先级，`allUsers`/`allAuthenticatedUsers` 是公网暴露的源头

**减分项：**
- 三种角色只背定义，说不出什么时候选 custom 什么时候用 predefined
- 对服务账号 JSON key 的泄露风险无感，还在推荐"把 key 放进 CI 变量"
- 不知道条件 IAM 能做什么（资源名、时间、来源 IP 条件）
- 不知道 `allUsers` 绑定会导致公网可访问，排查"为什么桶公网可读"无从下手
- 说不出权限审计/过度授权排查手段

**解答：**

重构分三步。第一步止血：全局搜索代码仓库和 CI 配置里的 service account JSON key，全部作废并轮换，生产环境访问改用 Workload Identity（GKE 里 Pod 用 `--workload-metadata=GKE_METADATA` 绑定 SA，通过 `gcloud iam workload-identity-pools` 或 GKE Workload Identity Federation 免 key 取短期 token）；本地开发用 `gcloud auth application-default login` + 临时凭据，绝不落盘 key。第二步定角色规范：新建项目一律禁止 primitive 角色（用组织策略 `constraints/iam.denyPrimitiveRoles` 之类强制），全部走 predefined + custom。predefined 能满足 80% 场景——运维给 `roles/compute.admin`、数据团队给 `roles/bigquery.dataEditor`、只读给 `roles/storage.objectViewer`；只有当"predefined 要么过宽要么缺失"时才建 custom，比如"只能读写 prod 桶里 report 前缀对象"这种组合，custom 角色按 permission 粒度拼（`storage.objects.get/create`），并配条件 IAM 限定资源：`--condition=resource.name.startsWith("projects/_/buckets/dl-prod/report")`。第三步上条件与审计：值班授权用时间条件（只在 on-call 窗口生效）、敏感操作加来源 IP 条件；用 Policy Analyzer 定期扫描"谁有 Owner/Editor"，把人均权限数作为治理指标（如 Owner 人数从 15 降到 3）。这里要讲清评估模型：IAM 是 Allow 累加 + Deny 绝对优先，`roles/owner` 之外的高危绑定是 `allUsers`/`allAuthenticatedUsers`——90% 的"桶突然公网可读"事故都是某人在 ACL 或 IAM 里绑了 `allUsers`，排查时 `gcloud projects get-iam-policy` + grep 这两个主体。实战坑：一是把 custom 角色建在项目 A、想在项目 B 用，导致"角色不存在"报错（custom 角色是项目级的，跨项目要在每个项目建或等组织级 custom 角色支持）；二是条件表达式写错（如 `resource.name` 前缀匹配对 folder/project 不生效），条件 IAM 对某些资源类型（如 VPC）支持有限，要查文档确认；三是 SA 权限给在"人"身上——SA 应当只绑定给工作负载，人用用户账号 + 组管理，避免"离职了密钥还在跑"。

**延伸考点：** Workload Identity Federation 和传统 SA JSON key 相比，在凭据轮换与泄露面控制上有哪些根本差异？

---

### Q10. 合规审计要求数据加密密钥自管，CMEK 怎么落地？安全监控怎么搭？

**问题：** 合规要求：①磁盘、GCS 对象、BigQuery 数据的加密密钥必须由公司自管；②安全团队要能看到全组织的风险与审计日志。Cloud KMS、CMEK、CSEK、Security Command Center、Cloud Audit Logs 各怎么用？审计日志怎么防篡改？

**期望加分项：**
- 讲清加密层级：Google 默认用 Google-managed keys；CMEK（Customer-Managed Encryption Keys）用 Cloud KMS 的 key 加密数据（`--kms-key` 作用于磁盘、GCS、BigQuery），删除 key 等于永久销毁数据；CSEK（Customer-Supplied Keys）是自己提供 key 给 Compute Engine/GCS（每次调用带 key），更重但最可控，已不建议新系统
- 说清 Cloud KMS 设计：key ring（按 region 或 global）+ key + version，支持自动轮换（`--rotation-period=90d` 与 `--next-rotation-time`）、key 权限独立于数据权限（`roles/cloudkms.cryptoKeyEncrypterDecrypter`），软删除/计划销毁时间可撤销
- 有安全监控落地：Security Command Center 聚合 IAM 扫描（过度授权）、CVE 漏洞（GKE/VM 镜像）、敏感数据暴露（`asset discovery` + 自定义 detector），免费层含核心扫描，高级层（Premium）才有事件检测和威胁检测；能举出"SCC 扫描到 VM 挂公网 22 端口 + 弱密码"这类实践
- 讲透审计日志分层：Admin Activity（默认开启、400 天）、Data Access（需显式开启、默认 90 天）、System Event、Policy Denied；用 `gcloud logging sinks create` 把日志导出到不可变存储（BigQuery/Cloud Storage with retention lock），实现"写入者不能删"
- 有防篡改方案：Cloud Storage bucket 开启 retention policy（`--retention-period`）+ Bucket Lock（不可撤销）、BigQuery 表用时间旅行/快照；日志导出到独立 project 并收紧权限
- 有量化：数据加密在 CMEK 下对性能影响 <5%（通常可忽略）、KMS 请求量级与费用（每 10k 操作约 $0.06 量级）

**减分项：**
- 分不清 CMEK 和 CSEK 的差别（key 谁管、怎么交互），或把 KMS key 和"给数据库加密码"混为一谈
- 删 key 等于删数据的后果没概念，或不知道 KMS key 销毁有软删除窗口
- 审计日志只提"开启"，说不出三类日志区别与 Data Access 需显式开启、默认保留期
- 不知道 SCC 免费层与高级层的差异，或不知道它扫什么（IAM/CVE/公网暴露）
- 说不清"防止日志被删/改"的手段（retention lock、独立 project、sink 只写）

**解答：**

合规落地方案分三块。第一块密钥自管：用 Cloud KMS 建 CMEK——`gcloud kms keyrings create app-ring --location=global`，`gcloud kms keys create app-key --location=global --keyring=app-ring --purpose=encryption --rotation-period=90d`，然后对资源启用：磁盘创建时 `--kms-key=projects/xxx/locations/global/keyRings/app-ring/cryptoKeys/app-key`，GCS 桶用 `--default-kms-key`（注意桶一旦设默认 key，新对象用它，存量对象要重写才生效），BigQuery 表/数据集设默认 CMEK（`gcloud bq` 或 SQL DDL）。这里的关键认知：CMEK 是"数据密钥由 KMS 托管、你控制 key 的权限与生命周期"，CSEK 是"key 完全由你持有、每次请求自带"，后者更重且对运维要求高，新系统一律 CMEK。KMS key 权限要独立收敛：只有 `roles/cloudkms.admin` 能管 key，加密解密权限给数据服务 SA；销毁 key 有软删除窗口（默认 24h 可撤销，可延长），必须配审批流程——删 key = 永久删数据，这是最高危操作。第二块安全监控：Security Command Center（SCC）开启免费层后会自动发现：IAM 过度授权（如绑了 Owner 的人、`allUsers` 暴露的桶）、VM/GKE 的 CVE 漏洞、公网暴露资产；我会把它的事件与 Slack/工单系统打通，高危事件（如新出现公网 22 端口）30 分钟内响应。第三块审计防篡改：Admin Activity 日志默认开（400 天保留），Data Access 日志（谁读了谁写了数据）需要显式开启且默认只 90 天，合规要求"至少 1 年"时必须用 sink 导出——`gcloud logging sinks create audit-sink bigquery.googleapis.com/projects/audit-proj/datasets/audit_logs --log-filter='logName:"cloudaudit.googleapis.com"'`，导出目标放在独立 Project（`audit-proj`），该 Project 只有审计团队有权限，且 BigQuery dataset 禁止业务方写入；更强的是把日志导到开启 Bucket Lock（retention lock，不可撤销）的 GCS 桶，实现"任何人（包括管理员）都不能提前删除"的 WORM 语义。实战坑：一是 Data Access 日志量大（尤其 BigQuery 每查询都记），费用爆炸，要按 `--log-filter` 只导敏感资源；二是 GCS 的 `--default-kms-key` 对已存在对象不生效，迁移存量要跑批量重写；三是 SCC 事件告警配了但没人接（响应流程缺失），等于白扫。

**延伸考点：** 如果 CMEK 的 KMS key 被意外删除且已过软删除窗口，数据恢复手段是什么？（提示：没有）你怎么设计防呆流程？

---

### Q11. 线上事故频发却定位不到根因，可观测性四件套怎么搭？

**问题：** 服务偶发 5xx、P99 抖动，日志查不到、指标看不到、调用链断。你会用 Cloud Monitoring、Cloud Logging、Cloud Trace、Error Reporting 怎么组合成一套可观测性体系？告警怎么设计不产生告警疲劳？

**期望加分项：**
- 讲清四件套分工：Monitoring 采集指标（CPU/内存/自定义 metric + uptime check）、Logging 聚合日志（Cloud Logging agent 或 Ops Agent 采集 VM 日志，Kubernetes 日志自动）、Trace 采集分布式调用链（`cloudtrace` exporter，opentelemetry 接入）、Error Reporting 聚合异常堆栈自动去重告警
- 有实战整合：指标报警 + 日志关联 + Trace 透传（trace id 写入日志字段，点击日志直接跳到 Trace），实现"指标异常 → 看日志 → 看链路"的排查路径；自定义 metric 用 `gcloud beta monitoring` 或 OpenTelemetry 写 `logging.googleapis.com/user/xxx` 指标
- 告警设计有方法论：SLO 驱动的告警（用 burn rate 而非裸阈值）、多级告警（warning/critical 分开）、告警分组与抑制（同一资源多次触发合并）、值班轮换（`notification-channel` 配 PagerDuty/Slack）
- 有量化：uptime check 全球 4-5 个点检测可用性；日志用 Log-based metric（`metricDescriptor`）把"错误率"当指标告警；Trace 采样（head-based 10% 或 tail-based）控制成本
- 有真实案例：如"P99 抖动定位到是某 Pod 的 GC 频繁，Trace 里能看到服务间耗时占比"，或"5xx 偶发是健康检查摘除延迟 + 连接池耗尽"这类排障故事
- 提到边界与成本：日志量（GB/月）与费用、Trace 采样率对成本的影响，日志 retention（默认 30 天）与导出

**减分项：**
- 四件套各自是什么都说不全，或只有"看监控"没有组合逻辑
- 告警只写"CPU>80%"这种裸阈值，没有 SLO/burn rate/多级/去重概念
- 没有 Trace 的接入实践，说不出 trace id 与日志的关联方式
- 不知道如何用日志生成指标（log-based metric）做告警
- 没有成本意识：不采样、全量日志、无限告警

**解答：**

我的体系是"一个入口 + 三条路径"：一个入口是 Cloud Monitoring 的 Dashboard（或 Grafana 接 GCP 数据源），三条路径是"指标→日志→链路"联动。落地四件套：①监控指标：VM/GKE 装 Ops Agent（替代老 stackdriver agent，采集 CPU/内存/磁盘/进程），业务侧用 OpenTelemetry 上报自定义 metric（如订单量、排队长度），Cloud Monitoring 建 dashboard；②日志：K8s 日志默认进 Logging，业务日志必须结构化（JSON 字段含 `trace_id`、`service`、`level`），Logging 里建 log-based metric（`logging.googleapis.com/user/err_rate`：匹配 `jsonPayload.level="ERROR"` 的计数除以请求数），它天然成为"错误率指标"；③链路：服务接 OpenTelemetry + Cloud Trace exporter，header 透传 `x-cloud-trace-context`，日志里塞 trace id 后，在 Logging 点击即可跳 Trace 视图；④异常：Error Reporting 自动聚合同源堆栈，重复错误只报一次。告警设计我按 SLO 驱动：核心 API 定 SLO 99.9%（月不可用 <43 分钟），用 burn rate 告警——1 小时窗口错误率 >5%（约 10x burn rate）触发 critical、24 小时窗口 >0.5% 触发 warning，比"CPU>80%"这种症状告警更能反映用户影响；每条告警配 notification channel（Slack + 电话分级），warning 只发 Slack、critical 才 pager，同资源同窗口自动去重合并。真实案例：曾有一次 P99 从 50ms 涨到 800ms，指标看起来一切正常（CPU 30%），最后是 Trace 里发现 A 服务调用 B 的耗时占比从 20% 涨到 85%，点进 B 的 span 看到它等 Redis 锁等待——顺着"指标异常→日志看错误→Trace 看耗时占比"一路点下来 20 分钟定位。实践中的坑：一是日志没带 trace id，异常时日志和链路对不上，只能靠时间猜；二是全量 trace 采样费用高，head-based 10% 足够定位绝大多数问题（低频问题用 tail-based 补）；三是告警阈值拍脑袋导致凌晨全是假警，值班人员麻木后真警也漏——所以一定要从 SLO/burn rate 反推阈值并每周 review 告警命中率，砍掉命中率为 0 的规则。

**延伸考点：** Trace 的采样策略（head-based vs tail-based）在"低频慢请求"定位上各自的适用性是什么？

---

### Q12. 用 Terraform 管 GCP 资源，状态、权限、模块怎么组织？和 Deployment Manager 比呢？

**问题：** 平台团队要用 IaC 管理 GCP：几十个 Project 的 VPC、GKE、IAM 都要代码化。Terraform 在 GCP 上的 provider 使用、远程状态（backend）、权限模型（service account）怎么设计？它和 Google 自家的 Deployment Manager / Config Controller 是什么关系？

**期望加分项：**
- 讲清 provider 使用：`google` provider 的 `project` 参数、按 Project 拆 state（`terraform init -backend-config` 或 `-backend=gs://tf-state/xxx.tfstate`），credentials 用 impersonation（`impersonate_service_account`）而不是静态 key
- 有状态与锁设计：状态放 GCS bucket（开启版本控制 + `--uniform-bucket-level-access`），`terraform init -backend-config="bucket=tf-state"`，锁由 GCS object 实现；按环境/项目分 state 文件（`tfstate/{env}/{project}.tfstate`），避免大 state 与锁竞争；关键资产（如 IAM、VPC）状态要严格 review
- 讲清权限模型：Terraform 用最小权限 SA（如 `terraform-sa@...` 只授 `roles/editor` 或按模块细分），通过 Workload Identity Federation 让 CI 以该 SA 身份执行（无 key 落地）；`impersonate` 代替下载 key
- 有模块与代码组织：按"landing zone"拆模块（folder/project/vpc/gke/iam），用 `terraform-google-modules` 官方模块（如 `terraform-google-project-factory`、`terraform-google-kubernetes-engine`），版本钉死、用 `terraform validate`/`plan` 在 CI 门禁
- 说清与 Google 原生工具的关系：Deployment Manager 是 Google 的声明式 IaC（Python/Jinja 模板、`gcloud deployment-manager`），生态弱、社区少，Google 现在推荐 Terraform（`gcloud` 生成的 Terraform 样板）；Config Controller（基于 KRM + Config Sync）做"集群内/组织内持续合规配置"（GitOps 式），与 Terraform 互补：Terraform 管"创建资源"，Config Controller 管"运行期配置漂移收敛"
- 有实践坑：provider 版本与 API 版本不对齐、`terraform plan` 与权限不全导致 plan 失败、state 锁过期（`-lock=false` 滥用）、import 存量资源（`terraform import`）的处理

**减分项：**
- 只会 `terraform apply`，说不出 backend/state 锁/按项目分状态这些工程化设计
- credentials 还是"放一个 owner key 在 CI"，没有最小权限与 impersonation 概念
- 不知道 Google 官方 terraform-google-modules 与 provider 的区别
- 分不清 Terraform 与 Deployment Manager / Config Controller 的定位（创建 vs 运行期合规）
- 没有 drift/import/state 管理经验，答不出存量迁移方案

**解答：**

IaC 落地我按"权限 → 状态 → 模块"三件事设计。权限：建一个专用 SA `tf-runner`，在 Organization 层给 `roles/resourcemanager.projectCreator`，在各 Folder 给对应 `roles/editor`（或更细的 `roles/compute.admin` 等），CI（GitHub Actions / Cloud Build）通过 Workload Identity Federation 以该 SA 身份执行——绝不在 CI 里放 JSON key；本地调试用 `gcloud auth application-default login --impersonate-service-account=tf-runner@...`。状态：全部放一个 GCS bucket（如 `gs://company-tf-state`，开版本控制防误删、开 `--uniform-bucket-level-access`），每个 Project/模块一个 state 文件：`gs://company-tf-state/{env}/{module}.tfstate`，backend 配置 `terraform { backend "gcs" { bucket="..." prefix="prod/vpc" } }`，锁由 GCS 对象自动实现——这样多人并发 apply 不会互相踩。代码组织：用官方 `terraform-google-modules` 打底（project-factory 建 Project、`terraform-google-kubernetes-engine` 建 GKE、`terraform-google-network` 建 VPC/子网/防火墙/NAT），内部再包一层业务模块；CI 里 `terraform fmt -check`、`terraform validate`、`terraform plan` 必须过人工 review 才允许 apply（配 `-detailed-exitcode` 让 plan 有 diff 时 fail）。与 Google 原生工具的关系要讲清：Deployment Manager 是 Google 自己的声明式 IaC，但模板生态弱（Python/Jinja）、文档少、Google 官方已转向推荐 Terraform，新项目我直接用 Terraform，只在存量 DM 服务上做 `terraform import` 迁移；Config Controller 则不同定位——它不是"创建资源"的工具，而是基于 GitOps（Config Sync + KRM）把"集群/组织的期望配置"持续收敛，比如强制所有 namespace 必须带 label、所有 Pod 必须声明 resources，它是"运行期合规兜底"，和 Terraform 是互补关系而非替代。实践坑：一是 provider 升级（如 4.x→5.x）可能推倒重来，升级前必须 plan + 在 dev 环境验证；二是 `terraform plan` 时 SA 权限不足会静默跳过某些资源检查，导致 apply 时才报错，所以要给 plan 也配读权限；三是刚接手时存量资源没进 state，用 `terraform import` 逐个导入，导入后跑 `plan` 看 diff 是否符合预期再 apply，别上来就 apply。

**延伸考点：** Terraform 在 GCP 上管理 IAM 时，`google_project_iam_member` 与 `google_project_iam_binding` 的语义差异会导致什么样的"权限被覆盖"事故？

---

### Q13. CI/CD：Cloud Build、Cloud Deploy、Artifact Registry 和 GitHub Actions 怎么分工？

**问题：** 团队代码在 GitHub，要构建 Docker 镜像并部署到 GKE（分 staging/prod），还要支持金丝雀发布。Cloud Build、Cloud Deploy、Artifact Registry 各自扮演什么角色？和 GitHub Actions 怎么协作？

**期望加分项：**
- 讲清四者定位：GitHub Actions 是触发与编排层（监听 push/PR，跑 lint/单元测试）；Cloud Build 是构建层（`cloudbuild.yaml` 的 build/test 步骤，生成镜像 + 单元测试 + 上传镜像到 Artifact Registry）；Artifact Registry 是制品仓库（镜像 + Maven/npm 包，`gcr.io` 迁移到 `us-xxx.pkg.dev`）；Cloud Deploy 是发布编排层（delivery pipeline：staging→prod 推进、金丝雀/滚动发布、自动回滚）
- 有完整流水线示例：GitHub Actions 里 `google-github-actions/auth@v2`（Workload Identity Federation 换取 token）→ `setup-gcloud` → 触发 `gcloud builds submit`（或直接调 cloudbuild API）；`cloudbuild.yaml` 用 kaniko/docker build + `docker push`，`images: ['$LOCATION-docker.pkg.dev/$PROJECT_ID/repo/app:${_COMMIT_SHA}']` 参数化
- 讲清 Cloud Deploy：`gcloud deploy apply --file=pipeline.yaml --region=...`（定义 staging→prod rollout 顺序），`gcloud deploy releases create` 触发，支持 `--canary`（金丝雀百分比、`--delivery-pipeline` 策略）、自动回滚（`--rollout-id` 失败回滚）、与 GKE manifest（kustomize/helm）结合
- 有安全视角：镜像签名（Artifact Registry 的 vulnerability scanning + 在 Cloud Deploy 里用 `requireTLS` 与 attestation 策略门禁）、凭据不落库（WIF 代替 SA key）、按环境分 Artifact Registry repository 与 IAM
- 有实践坑：Cloud Build 的镜像推送到同一仓库时 IAM 角色（`roles/artifactregistry.writer`）缺失、`${COMMIT_SHA}` 与 `$_CUSTOM` 的替换规则、GKE 集群在共享 VPC 时 Cloud Build 连不上（网络打通）、金丝雀发布中 ingress 权重 vs GKE 内 ClusterIP 的策略选择
- 有成本/速度视角：Cloud Build 的 machine type（e2-medium 默认 vs e2-highcpu-8 加速构建）、缓存（`--cache-from`）对构建时长的影响

**减分项：**
- 四者职责混为一谈（把发布部署全塞进 GitHub Actions）
- 不知道 WIF 免 key 认证，还在用 SA JSON key
- 金丝雀发布说不出机制（Cloud Deploy canary 的 rollout 阶段），或只会"先发布再看"
- 不知道 Artifact Registry 已取代 Container Registry（gcr.io），或说不出仓库按环境/类型分
- 没有镜像漏洞扫描/签名门禁意识

**解答：**

分工原则是"GHA 做触发与质量门禁，Cloud Build 做构建，Cloud Deploy 做发布，Artifact Registry 做制品仓库"。GitHub Actions 工作流：`push` 到 main 触发 → `google-github-actions/auth`（用 Workflow Identity Federation，CI 以 SA `gh-deploy@proj` 身份执行，无需 key）→ 跑单测/Lint → `gcloud builds submit --config=cloudbuild.yaml --substitutions=_COMMIT_SHA=${GITHUB_SHA}`。cloudbuild.yaml 里做两件事：用 docker build 出镜像（`--cache-from` 复用层缓存，构建时间从 5 分钟压到 90 秒）、推送到 Artifact Registry（`$REGION-docker.pkg.dev/$PROJECT/repo/app:${_COMMIT_SHA}`，开漏洞扫描，高危 CVE 阻断 push 或阻断 deploy）。发布用 Cloud Deploy：定义 delivery pipeline `staging → prod`（`gcloud deploy apply --file=pipeline.yaml`），GHA 里调 `gcloud deploy releases create rel-${GITHUB_SHA} --delivery-pipeline=app-pipe --to-target=staging`；prod 发布走金丝雀：rollout 先发 5% 流量（`--canary` + 时间窗），Cloud Deploy 自动用 GKE Service 的 `spec.traffic` 权重切流（金丝雀发布由 rollout 的 `canaryDeployment` 阶段管理），观测 Cloud Monitoring 的错误率 SLO 通过才自动推进 100%，失败则 `--rollback` 自动回滚上一个 rollout。为什么不让 GHA 直接 `kubectl apply`：发布要有"可审计、可暂停、可回滚"的发布对象和策略编排，Cloud Deploy 把 staging→prod 的推进、金丝雀、回滚做成了声明式资源，且天然有 IAM 门禁（谁可以 approve prod 发布）。实践中的坑：一是 Cloud Build 默认在 `us-central1` 或项目默认网络跑，GKE 在共享 VPC 里时构建容器连不上私有集群，要么构建也放同网络（`--vpc-config` 指定子网），要么用 Artifact Registry + Cloud Deploy 走 manifest 下发而非直连集群；二是镜像 tag 用 `latest` 导致无法回滚到确切版本，必须用 SHA 打 tag、`latest` 只作为便捷别名；三是 Artifact Registry 的 vulnerability scanning 默认只扫"推送时"的镜像，要配按天级联扫 + deploy 门禁（`gate`）把"修复中的高危漏洞"拦在 prod 外；四是忘了给 Cloud Build 的 SA 配 `roles/artifactregistry.writer` 和 `roles/clouddeploy.operator`，流水线在最后一步才报权限错，先做权限矩阵检查再上流水线。

**延伸考点：** 金丝雀发布用 Cloud Deploy 的 rollout