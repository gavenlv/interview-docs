# DevOps · Kubernetes（面试题库）

本文件考察候选人在 Kubernetes 工程落地上的真实能力，覆盖工作负载管理、网络与存储抽象、调度与弹性伸缩、集群排障、权限安全以及存量应用迁移等核心主题。所有题目均为线上可复现的场景化问题，要求候选人给出量化依据、取舍说明与真实排障路径，而非背诵 API 概念。难度自 Q1 至 Q20 渐进，从实践基础逐步过渡到架构级开放性思考题。

---

### Q1. Pod 反复重启且服务"假活"，怎么从生命周期角度定位？

**问题：** 发布后一批 Pod 处于 CrashLoopBackOff，另一批状态是 Running 但业务持续报错、日志里却什么都没有。你如何从 Pod 生命周期视角定位这两类问题？

**期望加分项：**
- 能讲清完整生命周期：Pending → ContainerCreating → Running，以及 Init 容器、PostStart/PreStop 钩子、Terminating 阶段
- 能准确解读退出码：0（正常退出）、1（应用错误）、137（SIGKILL，多因 OOM）、143（SIGTERM，优雅终止超时被杀）
- 能讲清 restartPolicy 三值（Always/OnFailure/Never）对重启行为的影响，并说明 BackOff 退避机制（10s 起翻倍，封顶 300s）
- 对"Running 但不可用"能想到检查容器内进程是否存活、端口监听、readiness 未就绪、探针路径问题
- 提到 terminationGracePeriodSeconds 与优雅停机、僵尸进程与 PID 1 处理

**减分项：**
- 只背 Pending/Running/Succeeded/Failed 四种状态，讲不出状态机细节
- 分不清 137 与 143 的语义差异（被杀 vs 主动终止）
- 不知道 CrashLoopBackOff 的退避逻辑，只会"重启 Pod"
- 不检查容器内实际进程，凭 Pod 状态下结论
- 忽略 Init 容器失败导致主容器永远无法启动的情况

**解答：**

先按"状态 → 原因 → 证据"三层走。第一层看状态：`kubectl get pod -o wide` 列出所有 Pod，对 CrashLoopBackOff 用 `kubectl describe pod <name>` 看 Events 与容器状态里的 `Restart Count` 和退出码。退出码是核心线索：137 通常是被 cgroup OOM 杀掉（看节点 `dmesg | grep -i oom` 或容器 status 里 `OOMKilled: true`）；1 是应用自身报错，此时必须 `kubectl logs <pod> --previous` 看上一次崩溃的日志，而不是只看当前空日志。注意 restartPolicy 默认 Always，意味着任何退出（包括成功）都会被拉起重启，所以"正常退出 0 也重启"是符合预期的。第二层看"Running 但假活"：Pod 状态 Running 只代表容器被拉起、主进程没退出，不代表业务可用。要用 `kubectl exec` 进容器检查进程、监听端口（ss -lntp）、尝试本地 curl 健康接口；若 Pod 显示 NotReady，查 readiness 探针，很多"没日志"其实是探针从未通过、流量根本没进来。第三层关注终止：优雅停机依赖 PreStop 钩子 + terminationGracePeriodSeconds，若应用没接 SIGTERM，到点会被 SIGKILL（退出码 137），在线请求被掐断。实践中的坑：容器内 PID 1 若被脚本替代，SIGTERM 无人处理，会稳定演变成"每次发布都 137"。

**延伸考点：** 容器退出码 137 怎么区分是节点 OOM 还是容器 limit OOM？Init 容器卡住 Pending 时 describe 里能看到什么？

---

### Q2. Deployment 发布后要回滚，怎么又快又稳？

**问题：** 你把镜像从 v1 升到 v2，5 分钟后错误率上升，决定回滚。回滚怎么做？回滚过程会不会中断正在处理的请求？平时滚动策略该怎么配才稳？

**期望加分项：**
- 能说出 `kubectl rollout undo deployment/<name>`、`kubectl rollout status -w`、`kubectl rollout history` 完整命令链
- 能讲清 maxSurge/maxUnavailable 的含义（百分比或整数）与典型配比，如 maxSurge: 25% / maxUnavailable: 0
- 能点破"回滚本质是再触发一次滚动更新"，同样有扩缩容窗口，不是瞬间切回
- 提到 readiness 门禁：新 Pod readiness 不通过时滚动会卡住（progressDeadlineSeconds 超时后标记为 failed）
- 有版本留痕意识：revisionHistoryLimit、修改 trigger（镜像/注解）产生新 revision、--record 记录变更原因

**减分项：**
- 只会说"重新发一遍旧镜像"，不知道 rollout undo 与 revision 机制
- 配出 maxSurge: 0 且 maxUnavailable: 0 的非法组合（两者不能同时为 0）
- 忽略回滚也会经历新旧共存窗口，未评估对在线连接的影响
- 不设 progressDeadlineSeconds，发布卡住时毫无感知
- 用 kubectl apply 直接覆盖而不是走 rollout 机制，导致回滚无历史可用

**解答：**

回滚的正确姿势是 `kubectl rollout undo deployment/myapp`，它会根据历史 revision 重新生成 ReplicaSet 并走一次滚动更新；`kubectl rollout history deployment/myapp` 查看版本，`--to-revision=N` 回滚到指定版本，`kubectl rollout status deployment/myapp -w` 观察进度。关键认知：回滚不等于"瞬移"，它是把旧 ReplicaSet 的副本数抬起来、新 ReplicaSet 压下去的过程，期间新旧 Pod 共存，所以回滚前要评估存量连接——如果 v2 的 Pod 因崩溃被反复拉起，回滚前最好先 `kubectl scale deployment/myapp --replicas=0` 把坏实例摘干净，避免新流量继续打到坏 Pod。滚动策略建议：`maxUnavailable: 0`（保证每时每刻可用副本不减少）+ `maxSurge: 25%`（多放一批新 Pod 验证），配合 readiness 探针做门禁——新 Pod 探针不通过就不摘旧 Pod，流量不会切过去；再设 `progressDeadlineSeconds: 300`，超过 5 分钟没完成滚动自动标记失败，配合 CI 里 `rollout status` 失败即报警。坑：只改环境变量或 ConfigMap 也会触发新 revision，回滚镜像时这些改动会一起回到旧版本，配置与代码版本要对齐；发布脚本里不要用 `kubectl set image` 后又手动 apply 旧文件，会把 revision 历史搅乱。

**延伸考点：** 新版本 readiness 一直不通过，滚动会停在什么状态？如何强制终止本次滚动更新？

---

### Q3. 压测流量全打在一台 Pod 上，长连接还频繁断，Service 怎么了？

**问题：** 服务通过 Service 暴露，压测发现流量明显偏向其中一台 Pod；另一条线上业务是长连接，每次重启 Pod 后客户端连接大量报错。从 Service 与 kube-proxy 角度分析原因并给出解法。

**期望加分项：**
- 能讲清 ClusterIP/NodePort/LoadBalancer 三层暴露方式及各自适用场景
- 能讲清 kube-proxy 三种模式差异：userspace（用户态转发，已废弃）、iptables（随机 DNAT，无连接数感知）、IPVS（内核态，支持 rr/lc/wrr 等算法）
- 长连接 + iptables 模式是流量不均的典型成因：DNAT 目标是随机选的，新建连接数不均，长连接一旦建立就固定，偏斜更明显
- 能提到 conntrack 表满（nf_conntrack: table full）导致连接新建失败，以及长连接被重启打断的优雅处理
- 知道 endpoints 与 readiness 联动：Pod 未 ready 自动从 endpoints 摘除
- 知道 externalTrafficPolicy: Local 保源 IP 与 Cluster 的差异

**减分项：**
- 说不清 iptables 模式下 kube-proxy 的 DNAT 转发链路
- 不知道 readiness 失败会从 Service 后端摘除
- 压测不均只会猜"网络插件问题"，不会区分四层转发路径
- 长连接场景不考虑连接断开时的重连与优雅停止
- 混淆 LoadBalancer 与 NodePort（云厂商 LB 本质是转发到各节点 NodePort）

**解答：**

先给结论：连接不均的第一嫌疑是 kube-proxy 的转发模式。iptables 模式下，kube-proxy 为每个 Service 生成 KUBE-SVC 链，里面的规则用 `random` 模块做近似均匀的 DNAT，它不感知每台后端当前连接数，短连接多时大数定律勉强均匀；一旦是长连接（WebSocket/gRPC/推送），连接建一次用很久，偏斜会被放大——压测工具本身新建连接数少时，随机落到哪台就固定粘在哪台。解法：改用 IPVS 模式（`kube-proxy --proxy-mode=ipvs`，调度算法默认 rr），或对无状态服务用 Deployment 层面的平衡调度；真正要均匀必须让业务侧做客户端负载均衡（如 gRPC 的 pick_first 换成 round_robin）。长连接断连问题分两半：一是 Pod 重启时没有优雅收尾，客户端感知到连接中断——需要在容器加 preStop 钩子（sleep 几秒 + 通知注册中心下线）+ terminationGracePeriodSeconds 给足排空时间；二是节点 conntrack 表满导致新建连接全部失败（`dmesg` 里见 `nf_conntrack: table full`），调大 `net.netfilter.nf_conntrack_max` 或排查是否有大量 TIME_WAIT/畸形连接。排障命令：`kubectl get endpoints <svc>` 确认后端是否完整、`kubectl get svc -o yaml` 看 externalTrafficPolicy 与 sessionAffinity，必要时 `iptables -t nat -L KUBE-SVC-XXX -n -v` 看每个后端被命中的包计数，直接量化不均程度。

**延伸考点：** 把 Service 的 externalTrafficPolicy 改成 Local 后，为什么会出现部分节点不通？sessionAffinity: ClientIP 的粘性粒度能解决连接均分问题吗？

---

### Q4. 几十个服务共用一个域名做路由，Ingress 怎么设计？404 了怎么查？

**问题：** 你们有几十个服务对外暴露但域名有限，计划共用域名按路径路由到不同服务。Ingress 规则怎么写？Nginx Ingress、Traefik、云厂商 LB 怎么选？线上某条路径突然 404，你按什么顺序排查？

**期望加分项：**
- 能一句话点破"Ingress 是 API 资源（声明），Ingress Controller 是实现（监听并生成代理配置）"，缺 Controller 规则不生效
- 能写出 host + path 路由、路径类型（Prefix/Exact/ImplementationSpecific）、rewrite-target 注解的使用场景与坑（捕获组）
- 选型能给出量化对比：Nginx Ingress（最成熟、正则/自定义 annotation 强、内存数百 MB 起）、Traefik（动态发现、配置热更、轻量，适合中小集群）、云厂商 LB（托管免运维、按量计费、扩展能力受限）
- 404 排查有清晰路径：Controller 是否运行 → ingress 规则是否命中（annotation、path 写法）→ backend Service 是否存在 → endpoints 是否为空 → 后端 Pod 是否 ready
- 提到 TLS：证书放 Secret，cert-manager 自动签发

**减分项：**
- 把 Ingress 和 Ingress Controller 混为一谈
- 不知道 rewrite-target 机制，配了前缀路径后转发时路径没被改写导致 404/302 跳转异常
- 只背"Ingress 是七层负载均衡"，写不出一条实际规则
- 排障无顺序，上来就重启 Controller
- 忽略 path 类型语义差异（nginx 的 `nginx.ingress.kubernetes.io/rewrite-target` 与 `pathType` 的配合）

**解答：**

设计上先定路由模型：域名按业务分（每个域名一组 host），路径按服务分。典型规则：`host: api.example.com`，path `/user(/|$)(.*)` → `serviceName: user-svc`，`pathType: Prefix`；因为后端路由往往不含前缀，需要 `nginx.ingress.kubernetes.io/rewrite-target: /$2` 用捕获组把前缀剥掉——这是 404 高发区：只写了 path 没配 rewrite，后端收到 `/user/...` 匹配不到自己的路由就 404。选型按集群规模与诉求：几十个服务、对正则和细粒度控制有要求选 Nginx Ingress（社区最大、资料多，线上 99% 场景够用）；团队想省心、配置即服务发现可选 Traefik；纯云上且不想维护 Controller 用云厂商托管 LB Ingress（如 ALB），但要注意它不支持复杂 rewrite 和灰度注解，和 K8s 原生行为有差异。404 排查顺序：① `kubectl get ingress -A` 确认规则存在且 `ADDRESS` 已分配；② `kubectl get pod -n ingress-nginx` 确认 Controller 存活；③ 看 `kubectl -n ingress-nginx logs <controller-pod> --tail=50` 里请求命中的 host/path 与服务名；④ 查后端 `kubectl get svc` 与 `kubectl get endpoints` 是否为空——Ingress 404 最隐蔽的根因是后端 Pod 全部未 ready，endpoints 为空，请求打到默认 404。坑：controller 日志默认级别看不到 rewrite 细节，调 `-v=2` 观察 upstream；annotation 拼写错误不报错，只是静默不生效。

**延伸考点：** rewrite-target 里的捕获组（$1/$2）与 path 里的括号组怎么对齐？灰度（canary）Ingress 注解怎么配、权重怎么设？

---

### Q5. 改完 ConfigMap 服务不生效，Secret 明文躺在 etcd，怎么办？

**问题：** 你改了 ConfigMap 里的某个配置，业务说没生效，要重启 Pod 才生效；另一边安全审计指出 Secret 在 etcd 里是明文。这两个问题分别怎么解决？

**期望加分项：**
- 能讲清两种注入方式的本质差异：环境变量只在容器启动时注入一次，文件挂载（volumeMounts）由 kubelet 周期同步（默认约 1 分钟）可以热更新，但应用要自己监听文件变化
- 能指出 subPath 挂载不支持热更新的坑，以及热更新不生效时"重启触发重新挂载"的兜底做法
- 能讲清 Secret 默认只是 base64 编码不是加密，静态加密要开 EncryptionConfiguration（KMS/aescbc），或外部化到 Vault/Sealed Secrets
- 有配置版本管理意识：immutable 字段、用 checksum annotation 让 ConfigMap 变更触发滚动更新
- 能主动谈配置回滚：配置与镜像版本对齐，回滚时配置要一起回

**减分项：**
- 认为改 ConfigMap 后环境变量注入方式也会自动生效
- 不知道 subPath 挂载不支持热更新
- 认为 base64 就是加密，secret 放 etcd 无所谓
- 热更新失效只会"重启所有 Pod"，不会用 rollout restart
- 不考虑配置与代码版本对齐，回滚镜像时配置新旧混杂

**解答：**

两个问题本质都是"配置生命周期的管理"问题。第一个：环境变量注入发生在容器创建时，之后改 ConfigMap 对已运行容器无效，想生效必须让 Pod 重建——正确做法是 `kubectl rollout restart deployment/<name>`，或更工程化地在 Deployment template 里加 `checksum/config: ${sha256sum(configmap)}` 注解，ConfigMap 一变镜像配置哈希就变，自动触发滚动更新，还能在 GitOps 里留痕。文件挂载方式：kubelet 每 ~1 分钟把 ConfigMap 同步到挂载目录，应用监听文件变化即可热更新，但注意：subPath 单文件挂载是启动时一次性快照、不支持热更新；热更新有秒级延迟且应用不监听也没用，所以线上通常把"重载类配置"（日志级别、开关）走挂载热更，"核心配置"走 rollout restart 求稳。第二个：etcd 里的 Secret 默认是 base64 明文，任何能读 etcd 备份的人都能还原。最低成本方案是开 kube-apiserver 的 `--encryption-provider-config`（aescbc 或 kms），对 secrets 资源做静态加密；更彻底的是密钥不进 etcd，用外部系统注入：Sealed Secrets（公钥加密、私钥在集群内解密）、Vault Agent 边车、云厂商 KMS。实践中的坑：开启静态加密只对之后写入的 Secret 生效，存量要先重写一遍（kubectl get secret -A -o yaml 再 apply）；加密配置升级（轮换密钥）要按官方流程多阶段操作，搞错直接所有 Secret 不可读。配置管理的坑：回滚镜像时 ConfigMap 不会自动跟着回，推荐把配置和发布模板放同一仓库同一 commit 管理，回滚时一起回。

**延伸考点：** 静态加密的密钥轮换流程怎么保证不中断业务？用 Vault 注入密钥时 Pod 启动顺序和注入失败怎么处理？

---

### Q6. MySQL 要跑进 K8s，存储怎么设计？PVC 一直 Pending 怎么查？

**问题：** 你们计划把 MySQL 迁进 K8s（不用 Operator），存储方案怎么设计？开发环境只有 NFS、生产要求云盘，怎么统一抽象？一个 PVC 申请后一直 Pending，你按什么顺序排查？

**期望加分项：**
- 能讲清 PV/PVC 的绑定机制与两种供给方式：静态供给（手工建 PV）与动态供给（StorageClass + provisioner 自动创建）
- 能写出 StorageClass 关键字段：provisioner、parameters（如云盘类型/IOPS）、reclaimPolicy、volumeBindingMode
- 能对比存储类型：云盘（SSD/ESSD，网络块设备、延迟毫秒级）vs 本地卷（Local PV，性能最好但节点漂移丢失）vs NFS（共享、性能差，不建议跑数据库）
- 数据安全有完整方案：reclaimPolicy Delete 默认会连数据一起删，数据库用 Retain；备份（Velero/快照）必须有
- Pending 排查路径清晰：storageClassName 是否匹配 → provisioner 是否部署（external-provisioner）→ 容量是否超售 → volumeBindingMode: WaitForFirstConsumer 未调度

**减分项：**
- 分不清静态供给与动态供给，PV 手工建了一堆没人绑
- 不知道 reclaimPolicy 对数据销毁的影响（Delete 直接删 PV 与数据）
- 只说"用云盘"，说不出 CSI 驱动、容量、IOPS、延迟要求
- 把 NFS 当万能存储，数据库也往上放
- 不会看 `kubectl get pvc`/`kubectl describe pvc` 的事件定位 Pending 根因

**解答：**

先定原则：数据库这类有状态高 IO 服务，首选"每实例一块云盘/本地 SSD"而非共享存储；开发环境想便宜才考虑 NFS。实现上推荐动态供给：写一个 StorageClass（`provisioner: disk.csi.aliyun.com` 之类，parameters 里指定云盘类型与性能等级，`reclaimPolicy: Retain`，`volumeBindingMode: WaitForFirstConsumer`），然后 PVC 只声明 `storageClassName` 和容量，provisioner 自动建 PV 并挂给 Pod。`WaitForFirstConsumer` 延迟绑定很关键：普通模式 PVC 一创建就绑任意可用 PV，可能绑到与 Pod 不同可用区的盘，延迟绑定等 Pod 调度定节点后再选盘，保证同可用区，本地卷（Local PV）必须用它。注意数据库场景 reclaimPolicy 用 Retain——Delete 模式里 PVC 删除时 PV 连同数据一起销毁，运维误删 PVC = 删库；Retain 则保留 PV 等人工处理（还可以恢复）。PVC Pending 排查顺序：① `kubectl get pvc -o wide` 看 STATUS 与绑定的 PV；② `kubectl describe pvc` 读 Events，常见三类：`storageclass not found`（名字拼错或没装对应 provisioner）、`WaitForFirstConsumer` 未调度（说明 Pod 还没起或调度失败）、容量不足/配额限制；③ 查 provisioner 侧：`kubectl get pods -n kube-system | grep csi` 确认 CSI controller 存活，云厂商控制台看磁盘是否创建、是否扣费。实践中的坑：测试环境随手建的 StorageClass 用 Delete + 普通绑定，生产直接复制导致数据裸奔；Local PV 的节点绑定问题、节点下线后数据丢失都要提前演练。

**延伸考点：** volumeBindingMode 从 Immediate 改成 WaitForFirstConsumer 解决了什么问题？磁盘扩容后文件系统在线扩容（CSI 扩容）怎么做？

---

### Q7. GPU 训练任务要落对节点，数据库节点要防混部，怎么用调度器实现？

**问题：** 集群里有 GPU 节点、普通计算节点和数据库专用节点。训练任务如何保证落到 GPU 节点？数据库 Pod 如何避免被业务 Pod 挤占节点资源？一个 Pod 一直 Pending，你用调度器视角怎么定位？

**期望加分项：**
- 能讲清调度两阶段：Filtering（过滤，硬约束）→ Scoring（打分，软偏好），以及 nodeSelector/nodeAffinity 的差异（nodeSelector 是简单等值匹配，nodeAffinity 支持 In/NotIn/Exists 与软硬之分）
- 能讲清污点（Taint）与容忍（Toleration）：NoSchedule/NoExecute/PreferNoSchedule 语义，NoExecute 会驱逐存量 Pod
- 数据库隔离方案：节点打标签 + 污点 NoSchedule + 数据库 Pod 配 Toleration，业务 Pod 不带容忍自然进不来
- Pending 定位用 `kubectl describe pod` 看 Events 里的 FailedScheduling（含 scheduler 的过滤理由）
- 进阶：PodTopologySpread 打散副本、descheduler 治理碎片、自定义调度器/调度框架扩展点

**减分项：**
- nodeSelector 和 nodeAffinity 的区别说不清（等值 vs 集合、硬 vs 软）
- 只会"打标签"不会用污点，数据库节点照样被混部
- Pending 只会猜"资源不够"，不看 Events 里的具体调度失败原因
- 不知道 NoExecute 会驱逐存量 Pod（加污点瞬间业务大面积重启）
- 说不清 request 与调度约束的关系（调度只看 request 不看 limit）

**解答：**

调度器先 Filter 再 Score：硬约束不满足直接丢弃（Events 里出现 `FailedScheduling` 且注明原因），软偏好只在打分阶段体现。需求一"训练任务落 GPU 节点"：给 GPU 节点打标签 `gpu=true` + 资源字段声明 `nvidia.com/gpu: 1`，训练 Pod 用 nodeAffinity `requiredDuringSchedulingIgnoredDuringExecution` 强制过滤，配合 device plugin 确保资源可分配；若 GPU 稀缺且任务多，再用 `preferredDuringScheduling` 软偏好 + 打分权重平衡。需求二"数据库节点防混部"：这是污点容忍的典型场景——数据库节点加 `kubectl taint nodes db-node dedicated=db:NoSchedule`，数据库 Pod 配 `tolerations: [{key: dedicated, operator: Equal, value: db, effect: NoSchedule}]`，业务 Pod 无容忍自然调度不上去；想更狠就 `NoExecute`，但注意它会驱逐节点上存量 Pod，上线前评估。Pending 定位：`kubectl describe pod <name>` 底部 Events 是关键，`FailedScheduling` 会给出多条过滤原因（`0/5 nodes are available: 1 node(s) didn't match node selector, 3 Insufficient cpu`）；再用 `kubectl get nodes -o wide` 看资源水位、`kubectl get node <n> -o yaml` 查 taints 与 labels 是否匹配。实践中的坑：只配了节点标签没配污点，其他业务照样被"软性"调度上去，所以"专属节点 = 标签 + 污点 + 容忍 + 必要时节点池隔离"四件套；调度只看 request，limit 设再大也参与不了调度决策，这是新手最容易理解错的地方。

**延伸考点：** NoExecute 污点带 `tolerationSeconds` 和不带有什么区别？节点有 `node.kubernetes.io/not-ready` 污点时，Pod 默认容忍时间是多少、超时后会发生什么？

---

### Q8. Pod 天天被 OOMKilled，节点还被打爆，request/limit 到底怎么设？

**问题：** 线上 Java 服务经常 OOMKilled，另一个服务的 Pod 把所在节点 CPU 打满导致节点不稳定。request 和 limit 分别起什么作用？K8s 决定先杀哪个 Pod 的规则是什么？你给出数值设置的方法论。

**期望加分项：**
- 能讲清语义：request 用于调度（承诺资源）、limit 用于运行时约束（CPU 可压缩限流、内存不可压缩超额即杀）
- 能讲清 QoS 三级（Guaranteed/Burstable/BestEffort）判定规则与被杀优先级：OOM 时优先杀 BestEffort，其次 Burstable，Guaranteed 最后
- 知道节点驱逐（eviction，按 kubelet 阈值）与容器 OOMKilled 是两套机制
- 数值设置方法论：压测取 P99 水位做 request（典型值）、峰值做 limit，不拍脑袋；解释容器内 `limits.cpu` 会让 /proc/cpuinfo 变小（cgroup 视角）
- Java 特殊坑：JVM 不感知 cgroup（老版本）导致按宿主机核数算堆，超过容器 limit 被 OOMKilled；堆外内存（元空间/线程栈）超出 limit

**减分项：**
- 把 request 当限制用，limit 当预留用
- 不知道 QoS 分级与 OOM 分数（oom_score_adj）的关系
- 只会背"CPU 可压缩、内存不可压缩"，答不出工程后果
- limit 凭感觉填，无压测依据
- 混淆节点驱逐与容器 OOMKilled

**解答：**

先定语义：request 是调度承诺——kube-scheduler 按 request 汇总算节点剩余容量，Pod 运行时可以用到超过 request 但不超过 limit；limit 是运行约束——CPU 超限被 cgroup 限流（进程变慢但不杀），内存超限触发 cgroup OOM 杀进程（`OOMKilled: true`）。QoS 分级由 request/limit 组合决定：两者都设且相等 → Guaranteed；只设 request 或二者不等 → Burstable；都没设 → BestEffort。节点内存压力时 kubelet 按 QoS 排序驱逐：BestEffort → Burstable → Guaranteed，容器 OOM 也遵循类似优先序，这就是"谁的 Pod 先被杀"的答案。数值方法论：拿压测或监控的真实水位——request 取 P99 典型值（例如 4C8G 实例 CPU 常驻 1.5 核、峰 3 核，就 request 2C、limit 4C，内存 request 4G、limit 6G），保证"调度容量"和"实际占用"贴近，避免集群 50% 节点 request 满载实际只用 20%；limit 与 request 差值就是超卖空间，无状态服务可超卖、数据库不可超卖。OOMKilled 排查：`kubectl describe pod` 看 `Last State: Terminated, Reason: OOMKilled`，再看节点 `dmesg | grep -i "out of memory"` 里的进程内存账本，判断是 limit 太小还是 JVM 堆外内存超限。坑：JDK8 默认不读 cgroup，按宿主机内存算堆 → 容器 limit 8G 但 JVM 堆 32G，启动即 OOM，必须 `-XX:+UseContainerSupport`（JDK8u191+）或显式 `-Xmx`；节点驱逐阈值（memory.available < 100Mi 等）与容器 limit 是两条独立防线，都要监控。

**延伸考点：** 节点内存压力驱逐与容器 OOMKilled 的触发顺序和判定指标分别是什么？把 Burstable 全改成 Guaranteed 对集群利用率有什么影响？

---

### Q9. 大促流量翻 5 倍，HPA 扩容慢半拍、收尾缩容掐断连接，怎么调？

**问题：** 大促流量涨 5 倍，HPA 扩容总比流量慢，且新 Pod 要 40 秒初始化；活动结束缩容时又掐断了在线连接。从 HPA 机制与参数角度，你怎么调优？

**期望加分项：**
- 能讲清 HPA 数据链路：metrics-server 采集（默认 15s 间隔）→ controller 计算期望副本数（ceil(当前值/目标值 × 当前副本数)）→ 更新 Deployment
- 能给出扩容慢的量化根因与解法：指标采集延迟 + 目标值设置偏高 + Pod 启动慢 → readiness/startup 探针、预扩容（大促前人工 scale）、基于 QPS 的自定义指标（Prometheus Adapter/KEDA）
- 能讲清稳定窗口：`--horizontal-pod-autoscaler-tolerance`（默认 0.1）、behavior 配置里 scaleDown 的 stabilizationWindowSeconds（默认 300s），缩容一次只降一部分
- 缩容保护：minReplicas 兜底、稳定窗口、配合 PDB（PodDisruptionBudget）与优雅终止
- 知道 HPA 与 Cluster Autoscaler 的区别与联动：Pod 扩容后节点不够要 CA 加节点，时序上 Pod 先 Pending

**减分项：**
- 只背"HPA 根据 CPU 自动扩缩容"
- 不知道 metrics-server 的采集延迟与 tolerance 的容差机制
- 扩容目标值设太高导致永远不扩；或 targetCPUUtilizationPercentage 与 request 的关系说不清
- 缩容瞬间把副本砍没，没有稳定窗口概念
- 不区分 HPA（Pod 数）与 Cluster Autoscaler（节点数）

**解答：**

先算账：metrics-server 15s 采一次、HPA 同步周期默认 15s、Pod 启动 40s（含镜像拉取与 JVM 启动），从流量涨到新 Pod 就绪至少要 15+15+40≈70 秒，这就是"慢半拍"的数学来源。优化分三层：① 缩短链路——指标采集走 Prometheus Adapter（可以 5-10s 拉取）并让 HPA 针对自定义指标（如 QPS、P99 延迟）而不是 CPU 代理指标；② 提前量——大促前按预估容量人工预扩（`kubectl scale` 或直接提高 minReplicas），把 40 秒初始化时间从"流量路径"里挪出去；③ 加速就绪——镜像预热（节点预先拉取）、startup 探针 + readiness 就绪即接流、JVM 用 CDS/分层编译减少预热。缩容问题用 behavior 解决：新版 HPA 支持 `behavior.scaleDown.stabilizationWindowSeconds: 300`（5 分钟内持续低水位才缩）、`selectPolicy: Max` 与 `policies` 限制单步缩容比例（如一次最多缩 25%）；配合 minReplicas 兜底（大促后回到 2，而不是 0）和 PodDisruptionBudget 保证缩容不触发 PDB 阻止。调参建议：`targetCPUUtilizationPercentage` 设 60-70%（太高等于不扩），tolerance 0.1 以下更灵敏但会锯齿抖动。坑：HPA 只能保证"扩容了"，节点容量不够时新 Pod 全 Pending——要接 Cluster Autoscaler（按 Pending Pod 数量扩节点），且 CA 加节点分钟级延迟，大促必须预扩节点或常驻 buffer 节点；HPA 缩容与 CA 缩容要联动配置，否则节点缩了 Pod 还挂着，下次扩容又要等节点。

**延伸考点：** HPA 目标值 50%、当前使用率 40% 时副本数怎么算（考虑 tolerance）？KEDA 相比原生 HPA 解决了什么问题、ScalingObject 怎么跨工作负载扩缩？

---

### Q10. 服务启动要 30 秒预热，一上线就被打爆；liveness 偶尔误杀重启，探针怎么设计？

**问题：** 你们的服务启动后要 30 秒做缓存预热，但一上线流量就进来全部报错；另一个服务的 liveness 探针偶发超时导致 Pod 被反复重启，业务投诉。探针体系怎么设计才合理？

**期望加分项：**
- 能讲清三探针职责：liveness（进程是否活着，失败就杀）、readiness（能否接流量，失败摘除不杀）、startup（慢启动保护，成功前 liveness 不执行）
- 慢启动解法：startup 探针 `failureThreshold × periodSeconds` 给出足够启动窗口（如 30s），启动成功前 liveness 不参与，避免"预热没完成就被 liveness 杀掉"
- 探测目标设计：用专用 /healthz 接口；DB/Redis 等依赖放 readiness（依赖抖动只摘流量不重启进程），别放 liveness——依赖抖动导致连环重启是常见事故
- 参数量化：periodSeconds（10-15s）、timeoutSeconds（1-3s，别和 HTTP 超时混）、failureThreshold（readiness 3 次、liveness 常设 1-2 次避免误杀）、initialDelaySeconds
- 能举误杀案例：GC 停顿导致探测超时、探针接口被限流或走了业务线程池、Exec 探针 fork 进程开销

**减分项：**
- 分不清 liveness 失败重启、readiness 失败摘除的行为差异
- 用 liveness 探测外部依赖，依赖一抖整个服务连环重启
- 探测接口"假健康"：返回 200 但业务没就绪（探针不走初始化逻辑）
- 不看 kubelet 事件与探针日志就调参
- 参数全凭感觉：periodSeconds 设 1s 把服务探出问题，timeout 大于 period 导致探测积压

**解答：**

设计原则一句话：liveness 只回答"进程还活着吗"，readiness 回答"现在能接流量吗"，startup 回答"还没准备好之前别拿前两个烦我"。慢启动场景：设 startup 探针 `httpGet: /healthz, periodSeconds: 2, failureThreshold: 20`（共 40s 窗口），startup 成功前 liveness/readiness 不执行，预热完成即转为正常探针周期——这是解决"30 秒预热被打爆"的正解，靠加大 initialDelaySeconds 只能猜时间、且猜短了照样杀。依赖治理：把 DB/Redis/下游服务可用性放到 readiness——依赖抖动时 Pod 标记 NotReady、从 Service endpoints 摘除、流量自动避开，进程不重启；liveness 只留"进程存活 + 主线程没死"级别的检查（如 /healthz 只读本地状态），避免"依赖抖动 → liveness 失败 → 重启 → 重启风暴"。参数建议：periodSeconds 10-15s，timeoutSeconds 1-2s，liveness failureThreshold 1-2（误杀代价远大于晚杀 10 秒）、readiness failureThreshold 3；探测接口别走业务线程池或加自己的超时，防止接口慢拖垮探测。误杀案例：Full GC 停顿超过 timeoutSeconds 时 liveness 超时 → Pod 被杀 → 重启后又要 Full GC，形成死循环，解法是 liveness 用本地超短检查 + failureThreshold 放宽；Exec 探针每次 fork 一个进程，高频率探测在低配节点上本身就制造负载。验证手段：`kubectl get events` 看 Killing/Restarting 事件、`kubectl describe pod` 看探针的 Last Probe 与状态、临时把探针路径 curl 出来对比。

**延伸考点：** readiness 与 startup 都失败时，Pod 状态和 Service endpoints 分别怎么变？TCP 探针 vs HTTP 探针在哪些场景下会互相误判？

---

### Q11. 跨节点 Pod 通信延迟高一倍，租户要求网络隔离，CNI 怎么选？

**问题：** 你们集群跨节点 Pod 通信延迟明显高于同节点，业务怀疑是网络插件问题；同时安全要求不同团队之间网络默认隔离。从 CNI 选型与 Overlay 架构角度，你如何分析、如何解决？

**期望加分项：**
- 能讲清 CNI 三大职责：IPAM（IP 分配）、连通性（数据面转发）、策略执行（NetworkPolicy），且三者可来自不同插件
- 能对比主流插件：Flannel（VXLAN/host-gw，简单、功能少）、Calico（BGP 直连 / IPIP 封装，自带 NetworkPolicy）、Cilium（eBPF 数据面，性能最好、功能最全）
- 能讲清 Overlay 开销：VXLAN 加 50 字节头 + 封装解封装 CPU 开销（吞吐损耗约 5-10%、延迟增加 0.1-0.5ms 量级）；host-gw/BGP 直连模式无封装
- 能讲 MTU 的坑：隧道 MTU 默认 1450（VXLAN 1450/Calico IPIP 1480），外部网络/云厂商 MTU 1500，大包会分片丢包，业务偶发卡顿先查 MTU
- NetworkPolicy 实现差异：Calico iptables/IPset、Cilium eBPF，默认拒绝策略怎么写（podSelector: {} + ingress 空列表）
- 排障手段：ping/tcpdump、calicoctl node status、检查隧道端口（4789/8472）、MTU 探测

**减分项：**
- 只知道"Flannel 是 Overlay"，讲不出其他模式与替代品
- 不知道 MTU 对网络性能与连通性的影响
- 把 kube-proxy（四层转发）和 CNI（Pod 网络）职责混在一起
- 跨节点延迟高只怪网络插件，不看 kube-proxy 路径、conntrack、节点负载
- NetworkPolicy 不生效时不会排查（Pod 标签、policyTypes、方向）

**解答：**

先分清两条链路：CNI 管"Pod 与 Pod/节点之间"的二三层网络，kube-proxy 管 Service 的四层转发，两者叠加才是完整数据路径。跨节点延迟高的排查顺序：① 排除 Overlay 封装损耗——确认插件模式，Calico 可查 `calicoctl node status` 看 BGP peer 状态，IPIP/VXLAN 封装模式比 host-gw/BGP 直连多一跳；在节点间跑 `ping -M do -s 1400/1450/1472` 分段探测，MTU 不匹配（隧道 1450 vs 物理 1500）时大包被丢弃表现为"小包通、大包丢、TCP 慢"；② 排除 kube-proxy 与 conntrack——跨节点访问 Service 要查 `iptables -t nat` 链命中与 conntrack 表满；③ 排除节点侧——`top` 看是否 CPU 打满（封装解封装吃 CPU）、网卡软中断。选型建议：规模小求简单用 Flannel host-gw（同网段直连）或 VXLAN；要网络策略、要跨网段路由用 Calico（BGP 直连，性能优于 IPIP）；对性能敏感、要用 eBPF 观测（带宽/延迟指标）选 Cilium——线上比较：同样 64B 小包 PPS，Cilium eBPF 比 iptables 转发高一个量级，延迟低约 0.05-0.1ms。NetworkPolicy 落地：先建"默认拒绝"策略（`spec: podSelector: {}`，`policyTypes: [Ingress]`，ingress 为空数组），再按业务白名单放行——注意 NetworkPolicy 只作用于 Pod 间流量，对 Service NodePort/外部流量默认放行，别指望它当南北向防火墙；策略不生效先查 Pod 是否被多策略叠加（任一策略不匹配即拒绝）、CNI 是否支持（Flannel 原生不支持需 Calico/Cilium/OVN）。坑：混合网段（Pod 网段与 VPC 重叠）会导致 BGP 路由冲突，规划 Pod CIDR 时避开云厂商 VPC 网段。

**延伸考点：** eBPF 模式相比 iptables 模式的转发路径差异在哪、延迟低在哪一环？NetworkPolicy 的 egress 策略漏配会导致什么线上事故？

---

### Q12. 一批新 Pod 三种异常状态，怎么用命令链快速止血？

**问题：** 发布后 `kubectl get pods` 里一批 Pending、一批 ImagePullBackOff、一批 CrashLoopBackOff。你拿到输出后按什么顺序排查？给出你的命令路径和每一步的判定逻辑。

**期望加分项：**
- 有系统化排障路径：状态分类 → describe 看事件 → logs 看日志（含 --previous）→ 退出码 → 节点/资源/存储侧
- Pending 分支：`kubectl describe pod` 看 Events 的 FailedScheduling（资源/污点/标签不匹配）、PVC 未绑定、镜像拉取阻塞
- ImagePullBackOff 分支：看事件里 ErrImagePull 的具体原因——镜像名/标签不存在、私有仓库鉴权失败（imagePullSecrets）、仓库不可达、Harbor 限流；`kubectl describe` 的 Events 会给出拉取报错原文
- CrashLoopBackOff 分支：`kubectl logs <pod> --previous` 看上次崩溃日志、退出码语义（1/137/143）、探针误杀、配置缺失；知道 BackOff 退避（10s 起翻倍至 300s）
- 会用 `kubectl get events -A --sort-by=.lastTimestamp` 全局看事件时间线，区分存量与新发问题

**减分项：**
- 一上来就 delete/restart Pod，不做 describe，问题永远复现
- 不看退出码乱猜根因
- 不知道 `--previous` 能看上次崩溃日志
- 忽略节点层故障（NotReady、DiskPressure/内存压力驱逐）
- 只看当前输出不复盘事件时间线

**解答：**

标准路径是"先分类、再深挖"：`kubectl get pods -o wide` 拿到状态与所在节点，`kubectl get events -A --sort-by=.lastTimestamp | tail -50` 先全局扫一眼时间线。三类分别处理：① Pending：`kubectl describe pod` 底部 Events 里的 FailedScheduling 会直接写出原因——`Insufficient cpu/memory`（看节点剩余 vs request）、`didn't match node selector/taint`（标签或污点）、`persistentvolumeclaim not found`（PVC 还 Pending，去 describe pvc）。② ImagePullBackOff：事件里带具体拉取错误，按层级定位——镜像名 tag 拼错（`manifest unknown`）、私有仓库鉴权失败（`pull access denied`，查 Pod 有没有 imagePullSecrets、Secret 是否过期）、仓库不可达（`connection refused`，从节点上手动 `docker pull` 验证网络）、Harbor 镜像被 GC 删 tag（`not found`）；注意事件里 `ErrImagePull` 是原因、`ImagePullBackOff` 是状态，退避重试 10s→300s。③ CrashLoopBackOff：`kubectl logs <pod> --previous` 看崩溃前日志，配合退出码定性——1 应用错误（日志必有异常）、137 大概率 OOM（describe 看 `OOMKilled: true` 或节点 dmesg）、143 是优雅终止没处理完被杀、0 但还在重启（检查是不是 restartPolicy 与 Job 语义）；还要警惕探针误杀：事件里 `Liveness probe failed: ...` + `Killing container` 连在一起就是探针问题。收尾：确认根因后走"修复 → 重新发布 → 观察恢复曲线"，并回查 `kubectl get events` 看是否还有反复；每个节点 NotReady、DiskPressure 这种全局性故障会同时制造 Pending 和 CrashLoop 两类现象，先排除节点层再查应用层。

**延伸考点：** ImagePullBackOff 与 ErrImagePull 的区别是什么、退避时间如何增长？CrashLoopBackOff 里"0 退出码但反复重启"的典型成因有哪些？

---

### Q13. CI/CD 系统要建删 Pod，安全审计还发现 token 泄漏，RBAC 怎么设计？

**问题：** 你们的 CI/CD 流水线需要在某个命名空间创建/删除 Pod，但运维要求最小权限且不能碰生产；与此同时审计发现仓库里有 serviceaccount token 被扫到。RBAC 四件套怎么用？泄漏怎么评估和处置？

**期望加分项：**
- 能讲清 Role/ClusterRole/RoleBinding/ClusterRoleBinding 的关系与绑定规则：Role 限命名空间、ClusterRole 全局，RoleBinding 可绑 ClusterRole 做命名空间内授权（常用组合）；ClusterRoleBinding 才是全局绑定
- 最小权限设计：apiGroup/resources/verbs 精确到资源与动作（如 `pods: get/list/watch/create/delete`），避免 `*` 通配；生产用独立命名空间 + 独立 ServiceAccount，绝不 cluster-admin
- 能用 `kubectl auth can-i create pods --as=system:serviceaccount:ci:builder` 验证权限、`--list` 列出可执行操作
- 知道 serviceaccount 与 token 机制：Pod 内默认自动挂载 SA token（可 `automountServiceAccountToken: false` 关闭）、v1.24+ 不再自动生成长期 token、用 TokenRequest 拿短时 token
- 泄漏处置有完整流程：先评估影响面（TokenRequest 短时 token 则评估有效期与来源，长期 token 视为失陷）→ 立即 revoke/rotate → 查 kube-apiserver 审计日志（audit log）确认是否被用来调 API → 复盘加固（SA 最小权限、关闭自动挂载、token 引入外部密钥管理）

**减分项：**
- 分不清 Role 和 ClusterRole 的授权范围与绑定规则
- 一上来给 cluster-admin 或 `verbs: ["*"]`
- 不知道 SA token 自动挂载机制，说不清 Pod 里的 token 从哪来
- 处置泄漏只删 token 不查审计日志，无法评估实际损失
- 对 v1.24+ 的 SA token 生命周期变化毫无概念

**解答：**

RBAC 设计套路：每个"系统身份"一个 ServiceAccount，绑定精确权限。以 CI 为例：建 `ci-builder` SA，Role 定义 `apiGroups: [""], resources: [pods, deployments, services], verbs: [get, list, watch, create, update, delete]`（只给流水线需要的动作，不给 secrets/events 等无关资源），再 RoleBinding 绑定到 ci 命名空间；流水线 kubeconfig 里用该 SA 的 token。生产隔离：CI 与生产用不同命名空间/集群，生产侧只有发布系统自己的 SA 且权限再收敛（只允许 create 不能 delete、只能动自己的发布 namespace）。验证：`kubectl auth can-i --list --as=system:serviceaccount:ci:builder -n ci` 直接列出该身份权限清单，用于审计与自检。token 泄漏处置分三步：① 定性——先看 token 类型：v1.24+ 默认 TokenRequest 是短时（默认 1h）且绑定 Pod 的投影 token，泄漏窗口小；手工创建的长期 Secret token 视为失陷，立即 `kubectl delete secret <sa-token>` 并重建（旧 token 即刻失效）；② 定损——开/查审计日志（apiserver `--audit-log-path`），按该 token 检索请求记录，确认是否被用来执行过操作（查询 vs 写操作后果完全不同）；③ 加固——SA 上设 `automountServiceAccountToken: false` 关掉无谓的默认挂载、CI token 改为短时 TokenRequest 或引入 OIDC/云厂商 IRSA（EKS 的 IAM Role for SA）实现免 token 身份，敏感仓库做 secret 扫描进 CI 门禁。坑：审计日志默认可能没开，事后无法取证，所以"先开审计再出事"是底线；Namespace 与集群级别的权限边界一定要在文档里写死，防止后续"图省事"给 cluster-admin。

**延伸考点：** v1.24 移除 Secret-based token 自动创建后，老的长期 token 还在生效吗、怎么识别？用 OIDC/IRSA 替代 SA token 的迁移成本和工作量是什么？

---

### Q14. Zookeeper 集群迁进 K8s，要稳定标识、独立存储、有序启停，StatefulSet 怎么满足？

**问题：** 你们要把 3 节点 Zookeeper（或 Kafka）迁进 K8s，要求每实例有稳定网络标识、独立持久化存储，且扩缩容按顺序进行。StatefulSet 的特性分别怎么对应这些需求？为什么必须配 headless Service？缩容时存储和顺序会发生什么？

**期望加分项：**
- 能讲清 StatefulSet 三大保证：稳定网络标识（`pod-0.svc...`，Pod 重建后名字与 DNS 不变）、稳定存储（volumeClaimTemplates 按序创建 PVC，Pod 重建重挂同一块盘）、有序部署/缩容（扩 0→N 逐个就绪，缩 N→0 逆序逐个删除）
- 能讲清 headless Service（clusterIP: None）的作用：为每个 Pod 提供独立 DNS 记录 `pod-0.zk-hs.ns.svc.cluster.local`，有状态集群节点互访靠它（Zookeeper 的 quorum 连接、Kafka 的 broker 注册）
- 更新策略：RollingUpdate 按序滚动 + partition 字段做金丝雀（如 partition: 2 只更新序号 >=2 的 Pod）
- 缩容行为：Pod 逆序删除但 PVC 保留（数据不丢，重扩回来原 Pod 重新绑定原盘）；手动删 PVC 才删数据
- 坑：缩容 3→1 后如果控制器/集群配置还引用另外两个节点会脑裂；PVC 数量与 Pod 序号强绑定，不能随机删除

**减分项：**
- 不知道 headless Service 是干什么的，以为只是"不用 ClusterIP"
- 认为缩容会删除 PVC、数据丢失
- 说不清扩缩容的严格顺序与"逐个 ready 才继续"的机制
- 无持久化诉求也硬用 StatefulSet
- 不知道 partition 能做金丝雀，只会整批更新
- 有状态服务裸跑 Deployment + 共享盘（把 ZK 数据放 NFS）

**解答：**

StatefulSet 与 Deployment 的本质差异在"身份"：每个 Pod 序号即身份，`pod-name.serviceName` 的 DNS 在 Pod 重建后不变。稳定性靠三块拼出来：① 网络——配 headless Service（`clusterIP: None`），它不为 Service 分配虚拟 IP，而是为每个端点生成独立 DNS：`zk-0.zk-hs.default.svc.cluster.local`。Zookeeper 配置里 myid/连接串直接用这些名字，节点重启后 IP 变了但名字不变，集群成员关系不破裂——这就是"为什么必须有 headless"的答案（普通 ClusterIP Service 只有一组随机 DNAT，没有逐 Pod 身份）；② 存储——`volumeClaimTemplates` 声明 PVC 模板，StatefulSet 为每个序号生成 `data-zk-0/data-zk-1/data-zk-2` 各一块盘，Pod 删除重建后按序号重新绑定原盘（Kubelet 以 PVC 名定位），数据随 Pod 生命周期保留；③ 顺序——创建时 `zk-0` 就绪才建 `zk-1`（RollingUpdate 更新也按序：`zk-2` 先更、`zk-0` 最后），缩容时逆序：3 副本缩到 1，删 `zk-2`、再删 `zk-1`，`zk-0` 保留。要点：缩容只删 Pod 不删 PVC——重扩回 3 时 `zk-1/zk-2` 会重新挂回原盘，数据完整；所以"想清数据"必须显式删 PVC，这是刻意设计的保护。partition 灰度：`updateStrategy: {type: RollingUpdate, partition: 2}` 只更新序号 ≥2 的 Pod，先验证一个再逐步降到 0 全量。坑：缩容前先把集群自身的成员配置同步好（ZK 的 server 列表、Kafka 的 broker 下线），否则控制器缩了、集群内部还认为有 3 个节点会脑裂；大规模有状态系统优先评估 Operator（zk-operator、kafka-operator、MySQL Operator），StatefulSet 只是底座，选举、备份、扩缩容编排 Operator 做得更完整。

**延伸考点：** StatefulSet 的 Pod 被节点驱逐后重建，怎么保证挂回原来的盘？`partition` 从 0 调到 N 后更新行为怎么变化？

---

### Q15. 日志采集、一次性迁移、定时备份，分别用哪种工作负载？定时任务超时重叠怎么办？

**问题：** 集群里每个节点都要跑日志采集 Agent，一次性的历史数据迁移任务，以及每天凌晨的数据库备份任务，分别适合 DaemonSet、Job、CronJob 中的哪一种？如果备份任务执行时间超过了下一次触发时间，会发生什么？

**期望加分项：**
- 能按语义准确归类：每节点一个 → DaemonSet（日志 Filebeat、监控 node-exporter、网络插件 CNI）；一次性任务 → Job（completions/parallelism/backoffLimit）；周期任务 → CronJob（schedule 表达式）
- DaemonSet 细节：默认每节点一个副本，天然容忍节点污点与 nodeSelector/亲和限制，滚动更新策略（OnDelete/RollingUpdate）
- Job 细节：completions 与 parallelism 的含义（完成几次/并发几个）、backoffLimit 控制重试次数、activeDeadlineSeconds 限制总时长、失败重试是新建 Pod 不是重启
- CronJob 细节：concurrencyPolicy（Allow 默认/Forbid/Replace）、startingDeadlineSeconds（错过调度的补偿窗口）、successfulJobsHistoryLimit/failedJobsHistoryLimit 控制历史清理
- 重叠问题：CronJob 上次还没跑完下次又触发 → 用 Forbid 拒绝并发或 Replace 杀掉重来；备份类用 Forbid + 上次超时告警
- 会手动触发：`kubectl create job --from=cronjob/xxx manual-run` 补跑

**减分项：**
- 用 Deployment 在集群里跑每节点 Agent（副本数永远对不上节点数）
- 不知道 concurrencyPolicy 的语义，任务重叠时数据重复/互相冲突
- 不知道 CronJob 的时区坑（K8s 1.27 之前调度按 etcd/controller-manager 的 UTC 时区，凌晨 2 点变成上午 10 点）
- 不设 history 限制，CronJob 攒几万个已完成的 Job/Pod 把 etcd 撑爆
- Job 失败后不知道看 backoffLimit 与 Pod 重试机制，一直以为"重跑=重启 Pod"

**解答：**

按"目标对象"归类：目标是"每个节点"用 DaemonSet（副本数 = 节点数，节点加进来自动补一个，节点故障自动在别处重建；配 tolerations 容忍 NotReady 等污点才能保证 Agent 永远在）；目标是"跑一次直到成功"用 Job；目标是"按时间反复跑"用 CronJob。典型组合：Filebeat/Node-Exporter 用 DaemonSet，历史数据迁移用 Job（`parallelism: 4, completions: 20` 把 20 批任务并发 4 个地跑，`backoffLimit: 5` 容忍单批失败重试），备份用 CronJob（`schedule: "0 2 * * *"`，注意时区——1.27 前 CronJob 控制器按 UTC 解释 schedule，想按上海时区凌晨 2 点跑要写 `schedule: "0 18 * * *"` 或用新版的 `timeZone` 字段）。重叠问题：备份跑了 6 小时，第二天凌晨 2 点又触发——默认 `concurrencyPolicy: Allow` 会再起一个实例，两个备份写同一文件必然出问题；正确做法是 `Forbid`（上次没结束就跳过这次触发，但要接告警：跳过 = 数据备份缺口）或 `Replace`（杀掉旧实例重新跑，适合可中断任务）。配套参数：`startingDeadlineSeconds: 600` 让错过的调度在窗口内补偿执行；`successfulJobsHistoryLimit: 3, failedJobsHistoryLimit: 1` 控制历史 Job 保留量——不设的话每天一次任务攒一年几百个 Job 与几千个已完成 Pod，etcd 对象数量膨胀导致 apiserver 变慢。坑：Job 的 Pod 失败重试是"新建 Pod"不是重启旧 Pod，所以应用要保证"可重入"（幂等），否则重试会把脏数据再写一遍；CronJob 所在命名空间被误删、Pod 因镜像拉取失败挂起时，事件里只有 `Job suspended` 之类的痕迹，要盯 `kubectl get cronjob,get job` 的状态差异。

**延伸考点：** DaemonSet 滚动更新策略 OnDelete 与 RollingUpdate 的区别、更新一个节点 Agent 的完整流程？CronJob 的 `startingDeadlineSeconds` 为 0 和缺省时的行为差异是什么？

---

### Q16. 一个集群 5 个团队 30 个应用，A 团队把节点打满影响 B 团队，怎么治理？

**问题：** 你们一个集群里跑着 5 个团队 30 多个应用，A 团队一个"吃满内存"的 Pod 让节点不稳，B 团队跟着遭殃。你如何用命名空间 + 配额 + LimitRange 体系治理？多租户"隔离"要做到什么程度才算够？

**期望加分项：**
- 能讲清 ResourceQuota 与 LimitRange 的分工：quota 管"命名空间总量"（requests/limits/pods/configmaps 等维度），LimitRange 管"单个 Pod/容器上下限"并给无声明资源自动补默认值
- 能写出配额样例：`requests.cpu/memory`、`limits.cpu/memory`、`pods`、`count/secrets` 等维度，注意 quota 必须覆盖 requests 与 limits 两类否则超发
- 能讲清多租户隔离的层次模型：资源隔离（quota）→ 网络隔离（NetworkPolicy）→ 调度隔离（节点池/污点）→ 权限隔离（RBAC + 独立 SA）→ 安全隔离（PodSecurity/PSS 准入），按信任级别选择
- 知道 LimitRange 不设默认值时 Pod 不带 request/limit 会以 BestEffort 运行、还可能绕过 quota 语义（quota 只统计声明了的值）
- 有治理配套：团队配额预算（按历史水位分账）、超发/利用率监控、成本核算（命名空间标签 + 计量）、配额超限的告警

**减分项：**
- 把命名空间当成唯一的隔离手段
- 只配了 quota 的 cpu 维度没配 memory，A 团队照样把内存打满
- 不知道 LimitRange 的 default/defaultRequest 机制，Pod 无声明时行为不可控
- 没有成本分摊与配额预算意识
- 忽略安全隔离（PodSecurity 准入、非 root 运行）层面的多租户风险

**解答：**

先把"隔离"拆成五层，按团队信任度与合规要求选择：① 资源隔离——每个团队一个命名空间，配 ResourceQuota 限定总量：`requests.cpu: "80"`、`limits.memory: 200Gi`、`pods: "50"`；注意 quota 是按"声明值"统计的，Pod 没写 request/limit 时按 0 算，所以必须同时用 LimitRange 给每个容器补默认值——LimitRange 配 `default: {cpu: 500m, memory: 1Gi}`、`defaultRequest: {cpu: 250m, memory: 512Mi}`，A 团队即使不写资源声明也会被套上默认值、被 quota 和 QoS 语义约束住。② 网络隔离——跨团队默认拒绝，用 NetworkPolicy 按命名空间 label 放行（见 Q11）。③ 调度隔离——关键/敏感团队用独立节点池 + 污点，避免资源争抢。④ 权限隔离——每个团队独立 SA + RoleBinding 只绑本命名空间，禁止跨空间（见 Q13）。⑤ 安全隔离——开 PodSecurity 准入（PSS 的 restricted 级别）强制非 root、只读根文件系统等。配额预算的工程做法：按团队过去 3 个月 P99 水位 × 1.3 定配额，配额使用率 80% 告警、95% 阻止新发布（配合 CI 检查 quota 余量）；成本核算用命名空间标签（`team=a, env=prod`）对接计量系统按 request 计费分摊。坑：quota 是"创建时校验"，Pod 创建瞬间超配额会被拒绝（事件里 `exceeded quota`），但运行期实际用超不会被打断——所以还要靠 LimitRange/QoS 控制单 Pod 上限；quota 里 `count/ingresses`、`count/services` 这类资源数量配额容易被漏配，团队疯狂建 Service 把 apiserver 拖慢。先立规再放量：新团队默认 restricted 模板（quota + limitrange + networkpolicy + PSS），评审后才放开。

**延伸考点：** ResourceQuota 的 `requests` 和 `limits` 维度统计口径分别是什么、只配 requests 会有什么漏洞？PSS 的 privileged/baseline/restricted 三档对业务改造的约束差异？

---

### Q17. etcd 一慢整个集群只读，备份没演练过，控制平面高可用怎么做？

**问题：** 某天大量节点 NotReady、`kubectl` 只能读不能写，最后定位是 etcd 空间/性能问题。etcd 在 K8s 里到底存什么？你们怎么做 etcd 的备份、恢复演练和控制平面 HA？

**期望加分项：**
- 能讲清 etcd 存"集群期望状态"（对象定义、Secret/ConfigMap 内容、ServiceAccount token 等），不存业务数据；kube-apiserver 是唯一读写入口，etcd 本身无查询语义
- 高可用架构：奇数节点（3/5 均可）、独立 SSD（fdatasync 是延迟关键，机械盘直接拖垮）、跨可用区部署；apiserver 多副本 + 前端 LB，controller-manager/scheduler 靠 leader 选举自动选主
- 性能与空间治理：历史版本堆积 → `etcdctl defrag`（碎片）、`etcdctl compact`（压缩旧版本）；默认空间配额 2GB（`--quota-backend-size`），满了之后 etcd 拒绝写入 → apiserver 只读，这是"只能读不能写"的典型根因
- 备份：`etcdctl snapshot save` 定时 + 异地存储（对象存储），恢复演练过：`etcdctl snapshot restore` 时 `--name`/`--initial-cluster`/`--initial-advertise-peer-urls` 与目标成员地址必须一致
- 影响链路：etcd 慢/抖动 → apiserver 读写超时 → 节点 Lease 续约失败 → 节点 NotReady → 失联节点上的 Pod 被重新调度/驱逐（5 分钟容忍期）

**减分项：**
- 认为 etcd 存业务数据或日志
- 从不备份或备份了但没做过恢复演练，出事不会 restore
- 空间不足只知道清数据，不知道 compact + defrag 的组合
- 控制平面高可用只答"多副本"，说不出 apiserver/etcd/scheduler/controller-manager 各自的 HA 机制
- 恢复时成员信息配置错（--initial-cluster 与现网不符）导致集群起不来

**解答：**

etcd 的角色一句话：K8s 的"状态数据库"，存所有 API 对象的期望状态（含 Secret 明文），apiserver 是唯一入口，业务数据不进 etcd。高可用与控制平面 HA 分开谈：控制平面四个组件——etcd（3 或 5 节点，quorum 分别容忍 1/2 台故障，放独立 SSD 与独立可用区）、apiserver（无状态多副本 + LB，前置 keepalived/云 LB）、controller-manager/scheduler（多副本靠 leader election 选主，无需 LB）。"只能读不能写"的根因排查：先 `etcdctl endpoint status -w table` 看 db size 是否逼近配额（默认 2GB），满了 etcd 拒绝所有写请求 → apiserver 变只读、创建/更新全部超时失败；处理是 `etcdctl compact`（压缩历史 revision）后 `etcdctl defrag`（回收空间碎片），并检查是否有循环写 etcd 的控制器。备份与恢复：备份命令 `ETCDCTL_API=3 etcdctl --endpoints=... snapshot save backup.db`，生产要求每小时 + 每日异地；恢复流程 `etcdctl snapshot restore backup.db --data-dir=/var/lib/etcd-restore --name=etcd-1 --initial-cluster=etcd-1=http://10.0.0.1:2380,... --initial-advertise-peer-urls=http://10.0.0.1:2380`，恢复后成员地址与 apiserver 的 etcd 地址配置必须一致，否则集群起不来——这就是"必须演练"的原因：restore 命令的坑（成员名、peer URL、data-dir 权限）只有真跑过才记得。影响链路认知：etcd 慢 → apiserver 全部读写超时 → kubelet 的 Lease 续约失败（默认 10s 续约）→ 节点 40s 后标记 NotReady → 再过 5 分钟 Pod 被视为失联被驱逐重建 → 整个集群"雪崩式"重建，所以 etcd 的监控（延迟 P99、db size、慢查询）必须进核心告警。坑：etcd 磁盘 IO 与网络延迟是同一集群内数据一致性质量的直接决定因素，和业务混部在共享存储上是大忌。

**延伸考点：** etcd 节点数选 3 还是 5，依据是什么（故障容忍与写性能）？`compact` 之后为什么还要 `defrag`，两者分别解决什么问题？

---

### Q18. 要按 header 灰度、统一 trace，有人提议上 Istio，服务网格和 Ingress 到底差在哪？

**问题：** 你们微服务要做按 header 的灰度分流、统一收集七层 metrics 与 trace，有人提议上 Istio。服务网格和你现在的 Nginx Ingress + 应用埋点方案本质区别是什么？边车（sidecar）模式有什么代价？

**期望加分项：**
- 能一句话点破方向差异：Ingress/网关是"南北向"（外部 → 集群入口），服务网格是"东西向"（服务间调用）＋ 可选入口网关；两者不是替代而是叠加关系
- 能讲清边车原理：Istio 给每个 Pod 注入 Envoy sidecar，通过 iptables 透明劫持进出流量（流量无感、代码零改造），VirtualService/DestinationRule 做路由与流量策略
- 能给出灰度例子：`VirtualService` 按 header `version: canary` 分流 + `DestinationRule` 定义子集，权重灰度（weight: 90/10）
- 可观测性：sidecar 自动产生七层 metrics（P99/错误码）、配合 Jaeger/Tempo 的 trace 传播（无需改代码），对比应用埋点方案的成本
- 能讲代价：资源（每个 Pod 多一个 Envoy，内存约 20-50MB、CPU 若干核的千分之几）、延迟（一跳转发微增 0.1ms 级）、复杂度（大量 CRD、版本升级、劫持排障难——istioctl proxy-status/describe 排查）
- 有落地取舍：渐进式（先只开可观测性，不开流量管理）、按命名空间选注入、不盲目全量上

**减分项：**
- 分不清南北向/东西向，以为 Istio 是"高级网关"
- 不知道边车的透明劫持机制，出了问题连数据路径都说不清
- 只背"服务网格 = 微服务治理"，给不出一个 VirtualService 配置
- 忽视 sidecar 的资源成本与全量注入的风险面
- 一上来全量 mesh，没有渐进与灰度意识

**解答：**

先定边界：Nginx Ingress 处理"外部流量进集群"（南北向），到集群后服务间互相调用（东西向）它管不到；Istio 用边车接管的是服务间流量（东西向）——它也有入口网关（istio-ingressgateway，本质是 Envoy）可做南北向，所以正确认知是"Ingress 网关与服务网格各管一段、可叠加使用，不是二选一"。边车模式的核心是"透明代理"：Pod 创建时被注入 Envoy 容器（sidecar），istio-init 容器在 Pod 网络命名空间里插 iptables 规则，把进出 Pod 的流量劫持给 Envoy——所以应用零改造，但代价也在这：每个 Pod 多一个进程（内存 20-50MB，1000 个 Pod 就多 20-50GB）、请求多一跳（延迟微增）、所有流量都过 Envoy 意味着 Envoy 故障 = 应用不可达（所以有 `traffic.sidecar.istio.io/includeInboundPorts` 等豁免手段）。灰度落地：`DestinationRule` 定义 `subsets: [v1, v2]`，`VirtualService` 里 `match: headers: {env: {exact: canary}}` → 路由到 v2 子集，或用 `weight: 90/10` 按比例——相比 Ingress 按路径/域名路由，网格的灰度粒度是"请求级 header/cookie"与"权重"，这是本质能力差异。可观测性收益：sidecar 自动输出 RED 指标（Rate/Errors/Duration）与 trace span，应用无需埋点 SDK，这是选它最实在的理由；同时内置 mTLS（双向 TLS）加密服务间流量，安全合规加分。实践建议：第一批只开 metrics/trace 与 mTLS，不开路由劫持的灰度能力（风险最小）；排障先学 `istioctl proxy-status`（看配置同步）、`istioctl proxy-config listeners`（看劫持规则），全量注入前先在小命名空间试 1-2 周。坑：Java 应用与 sidecar 的启动顺序（Envoy 未就绪时应用出站流量被劫持报错，需配 `holdApplicationUntilProxyStarts`）、Istio 升级是跨版本 Envoy 配置兼容的大工程，要有演练预案。

**延伸考点：** 边车流量劫持排障时，`istioctl proxy-status` 和 `proxy-config` 分别回答什么问题？不用边车的"Ambient mesh"（无 sidecar）方案解决的是什么代价、适合什么场景？

---

### Q19. 发布后镜像全拉不下来，私有仓库还总是慢，imagePullPolicy 和仓库侧怎么治？

**问题：** 内网私有镜像仓库（Harbor），某次发布后新 Pod 全部 ImagePullBackOff，但几分钟前同一镜像还能拉；另一头有人抱怨每次发布都重新拉镜像很慢。imagePullPolicy 的默认规则是什么？拉取失败和拉取慢分别怎么处理？

**期望加分项：**
- 能准确说出 imagePullPolicy 默认规则：tag 为 `latest` 或未写 tag → Always；显式非 latest tag → IfNotPresent；never 几乎不用
- 拉取失败分层定位：镜像不存在（manifest unknown/tag 被删）→ 鉴权失败（pull access denied，imagePullSecrets 配置或密钥过期）→ 仓库不可达/限流（Harbor 有并发与带宽限制）→ DNS/网络（节点侧 docker pull 验证）
- 慢的治理：节点预热（发布前先 `crictl pull` 到目标节点）、分层复用（基础镜像层不变只推业务层）、多阶段构建瘦身（几百 MB → 几十 MB）、registry mirror（Harbor 前置缓存/或 Daemon 配置 mirror）
- tag 策略：生产禁止 latest，用 git sha / 版本号不可变 tag，避免"缓存到旧镜像"污染与回滚歧义
- 排障命令：`kubectl describe pod` 看 Events 里的 ErrImagePull 原文、`crictl pull <image>` 在节点手动验证、`crictl images` 看节点缓存

**减分项：**
- 不知道默认策略规则，以为"写了 tag 就 Always"
- 生产用 latest 发布，回滚时"同一镜像名"拉到的是新内容
- 私有仓库没有 imagePullSecrets 概念或漏配导致大规模拉取失败
- 拉取失败只猜网络，不看 describe 事件里的具体错误
- 没有镜像预热与瘦身方案，发布即卡在拉镜像

**解答：**

先记住默认规则：镜像 tag 是 `latest`（或省略 tag）→ Always 每次拉取；显式非 latest tag → IfNotPresent（节点有缓存就不拉）。生产规范第一条：禁止 latest 发布，用 `release-20260810-<git-sha>` 这种不可变 tag——否则回滚时同一镜像名对应的内容已经变了，"回滚"变成了"升级"，且节点缓存永远失效（Always 强制重拉）。这次"几分钟前还能拉、突然全拉不下来"的排查：`kubectl describe pod` 的 Events 会给出 ErrImagePull 的原文，按错误分层——`manifest unknown`：tag 拼错或 Harbor 镜像被 GC/策略清理；`pull access denied`：Pod 没配 imagePullSecrets 或 Secret 里的凭证过期（Harbor 机器人账号有效期设置）；`connection refused/timeout`：节点到仓库网络、Harbor 限流（下载并发上限）或仓库磁盘满只读。定位后对应处理：改 tag、补 `imagePullSecrets: [{name: harbor-secret}]` 到 Deployment、扩容/排查 Harbor。拉取慢的三板斧：① 节点预热——发布流水线里先对目标节点 `crictl pull` 镜像，Pod 调度过去秒级就绪（配合 DaemonSet 预热或发布前手动拉取）；② 镜像瘦身——多阶段构建把运行时镜像从 1GB 压到 100-200MB，构建产物分层复用，业务层只推增量；③ 缓存加速——Daemon 配 `registry-mirrors` 或 Harbor 前置 CDN/代理缓存。量化：镜像 300MB、节点 10 台、仓库千兆网，预热后 Pod 启动从 30s 降到 3s，发布窗口收益明显。坑：Harbor 的不可变 tag 策略（immutable）没开，同名 tag 可以被覆盖，配合 Always 策略会让"已发布的镜像"悄悄变掉；多集群共用一个 Harbor 时要规划复制（replication）与命名空间隔离，避免跨环境串镜像。

**延伸考点：** imagePullPolicy: IfNotPresent 下，节点缓存损坏（镜像层不完整）时表现是什么、怎么强制重拉？Harbor 开启 immutability 后 push 同名 tag 会怎样、灰度发布怎么配合？

---

### Q20. 20 个虚拟机老服务 3 个月迁上 K8s，怎么排优先级？最大的坑在哪？

**问题：** 你们有 20 个跑在虚拟机上的老服务要迁上 K8s，老板要求 3 个月完成且不能有大事故。迁移顺序怎么排？哪些服务不适合第一批？迁移中最大的坑是什么？最后怎么评估"迁移成功"？

**期望加分项：**
- 有明确的分批策略：无状态、低依赖、易回滚的服务先迁（Web API、无状态 worker）；有状态、强依赖、存量数据敏感的后迁（数据库、ZK/Kafka 这类最后或不上 K8s）
- 有标准迁移步骤：评估清单（配置/日志/端口/定时任务/本地磁盘/依赖）→ 容器化改造（12-factor：配置外置、日志 stdout、无本地状态）→ 探针补全 → 灰度引流（Ingress 权重/流量复制）→ 回滚预案（切回 VM）
- 能讲清高频坑：写本地磁盘丢数据（临时文件、日志落盘）、/etc/hosts 直连 IP 或硬编码依赖、JVM 不认 cgroup 被 OOMKill、宿主机 crontab 与 CronJob 双跑、端口冲突（VM 里固定端口 8080）、服务发现方式改变（IP 直连 → Service DNS）
- 探针与优雅停机必须补齐：没有 readiness 就切流 = 事故；没有 preStop 排空 = 发布掐连接
- 成本与复杂度评估：小服务迁入的运维成本 vs 收益（容器化、弹性、自愈）；评估指标：成功率、故障率对比、资源利用率提升、发布耗时下降；灰度验证用流量对比（新旧版本错误率/延迟对齐）

**减分项：**
- 一上来就迁数据库/有状态核心系统
- 不配置外置，把环境配置焊死在镜像里，环境一变全废
- 不补探针就切流量，发布即事故
- 忽略定时任务与端口冲突，迁完才发现 crontab 在双跑
- 没有回滚预案与量化验收标准，迁完不知道成没成功
- 不看运维体系（日志采集、监控告警）是否在 K8s 侧就绪

**解答：**

排优先级的原则是"先易后难、风险隔离"：第一梯队迁无状态 Web/API 服务（无本地数据、可随时重建、回滚成本低），第二梯队迁无状态 Worker/定时任务，第三梯队才碰有状态服务（数据库尽量留在 VM/托管，ZK/Kafka 评估 Operator 后再动）——3 个月窗口内能迁多少算多少，而不是贪多。每批标准流程：① 评估——出一份清单逐项打勾：配置文件（全部外置到 ConfigMap/Secret）、日志（改 stdout + 采集到 ELK/Loki）、本地磁盘（临时目录换 emptyDir 且明确"不可持久"）、固定端口（换 Service 暴露）、定时任务（宿主机 crontab 迁移到 CronJob 并关闭旧的，防止双跑）、依赖地址（IP/域名改 Service DNS 或 ConfigMap 集中管理）；② 容器化——多阶段构建 + 非 root 用户 + 基础镜像最小化，Java 应用务必确认 JVM 读 cgroup（JDK8u191+ `-XX:+UseContainerSupport`），否则容器 limit 8G、JVM 按宿主机 32G 算堆 → 启动即 OOMKilled；③ 上集群——补三探针（见 Q10）、加 request/limit（按压测水位）、配 PDB 与优雅停机（preStop + terminationGracePeriodSeconds）；④ 灰度引流——Ingress 权重从 10% 起步，观察错误率/延迟与 VM 基线对齐后再放量，同时保留一键切回 VM 的预案；⑤ 验收——发布成功率、故障数、P99 延迟对比 VM 基线（误差 10% 内合格）、资源利用率（按 request 计费前后对比）、发布耗时（分钟级 vs 原来人工半小时）。最大的坑排第一的是"本地状态没清干净"：老服务在 VM 上写日志/临时文件是常态，迁进容器后 Pod 一重建数据就没了，表现是"偶发丢数据、重启后状态错乱"这类最难查的问题——所以迁移评估清单第一条永远是"这个服务重启后会不会缺东西"。其次是定时任务双跑与 JVM 内存这类"环境差异"问题，都要在迁移文档里逐项留痕。最后提醒：K8s 迁移是"运维体系迁移"不是"部署方式迁移"，日志、监控、告警、备份、成本核算在 K8s 侧全部就绪才算迁完一个服务。

**延伸考点：** 灰度引流时新旧两套（VM 与 K8s）数据一致性问题怎么处理（比如写库双写）？迁移后发现某个服务在 K8s 上延迟比 VM 高 30%，你会从哪些环节定位（网络、CPU 限制、内核参数）？

---

### Q21. 集群某节点 NotReady 且业务 Pod 受影响，完整排查流程与恢复

**问题：** 凌晨监控告警：集群里一个节点变为 NotReady，上面跑着的业务 Pod 错误率上升、部分请求失败。节点可能是磁盘满、kubelet 挂了或与 apiserver 失联。从接到告警到恢复，你会怎么完整处理？什么情况下先隔离观察、什么情况下要驱逐节点？

**期望加分项：**
- 排查链路完整：`kubectl get nodes` 确认状态 → `kubectl describe node` 看 Conditions（Ready/OutOfDisk/MemoryPressure/DiskPressure）→ 登节点查 kubelet（systemctl status + journalctl）、容器运行时、磁盘/内存/网络
- 能区分 NotReady 成因：kubelet 异常、节点资源耗尽（磁盘满/内存压力）、节点与 apiserver 网络分区、内核 OOM/panic——成因不同处置不同
- 驱逐/隔离决策正确：先 `cordon` 禁止新调度 → 紧急时手动驱逐受影响 Pod → 稳定后再 `drain` 维护；知道 `drain` 默认驱逐规则（DaemonSet 除外、PDB 约束、--delete-emptydir-data）
- 有 PDB 意识：驱逐前检查 PodDisruptionBudget，避免 drain 卡住或强删唯一副本打断在线连接
- 恢复流程完整：修好后 `uncordon`，观察 Pod 重新调度、节点 Conditions 转 Ready
- 有复盘意识：单副本 Pod 无保护、节点监控（磁盘水位/kubelet 存活）缺失、缺 PDB 与演练

**减分项：**
- 接到告警只会"重启节点"，不查 Conditions 与 kubelet 日志的根因
- 分不清 cordon / drain / uncordon 的语义与顺序，一上来就 drain 全删
- 驱逐前不看 PDB，把唯一副本或有状态服务干掉
- 分不清"节点故障"与"Pod 故障"的处置边界，节点好了也不 uncordon
- 恢复后不复盘单点风险与监控盲区

**解答：**

按"确认状态 → 定位根因 → 决定处置 → 恢复复盘"四步走。第一步确认状态：`kubectl get nodes` 看到 NotReady 后，`kubectl describe node <node>` 看 Conditions 里 Ready=False 的 reason 与 message——KubeletNotReady、DiskPressure、OutOfDisk、MemoryPressure 各指不同方向，这是分流的关键。第二步定位根因：登节点 `systemctl status kubelet` + `journalctl -u kubelet -f` 看 kubelet 是否反复崩溃，`df -h` 看磁盘（/ 和 /var/lib/docker 或 containerd 目录最常被打满）、`free -m` 看内存、`dmesg` 看内核 OOM/panic；同时区分"节点自身故障"与"网络分区"——在节点上 curl apiserver 健康检查，通不了就是 apiserver 侧网络问题或证书过期，而不是 kubelet 死了。第三步决定处置，顺序和粒度最关键：先 `kubectl cordon <node>` 禁止新 Pod 调度进来；然后评估受影响 Pod 能否安全驱逐——先查 PDB（`kubectl get pdb -A`），有 PDB 且副本唯一时直接 `drain` 会卡在"等待 PDB 允许"（如果强删就是打断唯一副本的在线连接），正确做法是紧急时先手动驱逐关键 Pod 或等副本在其他节点拉起再处理，日常维护用 `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`；DaemonSet 的 Pod 驱逐不掉，只能登节点处理。这里最经典的坑是"不看 PDB 直接 drain"：有状态服务（etcd/MySQL）被强删，数据节点全部下线。第四步恢复与复盘：节点修好后 `kubectl uncordon <node>`，观察 Pod 重新调度、Conditions 转 Ready；然后复盘这次暴露的结构性风险——Pod 是否单副本（无多副本/反亲和保护）、节点监控（磁盘水位、kubelet 存活告警）是否完备、PDB 是否覆盖所有有状态服务并演练过。排障中最反直觉的一点：NotReady 节点的 Pod 未必全部不可用——网络分区时 Pod 还在跑但 apiserver 失联，驱逐与否要看业务是否真的受损，先止血再动刀。

**延伸考点：** 节点磁盘被容器日志打满导致 NotReady，除了清理还有哪些长效手段（日志轮转、容器存储配额、节点级磁盘监控）？drain 时 PDB 一直不满足，命令会怎样表现、怎么处理？

---

### Q22. 大促前集群容量评估与扩容预案：压测、request/limit 核算、节点扩缩容、HPA 检查

**问题：** 大促前一周，业务预计流量翻 3 倍。让你做集群容量评估和扩容预案：怎么压测取数？怎么核算 request/limit？节点要不要扩、扩多少？HPA 配置要检查什么？给出完整流程与量化方法。

**期望加分项：**
- 容量评估有方法：以历史峰值（去年大促/日常峰值）做增量测算，压测分层（单服务压测 → 全链路压测 → 集群容量压测），用压测结果反推单副本吞吐与资源需求
- request/limit 核算量化：request 按"压测稳态值 × 冗余系数（1.3-1.5）"设，limit 按峰值上限留余量；区分 CPU request（调度预留）与内存 request（OOM 保护线）的语义
- 节点扩容算得清：用"总需求 request − 当前可分配"算缺口，再折算节点数；强调集群真实可分配量要扣除系统组件与 DaemonSet 预留；云上考虑机器交付提前量/弹性节点池，自建机房提前备机
- HPA 检查到位：metrics 来源（CPU 之外接自定义指标如 QPS）、阈值与 min/max、冷却时间（扩容短缩容长防抖）、配合 PDB 与优雅排空防止缩容掐连接
- 有兜底与演练：容量打满时的降级预案（限流/熔断/静态页）、HPA→节点扩容→Pod 就绪→接流量的整链路演练
- 能想到隐性瓶颈：DB/Redis 连接数随副本数增长、镜像拉取并发、节点 IP/子网配额

**减分项：**
- 不压测直接"感觉"扩节点，扩完不知道够不够
- 不核算 request/limit，HPA 扩出来的 Pod 超卖打满节点
- 忽略系统组件与 DaemonSet 预留，算出"看着够其实不够"
- HPA 只看 CPU 不看 QPS，或冷却时间配错导致反复抖动
- 没有兜底与演练，大促当天第一次跑整条链路就崩

**解答：**

大促容量预案分五步。第一步压测取数：先单服务压测拿"单副本稳态吞吐与资源占用"（QPS/CPU/内存），再做全链路压测暴露依赖瓶颈（DB/Redis/下游），最后集群容量压测看哪些节点先满；产出物是每个服务"单副本扛多少 QPS、要多少 CPU/内存"和瓶颈清单。第二步核算 request/limit：request 按"压测稳态值 × 1.3-1.5 冗余"设——它同时决定调度预留与内存 OOM 保护线，拍脑袋设会导致扩了节点 Pod 还是 Pending 或被 OOM；limit 按峰值上限再留 20-30% 余量。关键在算集群"真实可分配量"：不能只看 Allocatable，要减去 kube-system 系统组件、DaemonSet（日志/监控/网络）的实际占用——这是"看着资源够、扩容后 Pod 仍 Pending"的头号原因。第三步节点扩容：缺口 = 各服务峰值 request 总和 − 当前可分配，新增节点数 = 缺口 ÷ 单节点可分配；云上提前确认弹性节点池与交付时间（机器创建要几十分钟到小时级），自建机房提前备机；配置 cluster-autoscaler 让 HPA 扩出来的 Pod 能落到新节点。第四步 HPA 检查：纯 CPU 扩缩容波动大、反应慢，核心业务接自定义指标（QPS/队列深度，Prometheus Adapter）更贴合容量语义；检查阈值（如 CPU 70%）、min/max 副本数、冷却时间（扩容冷却短、缩容冷却长，如缩容 5 分钟起防抖）；配 PDB 与优雅排空，否则大促结束后缩容直接掐断存量连接。第五步兜底与演练：容量打满时的降级预案（限流、熔断、切静态页），大促前完成一次全链路压测 + 一次"流量突增演练"，验证 HPA→节点扩容→Pod 就绪→流量接入整条链路。最容易忽略的隐性瓶颈：Pod 从 20 个扩到 80 个，DB/Redis 连接池按副本数核算，否则集群资源够、数据库先被连接数打满；镜像拉取并发也要评估，几千个新 Pod 同时拉镜像可能把仓库压垮。

**延伸考点：** 压测发现的瓶颈在 DB 而非应用，集群扩容解决不了，预案怎么设计（连接池核算、读写分离、限流降级）？cluster-autoscaler 扩容的节点类型与业务 Pod 亲和/污点怎么配合才能落到目标节点？

---

### Q23. 跨集群/多云 K8s 架构设计：多集群划分、容灾、与 CI/CD 和 GitOps 的配合

**问题：** 公司要上多云（自建机房 + 两个公有云），业务要具备跨集群容灾能力。让你设计多集群 K8s 架构：集群怎么划分？容灾怎么做？与 CI/CD 和 GitOps 怎么配合？给出完整架构方案与取舍。

**期望加分项：**
- 多集群划分有逻辑：按环境（dev/staging/prod）、按故障域（云厂商/机房）、按业务域（爆炸半径）三维划分；生产至少跨故障域双集群
- 容灾设计完整：应用层"多集群部署 + 全局流量调度"（GSLB/DNS 健康检查、集群级故障切流）；数据层务实取舍（云托管多可用区/跨区域复制或外部化，不在 K8s 内硬塞有状态）
- GitOps 多集群管理清晰：一个 Git 仓库按集群目录分层，Argo CD/Flux 多目标部署同一套 manifests，集群差异用 Kustomize overlay / Helm values 表达
- CI/CD 配合有节奏：CI 只产不可变制品（镜像 digest + 渲染后清单），CD 层金丝雀→灰度→跨集群推进，集群间发布错峰
- 平台层取舍正确：不用联邦 kube-apiserver（已淘汰），用独立集群 + GitOps 统一管理 + Cluster API 管理生命周期；监控日志多集群聚合（Thanos/Prometheus 联邦、日志汇聚）
- 容灾可验证：季度故障演练（停掉一个集群验证切流）、RTO/RPO 量化目标

**减分项：**
- 用联邦 API 或跨集群 Service 直连这类过时/危险方案
- 只做"多集群"不设计容灾，集群间没有流量切换能力
- GitOps 一套配置硬塞所有集群，不处理集群差异导致灰度集群配置漂移
- 忽略数据层容灾，应用切过去了数据没有或不同步
- 没有故障演练与 RTO/RPO 量化，容灾停留在架构图

**解答：**

设计先回答"为什么分"：多集群划分按三维——按环境隔离（dev/staging/prod，爆炸半径可控）、按故障域拆分（跨云/跨机房，一个云挂了另一个还能扛）、按业务域隔离（核心交易与边缘业务分开，避免互相拖垮）；生产至少要有跨故障域的两套集群，这是容灾的地基。容灾分两层：应用层靠"多集群部署 + 全局流量调度"——同一套服务在两朵云各自部署，GSLB/DNS 做集群级健康检查，集群整体故障时把流量切到存活集群（切换粒度是"集群"而非"Pod"）；数据层是真正难点——DB/Redis 用云托管的多可用区/跨区域复制能力，或干脆外部化（不放进 K8s），K8s 里只放无状态业务，这是最务实的取舍，硬塞有状态进多集群只会把复制和故障转移复杂度翻倍。CI/CD 与 GitOps 是落地关键：CI 只负责产出不可变制品（镜像 digest + 渲染后的 manifests），不做部署；部署交给 GitOps——一个 Git 仓库按 `clusters/` 目录分层管理多集群，Argo CD/Flux 用 Application 把同一套清单推送到多集群，集群间差异（域名、Ingress 证书、资源规格）用 Kustomize overlay 或 Helm values 表达，保证配置漂移被 Git 统一审计；发布节奏上先在一个集群金丝雀验证，再灰度推进到另一个集群，最后全量，集群间错峰发布，避免"所有集群同版本同事故"。平台层取舍：不要用联邦 kube-apiserver（复杂度爆炸、故障域耦合），现代做法是"独立集群自治 + GitOps 统一管理 + 监控日志聚合"（Prometheus 联邦/Thanos 跨集群、日志汇聚到统一湖），集群生命周期用 Cluster API 管理。最后必须容灾验证：季度故障演练——直接停掉一个集群，验证流量切换、数据一致性、RTO 是否达标；把 RTO（如 15 分钟）与 RPO（如依赖外部数据层实现近零丢失）写成量化目标。没有演练的容灾架构只是架构图，第一次实战演练才会暴露"切换脚本半年没更新、权限过期、数据不同步"这些真实问题。

**延伸考点：** 跨集群切流时会话一致性怎么处理（无状态化 vs 全局会话存储）？多集群场景下发布如何保证"灰度→全量"的进度在集群间同步且可回滚（GitOps 的 sync 策略怎么设计）？

---
