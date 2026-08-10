# DevOps · Jenkins（面试题库）

本文件考察候选人在 Jenkins 上的真实落地能力：主从架构与动态 agent、自由风格与 Declarative/Scripted 流水线、共享库与 Groovy 沙箱、凭据与安全、多分支与触发策略、并行并发控制、制品归档、插件生态治理、高可用与备份、构建环境管理、测试报告集成、通知可视化与排障，以及 Jenkins 与托管 CI 的选型迁移。全部为场景化提问，不考八股文——重点看候选人能否给出量化依据、说清取舍、引用线上排障过程与事故佐证，难度从实践基础渐进到架构级设计。

---

### Q1. 单机 Jenkins 扛不住了，构建全挤在主节点上排队，怎么引入 Master/Agent 主从架构？

**问题：** 你们一台 Jenkins 单机跑了 200 多个 Job，高峰期构建排队 40 分钟，主节点还经常因为构建进程吃满内存而卡死。你要把架构改成主从，怎么设计 agent 的连接方式、标签体系和资源分配？

**期望加分项：**
- 讲清 Master 职责收敛：只做调度、SCM 轮询、Web UI、插件管理，构建一律下沉到 agent，Master 上不跑任何构建任务
- 能对比 agent 两种连接方式：SSH 方式（Master 主动连 agent 的 22 端口）vs JNLP 方式（agent 主动连 Master 的 TCP/JNLP 端口或 WebSocket），并说明 JNLP 适合跨网络/防火墙、SSH 适合内网
- 说明标签（label）的用法：按环境（java17/docker/linux-arm）打标签，流水线里用 `agent { label 'docker && linux' }` 精确选择，而不是随机挑节点
- 能给出动态 agent 方案：Kubernetes/Docker 按需弹 pod，构建结束自动回收，资源利用率从静态节点的 30% 提升到 60%+
- 有取舍说明：JNLP agent 掉线重连机制、agent 数量与队列深度的关系（队列积压触发扩容阈值）
- 有线上佐证：比如主从拆分后排队时间从 40 分钟降到 5 分钟以内

**减分项：**
- 只会说"加几台从机"，说不清 SSH/JNLP 两种连接方式的本质区别和各自适用场景
- 主节点继续跑构建，架构没实质变化
- 不知道 label 怎么用，agent 被随机分配导致环境不匹配报错
- 忽略 agent 掉线、凭证分发、Java 版本不匹配等落地细节
- 无量化数据，答不出资源利用率、排队时间的变化

**解答：**

主从架构的核心是把"调度"和"执行"分离。Master 只保留 UI、SCM 轮询、插件和调度逻辑，所有构建任务派发给 agent。连接方式两种：SSH 方式下 Master 通过 ssh 到 agent 启动 slave agent，适合内网互信环境，排查方便；JNLP 方式下 agent 进程主动向 Master 注册（TCP 端口或 WebSocket 50500），能穿透防火墙，适合跨机房/云上场景，但 agent 断线要依赖 Master 侧的标记清理。标签是调度命脉：给每个 agent 打 `java17`、`docker`、`linux-arm64` 这类 label，流水线里 `agent { label 'java17 && docker' }` 精确匹配，避免"拿到一台没有 Docker 的机器才报错"。规模再大就上动态 agent：Kubernetes 插件在构建请求到达时动态创建 Pod，构建完自动销毁，避免常驻节点的资源浪费——静态节点常驻利用率往往只有 30% 左右，动态 pod 可以把构建密度提上去，而且天然支持按队列深度扩容。实践中的坑：一是 JNLP agent 频繁掉线，要检查 Master 与 agent 的时钟偏移和网络超时；二是 agent 上的构建工具版本必须统一（容器化 agent 最省心，镜像即环境）；三是不要在 Master 上装构建依赖，否则某天有人手动在主节点触发构建会把 Master 打挂。我经历过一次主从拆分：拆分前高峰期排队 40 分钟，拆分后同类负载排队压在 5 分钟以内，且 Master 再没因为构建 OOM 过。

**延伸考点：** JNLP agent 掉线后队列里的任务会怎么处理？agent 上凭证（如 SSH key）怎么安全分发而不落盘明文？

---

### Q2. 团队既有自由风格 Job 又想上流水线，三种项目类型怎么选型？

**问题：** 你们现在全是"自由风格项目"（Freestyle），每个 Job 在 UI 里点出来的：一堆"执行 Shell"步骤、构建后动作里配了十几个插件。现在要统一 CI 规范，你会怎么推动迁移到流水线？Declarative 和 Scripted 怎么选？

**期望加分项：**
- 讲清三种类型的本质差异：Freestyle 是 UI 配置、步骤散落在页面里无法版本化；Declarative Pipeline 是结构化声明式 DSL，语法受限但可读性强；Scripted Pipeline 是完整 Groovy 代码，灵活但难维护
- 给出选型结论：默认 Declarative，需要复杂循环/动态逻辑/编程式控制流时才在 Scripted 或 Declarative 内嵌 `script {}` 块
- 说明迁移策略：Freestyle 的步骤能力 Declarative 都有对应（shell/archive/record test），按"高价值 Job 优先迁移"排序，迁移过程用"影子运行"（同一分支跑两套对比结果）降低风险
- 提到 Declarative 的优势：`options`、`post`、`agent`、`when` 都是声明式结构，天生支持重试、超时、跳过；Jenkinsfile 能进仓库做 code review 和版本化
- 有量化意识：流水线化后 Job 配置从"改 UI 无审计"变成"PR 评审"，配置漂移事件归零
- 指出 Scripted 的适用边界：复杂编排（动态 stage、矩阵生成）确实要它，但要用共享库封装防止失控

**减分项：**
- 只背"Declarative 简单、Scripted 灵活"，给不出具体取舍场景
- 认为迁移就是把 shell 复制进 pipeline 就完事，忽视结构重构
- 说不清 `script {}` 块、`@NonCPS` 这些混用场景
- 无迁移路径设计，一股脑全量切导致大面积失败
- 忽略 Jenkinsfile 进仓库带来的权限与安全变化

**解答：**

三类项目的本质差异在"配置从哪来、能否版本化"。Freestyle 配置散落在 UI 表单里，无法 review、无法审计、换人就失传；Declarative Pipeline 是声明式 DSL，把 agent/stages/options/post 结构化表达，是官方推荐的默认选择——80% 的场景（构建、测试、归档、通知）都能用它的受限语法写完；Scripted Pipeline 是完整 Groovy，能动态生成 stage、写复杂循环和分支，但维护成本高，是那 20% 复杂场景的兜底。我的默认结论是：新 Job 一律 Declarative；遇到"动态矩阵生成 stage""按运行时数据决定流程"这类需求，优先在 Declarative 里用 `script {}` 包一小段，实在不行才整体 Scripted，并把这部分逻辑收进共享库函数，避免流水线里堆 Groovy 海。迁移路径上别搞大爆炸：先挑 3-5 个高频核心 Job 试点，把 Freestyle 里的步骤一一映射（shell→sh、构建后归档→archiveArtifacts、JUnit 记录→junit、邮件→post），同时在旧 Job 旁开"影子流水线"跑相同构建对比产物一致性，确认无误再切流。迁移的隐性收益容易被忽略：Jenkinsfile 进仓库后，流水线变更走 PR 评审，能直接消灭"某人在 UI 改了个参数导致生产发布失败还没人知道"这类配置漂移事故。实践坑：把 Freestyle 的"构建后动作"想当然搬进 Declarative 会有行为差异（如 archiveArtifacts 只在 build 成功后默认执行，而 Freestyle 可以配 always），迁移时要在 `post` 里显式声明才能对齐。

**延伸考点：** Declarative 里想对两个 agent 分别执行不同构建（如编译和打包分开），怎么写？`when` 条件在 stage 里的求值时机（是在 agent 分配前还是后）？

---

### Q3. Declarative Pipeline 刚上手，agent/stages/options/post/环境变量这些块怎么组合出生产级流水线？

**问题：** 你们把 Freestyle 迁到 Declarative，但写出来的 Jenkinsfile 要么变量不知道在哪定义、要么 stage 失败后不归档产物、要么测试报告没采集到。一个生产级的 Declarative 骨架应该长什么样？

**期望加分项：**
- 能画出标准骨架：`pipeline { agent → options → environment → stages（多 stage）→ post（always/success/failure）}`
- 讲清 environment 块的作用域：pipeline 级全局、stage 级局部、`environment { }` 里引用凭据（`credentials('id')`）自动注入为环境变量
- 说明 options 常用项：`timeout(time: 1, unit: 'HOURS')`、`retry(2)`、`timestamps()`、`disableConcurrentBuilds()`、`buildDiscarder`；stage 级 options 只对该 stage 生效
- post 块按结果执行清理/归档/通知：`always` 里 archiveArtifacts + junit（保证失败也有产物），`failure` 里发通知
- 给出实践细节：`when { branch 'main' }` 做分支级跳过、`tools` 块指定 JDK/Maven、agent 的 docker 用法 `agent { docker { image 'maven:3.9' } }`
- 有量化或事故佐证：如"产物只在 success 归档导致失败构建查不到日志包"这类事故

**减分项：**
- 只会背关键字，写不出一个能跑的完整骨架
- 不知道 environment 里引凭据、`params` 的用法，变量定义混乱
- post 只写 success 不写 always/failure，失败路径无归档无通知
- options 超时、重试、并发控制一个都不会配
- 说不清 pipeline 级和 stage 级作用域的差异

**解答：**

生产级 Declarative 骨架应该是：`pipeline { agent { label 'java17' } options { timestamps(); timeout(time: 1, unit: 'HOURS'); disableConcurrentBuilds(); buildDiscarder(logRotator(numToKeepStr: '30')) } environment { MAVEN_OPTS = '-Xmx2g' SONAR_TOKEN = credentials('sonar-token') } stages { stage('checkout'){...} stage('build'){...} } post { always { archiveArtifacts artifacts: 'target/*.jar'; junit 'target/surefire-reports/*.xml'; cleanWs() } success { slackSend ... } failure { emailext ... } } }`。要点逐个说：environment 块有作用域——pipeline 级全局可用，stage 级只在该 stage 内可见；引用凭据用 `credentials('id')`，它会把用户名密码映射成 `id_USR`/`id_PSW` 两个环境变量，secret text 映射成 `id`，这一步是避免明文的关键。options 里 `timestamps()` 让日志带时间、`timeout` 防止 job 挂死占 agent、`disableConcurrentBuilds` 防止同一分支重复构建互相覆盖、`buildDiscarder` 控制留存，这些都是线上必需项。post 块最容易踩坑：很多人只在 success 里归档，结果是构建失败时拿不到任何产物和测试报告——正确做法是 `always` 里归档 + 采集 JUnit（junit 插件能收集失败/跳过明细），`failure` 里只做通知和告警，`cleanWs()` 放在 always 末尾清理工作区。agent 也可以直接容器化：`agent { docker { image 'maven:3.9-eclipse-temurin-17' } }`，环境即镜像，杜绝"这台 agent 有 JDK8 那台没有"的漂移。真实事故：某团队产物只在 success 归档，一次单测失败后线上排查需要当时的构建包，日志里什么都没有，只能让开发重跑构建——这就是 post 块没设计好的代价。

**延伸考点：** `environment` 里定义的值能覆盖默认值吗？`options` 里的 `buildDiscarder` 和全局系统配置里的策略是什么关系？stage 级 `agent` 与 pipeline 级 `agent` 冲突时怎么解析？

---

### Q4. Scripted Pipeline 里 Groovy 沙箱到底拦什么？`@NonCPS` 什么时候必须用？

**问题：** 你们有个 Scripted Pipeline 用了 `sum = [1,2,3].collect { it * 2 }` 这种集合操作，跑起来直接报"NotSerializableException"或者沙箱拒绝执行。为什么 Groovy 在 Jenkins 里这么"难用"？CPS 和沙箱的坑在哪？

**期望加分项：**
- 讲清 CPS（Continuation-Passing Style）的本质：Jenkins 把 Groovy 脚本变换成可暂停/可恢复的状态机，才能支撑"构建暂停、agent 重启后续跑"
- 说明沙箱规则：未批准的方法（如反射、`System.exit`、读写任意文件、`new URL().text`）在 sandbox 模式被拦截，需要管理员审批或在可信库中运行
- 能解释 `@NonCPS` 的适用场景：对大型集合做不可暂停的纯函数计算（遍历、转换、聚合）时标注 `@NonCPS`，避免 CPS 序列化每个迭代步骤导致的性能灾难；但标注后方法内不能调用流水线步骤
- 给出性能量化：CPS 状态下 `list.collect{}` 慢几十倍、且大列表 OOM/序列化爆炸，`@NonCPS` 后回到毫秒级
- 说明"引擎外执行"的替代方案：复杂逻辑放共享库的 `@NonCPS` 方法，或干脆用 `sh` 脚本/写临时文件用外部程序算，避免 Groovy 全家桶
- 有排障佐证：比如 `readFile` 在 sandbox 的限制、`java.lang.SecurityException` 的典型报错

**减分项：**
- 只背"有沙箱"，说不清具体哪些操作被拦、怎么绕过（审批 or 可信库）
- 不知道 `@NonCPS` 是什么、什么时候用，遇到序列化异常只会改来改去
- 在 `@NonCPS` 方法里调用 `sh`/`echo` 等步骤导致"只能在 CPS 里调用"报错
- 把复杂算法写在流水线主体里导致执行超慢，无优化意识
- 无实际报错/排障佐证

**解答：**

Scripted Pipeline 被 Jenkins 用 CPS 变换重写：Groovy 源码被编译成"可挂起/可恢复"的控制流状态机，这样 `sh` 这类耗时步骤执行时能挂起整个流程、agent 掉线后可以从上次中断点恢复，代价是性能——所有对象经过序列化检查，集合的迭代也是逐元素暂停。Groovy 沙箱是第二层限制：默认 sandbox 模式下只放行白名单方法，反射、`System.exit()`、直接写任意文件、`new URL().text` 抓外部资源这些会被 `SecurityException` 拦截，解决方式有两条：脚本里调用受限方法需要管理员在"进程内脚本审批"里批准；或者把逻辑放共享库（共享库在可信环境中运行，不受 sandbox 限制，但依然受 CPS 影响）。`@NonCPS` 是给"纯计算"用的：方法标注后不再做 CPS 变换，对大数据集做 `collect`/`groupBy`/`sum` 这类操作直接跑原生 JVM，快几十倍，也不会有序列化开销——线上典型坑是没标注时遍历几万条记录，构建从 30 秒变 10 分钟甚至 `OutOfMemoryError`。但 `@NonCPS` 方法里绝对不能调 `sh`、`echo` 这类流水线步骤，否则报"NonCPS method called from CPS context"之类的错，正确模式是：方法内只做纯数据变换，返回结果给 CPS 侧再执行步骤。工程建议：别在流水线主体里写复杂算法，把纯逻辑封装成共享库里的 `@NonCPS` 方法并写单测；更重的处理直接 `sh` 调外部工具（python/jq），Groovy 只做编排不做事。我自己定位过一次：构建阶段 `inventory.collect{}` 在 CPS 下跑 2000 条数据用了 6 分钟，标 `@NonCPS` 后 3 秒。

**延伸考点：** 共享库里的方法默认受沙箱限制吗？在 sandbox 模式下调用 `readFile` 读取 agent 工作区之外的路径会怎样？

---

### Q5. 几十个 Job 都有重复的构建步骤，怎么用共享库（Shared Library）收敛？

**问题：** 你们 30 个流水线 Job 里都复制粘贴了同一段"构建 Docker 镜像并推送、打 tag、发通知"的代码，现在改一次推送仓库地址要改 30 个文件。共享库怎么建？目录结构、版本管理、加载方式怎么设计？

**期望加分项：**
- 能默写共享库标准目录：`vars/`（全局变量/函数）、`src/`（Groovy 类）、`resources/`（模板资源），`vars/*.txt` 是帮助文档
- 讲清加载方式：`@Library('my-shared-lib')` 注解 vs 系统配置里的 Global Pipeline Libraries（global 默认对所有 Job 生效），folder-level 库只对特定文件夹的 Job 生效
- 版本管理要点：库本身是 git 仓库，用分支/tag 管理，Jenkinsfile 里 `@Library('lib@1.2.3')` 固定版本而不是默认分支，避免共享库变更引起全局炸
- 说明 vars 里每个 `.groovy` 文件就是一个可调用全局函数（如 `vars/buildDockerImage.groovy` 暴露 `buildDockerImage(...)`），且默认是 CPS 方法
- 强调测试：共享库用 Jenkins 官方库测试框架（JenkinsPipelineUnit）写单测，变更走 PR
- 有治理意识：库函数参数化设计、错误处理（`error()` 终止）、把环境差异收敛为参数
- 有量化佐证：收敛后一次改动从改 30 个文件变成改 1 个库 + 版本升级

**减分项：**
- 只会说"把公共代码抽出来"，说不出目录结构、加载方式
- 不知道 `@Library` 注解怎么指定版本、global vs folder 的区别
- 共享库变更走默认分支，上线即全局爆炸，无版本化意识
- 没有测试与评审机制，库成为新的"黑盒不可维护点"
- 把复杂逻辑全部塞进共享库但无参数化、无文档，反而更难用

**解答：**

共享库本质是"流水线的代码复用层"，标准结构：`src/` 放 Groovy 类（可写单测的纯逻辑）、`vars/` 放全局变量和函数（每个 `.groovy` 文件对应一个可在流水线直接调用的函数，配同名 `.txt` 写帮助文档）、`resources/` 放模板文件。调用方式两种：一是 Jenkinsfile 顶部 `@Library('devops-lib@v2.1.0') _` 显式加载并锁版本；二是在"Global Pipeline Libraries"系统配置里注册为 global，对所有 Job 隐式可用——global 方便但危险，库一改所有 Job 行为都变，所以上线规范建议：库自己用 git 管理、用 tag 发版，Jenkinsfile 里固定版本号 `@Library('lib@2.1.0')`，要升级就改 Jenkinsfile 走 PR，形成"库版本 + 消费者版本"的双向可追溯。vars 函数默认在 CPS 下执行，纯计算部分记得 `@NonCPS` 或把逻辑下沉到 `src/` 的普通类。实践中还涉及敏感信息：库函数接收参数时，凭据 ID 应该由 Jenkinsfile 传入而不是硬编码在库里。测试必不可少——用 jenkins-pipeline-unit 对 vars 函数做单测，库本身 CI 化，发布新版本前跑通所有测试和一条"试用 Job"；我所在团队把这个做成规范：库改动必须 PR 评审 + 至少一个试用 Job 全绿才发 tag。落地收益量化：当时把"镜像构建推送 + tag + 通知"收敛成一个 `buildDockerImage()` 函数后，一次推送仓库迁移从改 30 个文件降为改 1 个文件 + 发一个版本，且误改导致的发布事故归零。

**延伸考点：** 共享库的 `vars` 函数里怎么拿到当前构建的参数、环境变量（如 `env.BRANCH_NAME`）？global 库和 folder 库同时存在时优先级怎么定？

---

### Q6. Jenkinsfile 里出现明文密码、仓库里躺着一堆密钥，凭据管理怎么设计？

**问题：** 排查时发现你们仓库里有明文数据库密码、Jenkinsfile 里写死了 `password = 'xxx'`、Job 配置里直接填了 SSH 私钥。怎么系统性治理？Jenkins 凭据体系有哪些类型、怎么用才不会踩坑？

**期望加分项：**
- 讲清凭据类型：Username with password、SSH Username with private key、Secret text、Secret file、Certificate，各对应什么场景
- 说明标准用法：凭据存在 Jenkins 凭据存储（加密存储于 `credentials.xml`，用 `secret` 密钥加密），流水线里用 `credentials('id')` 或 `withCredentials([usernamePassword(...)])` 注入，绝不写入日志和产物
- 讲清与 Vault 的集成：凭据托管在 HashiCorp Vault，Jenkins 用 vault 插件按需拉取，实现"凭据不落 Jenkins 磁盘"、动态轮换
- 有安全细节：`withCredentials` 里开启 `maskValue` 脱敏日志、防止脚本 `echo` 打印；SCM 提交前用 git-secrets/scan 工具扫描历史明文
- 讲权限模型：凭据按 domain 分（全局/文件夹级）、按 Job 授权使用，避免"一个团队能看所有团队的密钥"
- 有事故佐证：明文密码泄露到构建日志/私有仓库被爬取这类事件及整改

**减分项：**
- 只知道"把密码存到凭据里"，说不出类型区分和注入方式
- 凭据 ID 硬编码在共享库或全局，换环境就失效
- 不知道 `withCredentials` 的作用域和脱敏机制，日志里照样打印密码
- 对 Vault 等外部凭据管理只有概念，说不清 Jenkins 侧怎么集成
- 无权限隔离与轮换意识，一把全局密钥通行

**解答：**

治理分三层：第一层是"禁止明文"，从仓库和历史里清掉硬编码密钥（用 git-secrets 在 commit hook 拦、用历史扫描脚本清除并提醒轮换）；第二层是"凭据入库"，按类型选择：服务账号密码/API token 用 Username with password，SSH 部署用 SSH key 类型，通用 token 用 Secret text，配置文件用 Secret file，证书走 Certificate 类型。流水线里标准用法是 `withCredentials([usernamePassword(credentialsId: 'nexus-auth', usernameVariable: 'NEXUS_USR', passwordVariable: 'NEXUS_PSW')]) { sh './push.sh' }`——凭据只在闭包内可用，配合 Mask passwords 插件自动对日志脱敏；Declarative 里更简单的 `environment { PWD = credentials('nexus-auth') }` 会把密码映射为环境变量。注意把凭据 ID 作为流水线参数/环境注入，而不是写死在共享库里，这样 dev/prod 环境切凭据不需要改代码。第三层是"外部化与隔离"：团队规模大或安全要求高时，把凭据迁到 HashiCorp Vault，Jenkins 用 vault 插件在构建时动态拉取 secret 并注入，实现凭据不落 Jenkins 磁盘、支持自动轮换和审计，Jenkins 侧只留"拉取权限"而非"密码本体"。权限上还要按 folder 划分凭据作用域，team A 的 Job 看不到 team B 的凭据，避免"一把全局 key 所有人可见"。真实案例：某团队把 MySQL 密码写进 pipeline 并 echo 到日志，日志被采集进 ELK 后又被公开索引，最终密码被外部扫描到——治理后所有敏感项收敛到凭据存储，日志扫描确认零明文泄漏。

**延伸考点：** 从 Vault 拉取的动态凭据在 `post` 块里还可用吗？凭据 ID 变更导致全线 Job 失败时，怎么做到"凭据旋转不破构建"？

---

### Q7. 每个分支一个构建、PR 也要验证，怎么用多分支流水线管起来？

**问题：** 你们现在每个分支要手动建一个 Job 才能构建，feature 分支合进 main 前没有自动验证，线上出过"本地能过、合并就炸"的事故。多分支流水线（Multibranch Pipeline）怎么解决？分支发现、PR 构建、按分支差异化怎么配？

**期望加分项：**
- 讲清多分支的本质：一个 Multibranch Pipeline 项目自动扫描仓库所有分支（branch 发现策略：只发现 main/PR/所有分支），每个分支自动生成对应子 Job，行为由该分支上的 Jenkinsfile 决定
- 能说明分支发现策略的配置：Git 的 "Discover branches"（所有分支/仅 PR/仅 main+PR）、"Discover pull requests from origin/forks"，以及定期扫描频率（如每 5 分钟）与 webhook 触发
- 讲清"按分支差异化构建"：Jenkinsfile 内用 `env.BRANCH_NAME` + `when { branch 'main' }` 控制——feature 只跑 lint/单测，main 跑完整流水线并部署
- 说明 PR 构建的两种形态：pipeline 直接在 PR 上跑（检出合并结果 `refs/pull/1/merge`）或与 GitHub/GitLab 插件的 build status 回写集成，实现"PR 必须全绿才能合并"
- 有治理意识：PR 里防止恶意 Jenkinsfile（安全涉及 pipeline 权限）；分支过多时清理长期不活跃分支的子 Job
- 有量化佐证：接入后"合入即炸"事故减少、PR 平均验证时长数据

**减分项：**
- 只会说"它能自动建 Job"，说不清分支发现策略和扫描机制
- 不知道 PR 构建检出的到底是 PR 源分支还是合并结果，验证失真
- 所有分支跑同一套完整流水线，资源浪费且 feature 分支也会触发部署
- 忽略安全：PR 里的 Jenkinsfile 是未审查代码，存在被恶意利用的路径
- 无分支清理/扫描频率等运维意识

**解答：**

多分支流水线把一个仓库的所有分支纳管为一个项目：Jenkins 按分支发现策略定期扫描仓库（或收 webhook），每个发现的分支/PR 自动生成一个子任务，子任务读取该分支上的 `Jenkinsfile` 执行——关键点：每个分支跑的是"自己分支里的流水线定义"，所以主分支在演进流水线时，老分支构建的还是老逻辑，天然兼容。分支发现策略要讲清楚：Git 插件支持"只发现 main"、"所有分支"、"PR"三种维度，一般团队用"main + 所有 PR"组合，避免为每个临时分支都建 Job；扫描间隔 1-5 分钟，生产环境建议接 webhook 触发扫描而不是纯轮询。PR 构建的验证点：默认检出的是 PR 合并后的结果（`refs/pull/N/merge`），这样才能验证"合并后是否真的能通过"——这正是"单分支绿、合完就炸"事故的解法；配合 GitHub/GitLab 插件的 commit status 回写，PR 页面上直接显示构建红绿。差异化用 `when` 实现：`stage('deploy') { when { branch 'main' } ... }` 让部署只在 main 跑，feature 分支只跑 lint+单测（约 2-3 分钟），主分支完整流水线（10 分钟），这样既快又不浪费。两个坑要主动讲：一是安全——PR 的 Jenkinsfile 是陌生人提交的代码，默认策略下 PR 构建会在受信环境执行，要配置"仅对受信作者执行 PR 构建"或限制 PR 可执行的能力；二是清理——长期不活跃分支会积累大量子 Job，配合扫描时的"自动清理旧子任务"选项。我们接入后，PR 全绿自动合并成为强制门禁，"合入即炸"事故基本绝迹。

**延伸考点：** PR 构建时检出合并结果，但如果目标分支本身也同时在构建，两者产物会互相覆盖吗？多分支里的"重放（Replay）"和"重建（Rebuild）"有什么区别？

---

### Q8. 构建触发方式一堆：轮询、webhook、定时、上游触发、参数化触发，怎么选？

**问题：** 你们现在全是"定时每 5 分钟轮询一次 SCM"，一个仓库上百次轮询请求、构建延迟 5 分钟才被发现，PR 的 commit 状态回写也是延迟的。你打算改成 webhook，同时还有其他 Job 需要定时跑、需要等上游构建完成才能跑。触发体系怎么设计？

**期望加分项：**
- 讲清轮询 vs webhook 的本质差别：轮询是 Jenkins 定期问 SCM"有没有变化"（有 `pollSCM` 且需要 SCM 凭证权限），webhook 是 SCM 主动通知（GitHub webhook / GitLab system hook / Gitee 等），延迟从分钟级降到秒级
- 定时触发用 `triggers { cron('H 3 * * *') }` 配哈希散列值（`H`）避免整点风暴，说明 nightly 构建的典型用途（全量回归、依赖升级检查）
- 上游触发：`triggers { upstream(upstreamProjects: 'lib-build', threshold: hudson.model.Result.SUCCESS) }` 或流水线里 `build(job: 'deploy', wait: true)` 显式编排，讲清"被动等待"和"主动调用"两种风格
- 参数化触发：用参数（环境/版本）驱动同一 Job 多场景复用，结合 `disableConcurrentBuilds` 控制重复参数触发的并发
- 有取舍：webhook 的可靠性兜底（漏事件时保留低频轮询做兜底扫描）、事件风暴下的限流与合并触发（GitHub 的 check run 语义）
- 有量化：轮询延迟 5 分钟 → webhook 秒级触发，构建反馈循环缩短的数据

**减分项：**
- 只会说"用 webhook 快"，说不清配置链路（webhook 打到哪个 URL、带不带 token 鉴权）
- 定时 cron 写 `* * * * *` 整点风暴，不知道 `H` 哈希散列
- 分不清 upstream 触发和流水线里 `build` 步骤编排的区别
- 只讲"轮询简单"，忽视其延迟、SCM 负载与权限问题
- 没有 webhook 丢失事件的兜底方案

**解答：**

触发体系按"事件来源"分四类设计。第一类 SCM 变更：首选 webhook——GitHub/GitLab 配置 webhook 指向 Jenkins 的 `/github-webhook/` 或 `/multibranch-webhook-trigger/`，Jenkins 收到事件立刻触发对应 Job，延迟从轮询的 5 分钟降到 1-2 秒；轮询 `pollSCM` 只该做兜底（webhook 偶尔丢事件），频率拉低到每 10-15 分钟。webhook 要鉴权（配 token 或 IP 白名单），否则任何人可以伪造事件触发一堆构建。第二类定时：`triggers { cron('H 4 * * *') }`，`H` 是哈希散列，把任务分散到该小时的随机分钟，避免几十个 Job 整点同时开跑打爆资源；定时任务适合 nightly 全量回归、依赖升级检查、证书过期巡检。第三类上游依赖：两种风格——被动型 `triggers { upstream(...) }` 声明"我依赖谁，谁成功我就跑"；主动型在流水线里 `build(job: 'deploy-lib', parameters: [...], wait: true)` 显式调用并等待结果。工程上推荐主动型，因为依赖关系写在流水线里、看得见且支持失败处理，`upstream` 触发器适合"共享库被改后所有下游 Job 自动重建"这类广播场景。第四类参数化触发：同一个 Job 暴露"环境/版本/分支"参数，测试环境发布和预发发布复用一份流水线，配合 `disableConcurrentBuilds` 防止同一参数重复触发互相打架。线上经验：webhook 上线后开发反馈"push 到 PR 全绿"的时间从约 6 分钟缩到 2 分钟内，且 webhook 配了"事件丢失兜底轮询 + 失败重试"双保险，没再出现漏触发。

**延伸考点：** GitHub webhook 与多分支流水线的扫描是什么关系？一个 webhook 事件进来，怎么做到只触发受影响的那个子 Job 而不是全仓库扫描？

---

### Q9. 同一个发布 Job 要支持选环境、选版本，参数化构建怎么做才不失控？

**问题：** 你们有个"发布到测试环境"的 Job，现在要发布到哪个环境、用哪个制品、要不要跳过测试，全靠改 Jenkinsfile 或者问运维。想改成参数化构建：Choice、Boolean、Active Choice 这些参数类型分别怎么用？参数怎么进流水线？

**期望加分项：**
- 讲清参数类型：Choice 是固定选项下拉、Boolean 是开关、String 是自由输入、Active Choices 是动态参数（参数值依赖其他参数，如选完环境自动列出该环境可用制品列表）、Extended Choice 支持多选
- 参数在流水线里的读取：Declarative 里 `parameters { choice(...) booleanParam(...) }` 声明后通过 `params.xxx` 引用；参数声明必须在 pipeline 块顶部 `parameters` 段
- 强调参数默认值和校验：String 参数必填校验、非法值拦截（`when` + `error()`），防止手滑把测试环境发布参数填进生产
- 参数与并发/安全：参数化 Job 开启 `disableConcurrentBuilds`，生产相关参数 Job 限指定角色触发；参数不要直接拼进 `sh` 造成注入风险（用 `sh script: "..."` 时对参数做转义）
- 有实践细节：Active Choices 脚本的缓存问题、参数触发与手动触发的行为差异
- 有事故佐证：如"参数填错环境导致误发布"及防护措施

**减分项：**
- 只会说"加个参数"，说不出 Choice/Boolean/Active Choice 的差异和适用场景
- 不知道 `params` 的读取方式，或把参数声明写错位置
- 无默认值与校验，误操作没有拦截
- 参数直接拼接 shell 命令导致注入或引号爆炸
- 无参数化引发的并发/权限失控意识

**解答：**

参数化设计的核心是"把 Job 变成可复用模板，且错误输入成本为零"。类型选择：环境这类有限枚举用 Choice（`choice(name: 'ENV', choices: ['dev','test','prod'])`）；"是否跳过测试""是否发通知"用 Boolean（`booleanParam(name: 'SKIP_TESTS', defaultValue: false)`）；制品版本这类自由值用 String 但必须配默认值和校验；Active Choices 是进阶：用 Groovy 脚本根据其他参数动态生成选项——比如选完 `ENV`，Artifact 参数自动从制品库 API 拉出该环境可用版本列表（`activeChoiceParam` + `referencedParameters`），把"填错版本"从源头消灭。读取方式：Declarative 里声明在 pipeline 顶部的 `parameters` 块，运行中统一用 `params.ENV`、`params.SKIP_TESTS` 访问（注意 `params` 是 map，未声明参数用 `env` 读不到）；Scripted 里用 `currentBuild.rawBuild.getEnvironment()` 或在 `properties([parameters(...)])` 声明后以全局变量访问。工程防护三层：一是校验——参数化 stage 开头对 `params` 做白名单检查，非法组合直接 `error()` 中止，比如"env=prod 但 version 是 SNAPSHOT"直接拒绝；二是并发——参数化 Job 默认允许并发跑不同参数，极易出现"dev 和 prod 两个构建同时写同一份产物"，开 `disableConcurrentBuilds()` 或按参数维度串行；三是注入——String 参数拼接进 `sh` 前必须转义（用 `sh script: "echo ${params.VERSION}"` 时注意引号，最佳实践是写成参数文件或环境变量传入），否则特殊字符能逃逸出命令。真实事故：有人发布时把 ENV 下拉选成 prod，流水线没校验直接推了包，后来加了一条 `if (params.ENV == 'prod') { input '确认发布生产?' }` 人工确认门禁，此类误操作归零。

**延伸考点：** 参数化构建的值在构建队列里能看到吗？`params` 与 `env`、`currentBuild.getBuildVariables()` 三者的取值差异是什么？

---

### Q10. 测试全串行跑 40 分钟，怎么并行？并行后资源又不够、产物互相覆盖怎么办？

**问题：** 你们单测、集成测试、E2E 串行跑完要 40 分钟，测试阶段内部还是一条命令全跑。你想用 `parallel` 并行，但怕并行 job 抢资源、工作区互相覆盖。Jenkins 的并行和并发控制有哪些手段？

**期望加分项：**
- 讲清 `parallel` 在 Declarative 里的用法：`stage('test') { parallel { stage('unit') {...} stage('it') {...} } }`，并行分支各自独立 step 和 agent
- 说明三种并发粒度：stage 内部 parallel（同 agent 并发）、`agent { label 'x' }` 多 agent 并行、`node('label')` 显式并行块；`parallel` 的 `failFast true` 一票否决
- 讲并发控制三板斧：`disableConcurrentBuilds()` 防止同 Job 并发、`lock` 插件对共享资源加锁、`quietPeriod`/队列控制限制全局并发量（`org.jenkinsci.plugins.workflow` 的 numExecutors + throttle 插件）
- 说明工作区隔离：多 agent 并行天然隔离 workspace；同 agent 用 `ws('dir')` 指定不同工作区避免产物冲突
- 有量化意识：串行 40 分钟 → 三路并行 15 分钟内，同时说明并发提升受"资源瓶颈"约束不是无限扩
- 有实践佐证：并行后的 flaky 问题（并行下测试顺序/端口冲突）与资源争抢调优

**减分项：**
- 只会说"用 parallel"，说不出并行分支怎么分配 agent、failFast 怎么用
- 不知道 disableConcurrentBuilds 与 parallel 的区别（一个是构建之间、一个是构建内部）
- 并行任务共享工作区/产物互相覆盖的问题没意识
- 忽略并发上限与资源配额，盲目并行导致集群雪崩
- 无量化对比，说不清并行带来的实际收益和瓶颈

**解答：**

并行设计分"构建内并行"和"构建间并发"两个维度。构建内：Declarative 里 `stage('test') { parallel { stage('unit') { agent { label 'test' } steps { sh 'mvn test' } } stage('e2e') { agent { label 'test' } steps { sh 'npm run e2e' } } } }`——每个并行分支可以是独立 step，也可以带自己的 agent；`failFast true` 表示任一分支失败立即中止其余分支，适合"失败没必要等"的场景。注意并行分支默认共享主 stage 的 agent 和工作区，跨分支写同一目录会互相覆盖，要么每个分支显式指定不同 agent/`ws('unit-ws')`，要么让每路自己 checkout。构建间：`disableConcurrentBuilds()` 是"同一 Job 同一时刻只允许一个构建"，适合有共享副作用的发布类 Job；`lock('resource-name')` 插件对"部署到同一台机器""操作同一个制品库目录"这类跨 Job 共享资源加互斥锁；全局层面用 throttle-concurrents 插件或控制 agent executor 数（numExecutors）限定并发上限，防止队列风暴。收益和瓶颈要量化：当时把"单测（18 分钟）→ 集成测试（12 分钟）→ E2E（10 分钟）"串行 40 分钟改造成三路并行（各占独立 agent），总时长压到 15 分钟，但再想压就要扩测试 agent 资源——因为 CPU 并行度是硬上限，盲目加并行分支只会把每个分支都拖慢。实践中的坑：并行后出现 E2E 偶发失败，排查发现是两路测试共享了同一端口和测试数据库，用 `lock` 对端口资源加锁 + 每路独立测试库解决；还有一次并行分支多开到把 8 台 agent 全部占满，其他团队的 Job 排队两小时，后来加了全局 throttle 才算完。

**延伸考点：** parallel 分支里某个分支用了 `when` 条件跳过，会影响整体并行结构吗？同一 Job 的并行构建与 `disableConcurrentBuilds()` 同时存在时行为如何？

---

### Q11. 构建产物怎么归档和追踪？"构建 42 的产物怎么被谁用了"查得到吗？

**问题：** 你们构建产物从不归档，部署脚本都是"从工作区现拿"，结果：构建被清理后包没了、线上跑的版本不知道是哪个构建产出的、还有一次部署脚本拿到的是另一个 Job 的产物。构建产物管理应该怎么设计？

**期望加分项：**
- 讲清 archiveArtifacts 的用法与局限：`archiveArtifacts 'target/*.jar'` 归档到 Jenkins 存储（`JENKINS_HOME/jobs/<job>/builds/<n>/archive`），有留存策略；不适合当制品库（不可变、无版本策略），大制品要引外部制品库（Nexus/Artifactory）
- 说明 fingerprint（指纹）的价值：`fingerprint 'target/*.jar'` 基于 MD5 建立"产物 ↔ 构建 ↔ 使用者"的关联，能回答"哪个构建被谁用了""这个 jar 来自哪个构建"
- 制品命名与元数据：制品名带 build 号/commit（如 `app-1.2.3-<sha>-42.jar`），`build` 步骤传参 + `FINGERPRINT` 生成唯一标识
- 讲清"归档"与"制品库"的分工：Jenkins 归档用于短生命周期调试留存，生产制品必须推制品库（maven deploy / docker push / nexus 上传），并记录完整溯源链（SBOM、commit、构建环境）
- 有治理意识：归档留存策略（buildDiscarder + artifact 留存天数）、磁盘清理（定期删 archive 目录）、大制品告警
- 有排障佐证：如"构建目录被清理导致产物丢失"、"拿错 Job 产物部署"事故

**减分项：**
- 只会背 `archiveArtifacts` 命令，说不出它存哪、什么时候清理、适不适合当制品库
- 不知道 fingerprint 的存在，答不出"产物被谁用了"怎么查
- 把 Jenkins 归档当制品库用，制品没有不可变与版本策略
- 无留存策略，JENKINS_HOME 磁盘被 archive 撑爆
- 产物命名无 commit/构建号关联，无法溯源

**解答：**

产物管理要拆两个层次。第一层是"短生命周期调试留存"：`archiveArtifacts 'target/*.jar'` 把构建产物收进 `JENKINS_HOME/jobs/<job>/builds/<n>/archive`，与构建记录绑定、随构建一起看，配合 `buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))` 控制留存——注意 artifact 留存可以单独设置，避免"构建记录留 30 个但磁盘被 30 份大 jar 撑爆"。这里的关键认知是：Jenkins 归档不是制品库，它没有不可变性保障、没有跨环境拉取 API、没有版本策略，生产发布绝不能从"工作区现拿"或"从 archive 目录拷"，必须走制品库。第二层是"生产制品全链路溯源"：构建完成后把制品推送到 Nexus/Artifactory（`mvn deploy` 或 HTTP 上传 API），制品坐标带完整元数据（groupId/artifactId/version-buildNumber-commitSha），部署脚本从制品库按坐标拉取并记录 SHA-256，做到"线上版本 → 制品坐标 → 构建记录 → git commit"四段可查。中间环节用 fingerprint 补"使用关系"：`fingerprint 'target/*.jar'` 基于内容 MD5 建索引，打开任一构建页能点开指纹看"这个产物被哪些 Job/构建使用过"——排查"线上跑的 jar 是谁构建的、哪个 Job 用了我的产物"这类问题是直接答案。事故举例：某团队部署脚本从 workspace 取包，Jenkins 磁盘清理把旧构建目录删了，回滚时找不到旧版本包，被迫重新构建旧 commit（时间戳不同、产物不完全一致）；整改后所有环境统一从制品库按坐标拉取并留存全部历史版本，回滚从"碰运气"变成"点一个版本号"。

**延伸考点：** fingerprint 的匹配粒度（内容级 vs 文件级）怎么影响使用关系分析？制品推 Nexus 后，Jenkins 怎么知道制品上传成功并回写构建记录？

---

### Q12. 插件装了一百多个、升级就炸、功能冲突，插件生态怎么治理？

**问题：** 你们的 Jenkins 装了 140 多个插件，上个月升级了一个插件导致全局构建全挂，还有两个插件功能重叠（比如两个 Docker 相关插件都在管构建）。插件治理该怎么做？常用插件清单、冲突排查、升级策略是什么？

**期望加分项：**
- 能列出核心常用插件并按功能分组：SCM（Git/Bitbucket/GitHub）、流水线（Pipeline、Git、Docker、Kubernetes、Shared Library 相关）、质量（JUnit、HTML Publisher、SonarQube scanner）、通知（Email Extension、Slack/企业微信）、凭据安全（Credentials、Mask Passwords、Vault）、系统（Timestamper、Workspace Cleanup、Configuration as Code）
- 讲清升级治理流程：先看升级兼容性（插件官网/Release notes + Jenkins 版本要求）、先在 staging 实例验证、升级顺序按"核心依赖优先"、升级后全量回归关键 Job；用插件管理 API（`/pluginManager/api/json`）记录版本基线
- 冲突排查方法：看启动日志的插件加载错误、`PluginManager` 的兼容性报告、用 `dependency` 视图找传递依赖冲突；典型如两个插件各自捆绑不同版本的同一依赖
- 讲清"最小化安装"原则：每个插件都是攻击面和性能负担（启动慢、内存涨），定期清理不用插件
- 有量化：插件数从 140 降到 60，启动时间从 5 分钟降到 2 分钟、构建稳定性提升；或升级事故的处理复盘
- 提到自动化：用插件安装清单（requirements.txt 风格或 Configuration as Code）做插件版本基线管理

**减分项：**
- 只知道"少装插件"，说不出常用插件清单和分组
- 升级无验证直接上生产，出事才回滚
- 不会排查插件冲突（启动日志、依赖视图都不提）
- 忽视插件是安全攻击面，无最小化与更新治理
- 无版本基线概念，插件版本漂移导致 staging 和 prod 行为不一致

**解答：**

插件治理的本质是"把插件当成有版本、有依赖、有风险的软件资产管理"。第一步收敛清单：按能力分组维护——SCM 类（Git、GitHub、GitLab）、Pipeline 类（Pipeline、Docker Pipeline、Kubernetes、Pipeline Utility Steps）、质量报告（JUnit、HTML Publisher、SonarQube）、通知（Email Extension、Slack/企业微信）、安全（Credentials、Mask Passwords、Vault、Configuration as Code）。凡是两个插件功能重叠（如旧版 Docker 插件与 Docker Pipeline 插件、多个 SCM 供应商插件），定一个标准、卸载另一个；不用的插件直接卸——每个插件都拖慢启动、占内存、扩大攻击面，当时我们把 140 个收到 60 个，启动时间从 5 分钟降到 2 分钟。第二步是版本基线：用插件管理 API 或 Configuration as Code 把插件清单 + 版本号固化成文件进 git，新环境一键复现同版本环境，杜绝"staging 能过、prod 插件版本漂移导致行为不一致"。第三步是升级治理：先看 Release notes 和兼容性矩阵（对 Jenkins 版本的要求），在独立 staging 实例升级 + 跑一条全流程冒烟 Job（覆盖 checkout、构建、测试、归档、通知），确认后滚动升级生产，核心依赖（Pipeline 全家桶、Credentials）优先、跟随升级包一起验证；升级顺序错了极易出现"升级 A 插件连带升级了 B 插件"的连锁事故。第四步是冲突排查方法：启动时看日志里的 `Plugin.* failed to load`、打开"Manage Jenkins → Plugin Manager → 已安装插件"里的兼容性警告、用依赖树找两个插件捆绑了同一 jar 的不同版本，最彻底的隔离方案是不同 Job 群拆成不同 Jenkins 实例。复盘一次事故：升级 Credentials Binding 插件时连带升级了 Pipeline 核心，部分 Job 报 `WorkflowScript` 类错误，靠版本基线文件秒级回滚到旧组合。

**延伸考点：** 插件升级后 Jenkins 重启失败怎么恢复（有可用修复路径吗）？Configuration as Code 怎么管理插件的非默认配置？

---

### Q13. 流水线是"可执行代码"，安全上要防什么？Groovy 沙箱、SCM 权限、agent 隔离怎么配？

**问题：** 你们允许"非管理员也能创建 Job 和写流水线"，有人写的 Jenkinsfile 里调了 `new URL().text` 抓外网数据、还有人往构建脚本里塞了删除 agent 目录的代码。Jenkins 的权限模型和流水线安全边界是什么？供应链攻击怎么防？

**期望加分项：**
- 讲清 Jenkins 权限模型：矩阵授权策略（Matrix-based security）按角色分权（admin/developer/viewer），Job 级权限（build/read/configure）、凭据按 folder 授权；"匿名用户只读、系统管理员全权"是底线
- 说明 Groovy 沙箱的边界：sandbox 模式拦截未批准方法（反射、`System.exit`、读任意文件、外网请求），管理员审批受控；共享库和系统脚本不受沙箱限制，所以要管好"谁能改共享库"
- 讲 agent 隔离：agent 上运行的是不受信任的构建代码，多租户必须用容器/K8s pod 隔离，禁止共享 agent 跑不同团队的 Job；agent 权限最小化（不挂管理凭据、工作区不共享）
- 供应链风险：Jenkinsfile/共享库/构建依赖（Maven/Gradle/npm）都可能被投毒，要求 SCM 强制 PR 评审、构建依赖锁版本 + 校验和、插件来源可信（官方 update center）
- 有线上佐证：未授权用户通过 Job 配置执行系统命令的漏洞、`SECURITY-xxxx` 类漏洞修复实践
- 能给出落地配置清单：SSRF 防护（禁用外网请求或走代理白名单）、构建脚本里禁止 `sudo`、删除无主 agent

**减分项：**
- 只会背"开启安全、设个账号密码"，说不出矩阵授权怎么按角色分
- 不知道沙箱拦什么、怎么绕过（审批/可信库），也不知道"谁能改共享库"意味着什么
- 多租户共用 agent 无隔离意识，认为"反正都是自己人"
- 忽视供应链：Jenkinsfile 从不受信 fork 执行、依赖不加锁
- 无具体漏洞/事故案例支撑

**解答：**

Jenkins 安全的边界要从四个面看。一、访问与授权：开启"系统管理 → 安全 → 全局安全配置"里的访问控制，用矩阵授权策略——admin 全权、developer 只能创建/配置自己文件夹下的 Job 和 view、viewer 只读；再配合"项目级授权"和 folder 权限，做到 team A 的人看不到、改不了 team B 的 Job 与凭据。匿名用户默认全部 deny，是安全基线。二、执行沙箱：Jenkinsfile 默认跑在 Groovy sandbox 里，反射、`System.exit()`、读写工作区外文件、`new URL().text` 这类外网访问都会被拦，需要管理员在"进程内脚本审批"里显式放行；共享库在可信区执行、不受沙箱约束，所以"谁能改共享库"等价于"谁能执行任意代码"——共享库必须走独立 git 仓库 + PR 评审 + 仅管理员合并。三、agent 隔离：agent 上的构建代码本质不可信，多团队共用一台 agent 等于共享执行环境，一旦有人在构建脚本里写 `rm -rf` 或偷读别的 Job 的工作区就出事；正确做法是 Docker agent 或 K8s pod 逐构建隔离，agent 上不挂生产凭据（凭据通过流水线按需注入）、不共享 workspace、构建用户权限最小化。四、供应链：攻击链不止 Jenkinsfile——共享库依赖、Maven/Gradle/npm 依赖、乃至插件本身都可能被投毒；对应防线：所有 Jenkinsfile/库变更强制 PR 评审、依赖锁文件 + 校验和、只从官方 update center 装插件并追踪安全通告（官方安全通告页每周刷）。真实案例：某公司没开沙箱审批，开发者顺手在 pipeline 里调 `new URL('http://169.254.169.254/')` 抓云 metadata，差点把云厂商的临时凭据打出来——这种 SSRF 类问题靠沙箱白名单 + 出网代理管控双保险挡掉。

**延伸考点：** 非受信用户的 PR 构建在什么情况下会执行 Jenkinsfile？`withCredentials` 里的凭据在并行分支里泄漏给其他分支的风险怎么控制？

---

### Q14. 团队坚持"改配置都在 UI 点"，Jenkinsfile 进仓库值不值？怎么迁？

**问题：** 你们 Jenkins 上 90% 是 UI 里手工配的 Job，最近一次线上事故是"某人在 UI 改了发布 Job 的脚本"导致的。你想推动配置即代码：Jenkinsfile 进仓库 vs 留在 UI，各自的取舍是什么？迁移怎么做不翻车？

**期望加分项：**
- 讲清"配置即代码"（Configuration as Code，CasC）两层含义：流水线定义进仓库（Jenkinsfile）+ 系统/Job 配置用 CasC 插件（jenkins.yaml）或 Job DSL 声明式生成
- 说清取舍：Jenkinsfile 进仓库的价值是版本化、可评审、可回滚、分支差异化（每个分支有自己的流水线）；UI 配置的好处是上手快、改完即生效，但不可审计、不可追溯、依赖"会点 UI 的那个人"
- 迁移策略：分阶段——新建 Job 一律 Jenkinsfile；存量高价值 Job 用"影子运行"迁移；系统配置（全局工具、凭据、插件）用 CasC 固化
- 讲清 CasC 的边界：Jenkins 官方文档明确"系统级配置"（安全、工具、插件）适合 CasC，Job 级建议 Jenkinsfile 或 Job DSL；CasC 不覆盖全部配置项、不可回滚（覆盖式写入）
- 有治理：Jenkinsfile 模板化（共享库提供标准阶段）、配置变更走 PR、迁移期间双跑对比
- 有量化佐证：迁移后"改配置无审计"事故归零、新 Job 上线时间缩短

**减分项：**
- 只会喊"配置即代码好"，说不出具体哪些配置适合进代码、哪些留在 UI
- 不知道 CasC 插件的存在和它的适用边界（系统级 vs Job 级）
- 迁移一把梭，存量 Job 全量切换导致大面积失败
- 忽视 Jenkinsfile 进仓库后对 SCM 权限、PR 审查流程的新要求
- 无事故或收益数据佐证

**解答：**

"配置即代码"要拆成两层分别论证。第一层是流水线定义（Jenkinsfile）进仓库：收益是版本化 + 可评审 + 可回滚 + 分支差异化——每个分支跑自己分支的流水线版本，改流水线走 PR，出问题 git revert 即回滚，换人不失传；代价是要求团队会写 pipeline、SCM 要有权限与评审流程。第二层是系统级配置用 CasC（Configuration as Code）插件：把全局安全、工具链（JDK/Maven 路径）、插件、系统消息固化成 `jenkins.yaml` 进 git，新实例从 yaml 一键复现，环境漂移和"手滑点错配置"从根本上消失；但 CasC 有明确边界——它覆盖的是系统级配置，Job 级配置官方建议 Jenkinsfile 或 Job DSL，且 CasC 是覆盖式写入、改错了不能自动回滚，所以 yaml 本身必须走评审和版本管理。迁移路径分三步：第一步新 Job 从创建起一律 Jenkinsfile（存量不动），把"UI 建 Job"在团队规范里禁掉；第二步存量高价值 Job（发布类、核心 CI）逐个迁移，用"影子运行"——Jenkinsfile 版 Job 与旧 UI Job 并行跑同一分支，对比构建产物与测试结果一致后再停旧的；第三步系统配置 CasC 化，注意先备份 `JENKINS_HOME/config.xml` 和现有配置再试点。实践坑：CasC 加载失败的"配置被覆盖为空"问题——上线前要在 staging 验证 `jenkins.yaml` 完整可解析，并保留一个已知良好的备份；还有 Jenkinsfile 化后，原本"运维在 UI 里偷偷改参数救火"的路径没了，要配套建立"参数化 + 审批门禁"的能力，否则运维会抵触。我们团队迁移后，配置相关事故（改 UI 导致发布失败）归零，新服务接入 CI 从"排期改 UI"变成"合并一个 Jenkinsfile PR"，时间从 1 天缩到 2 小时。

**延伸考点：** CasC 的 yaml 配置与 UI 修改同时存在时，以谁为准？Jenkinsfile 化之后，紧急修复线上流水线（生产事故）的流程应该怎么设计？

---

### Q15. Jenkins 挂了一次，构建记录全没、恢复花了 4 小时，高可用和备份怎么做？

**问题：** 你们 Jenkins 单机跑了好几年，上周磁盘坏了，JENKINS_HOME 里几百个 Job 配置、构建历史、凭据全没了，从备份恢复花了 4 小时还丢了一天的数据。Jenkins 的高可用方案和备份策略应该怎么设计？

**期望加分项：**
- 讲清 Jenkins 高可用的本质：Jenkins 是有状态应用（JENKINS_HOME 是单一事实源），官方无原生集群，HA 的通用思路是"无状态化"——把构建执行下沉到外部 agent、把元数据（Job 配置/流水线定义）全部代码化，Jenkins 本体变成"可随时重建的薄壳"
- 备份策略：JENKINS_HOME 定期备份（Job 配置 `jobs/*/config.xml`、凭据 `credentials.xml` + secret 目录、插件列表），用 thinBackup 插件做增量备份并验证恢复流程（restore 演练）
- 讲清哪些数据"不必备份"：构建产物（该进制品库）、工作区（可重建）、构建记录中的大数据（可舍弃），真正要保的是配置 + 凭据 + 历史元数据
- 说明迁移路径：单机 → 外部制品库 + 代码化配置 → 任一节点可重建；多实例前端负载均衡 + 共享外部存储（NFS/对象存储）的局限（并发写冲突、插件状态不一致）
- 恢复演练：定期"从备份重建新实例"演练，量化 RTO/RPO（如 RTO 1 小时、RPO 15 分钟）
- 有事故佐证：磁盘故障、误删 JENKINS_HOME 后的恢复过程

**减分项：**
- 只会说"加个集群"，不知道 Jenkins 官方没有原生集群模式
- 备份只备份 JENKINS_HOME 目录，不知道凭据加密密钥（`secrets/`、`master.key`）必须一起备份
- 不区分"该备份的数据"和"不该备份的数据"，把工作区和产物都塞进备份
- 从未做过恢复演练，"备份是有的，但从没验证过能不能恢复"
- 无 RTO/RPO 意识，恢复时间与数据丢失量没有量化

**解答：**

先把认知立对：Jenkins 官方没有原生集群，HA 的正解是"无状态化 + 外部化"而不是"多节点共享一个 JENKINS_HOME"。无状态化的三层：一是构建执行全部交给外部 agent（含 K8s 动态 pod），Master 不再持有构建状态；二是 Job 配置和流水线全部代码化（Jenkinsfile + CasC + 共享库），Master 只剩"壳"；三是构建产物、测试报告、凭证尽量外部化（制品库、Vault）。做到这三层后，"Jenkins 坏了"退化成"重新部署一个 Jenkins + 恢复配置"，而不是"数据恢复"。备份要分清主次：必须备份的是 `jobs/*/config.xml`（Job 配置）、`credentials.xml` 与 `secrets/` 目录（注意 `master.key` + `hudson.util.Secret` 密钥必须成套备份，否则凭据解不开）、`config.xml`（全局配置）、插件清单；不必备份的是 workspace（重建即可）、大块构建产物（该在制品库）、过期的构建记录。工具上用 thinBackup 插件做定时增量备份 + 定期异地拷贝，关键是**恢复演练**——每季度从备份在空机器上重建一个实例，验证 Job 能跑、凭据能解、流水线能执行，很多团队"备份有了但从没恢复过"，真出事才发现密钥缺失或备份损坏。量化的目标是 RPO 15 分钟（备份频率）、RTO 1 小时（从备份恢复 + 重建）。事故复盘：我们有一次误删 JENKINS_HOME 中 `jobs` 目录，因为 Job 配置全部在 git（Jenkinsfile 化）且有 thinBackup 的 config.xml 备份，恢复只用了 40 分钟，而此前另一个团队靠"整盘复制"的备份恢复花了 4 小时还丢了凭据。多实例方案（两台 Master 共享 NFS 上的 JENKINS_HOME）不推荐做主力，NFS 锁与插件状态不同步会带来更诡异的故障，它只适合"故障转移"场景，正常负载还是单活 + 快速重建最稳。

**延伸考点：** 凭据的加密密钥和 `credentials.xml` 分开备份会有什么后果？K8s 上部署 Jenkins 时，PVC 丢失场景下的恢复路径怎么设计？

---

### Q16. 不同 Job 要不同环境（Java 8/17、Node 14/20），agent 怎么管理才不乱？

**问题：** 你们 20 个 Job 要求各不相同：有的要 JDK8、有的要 JDK17，有的要 Node 14、有的要 Node 20，还有的要 Docker 环境。现在全挤在几台装了一堆版本的大杂烩 agent 上，经常"这台能过那台挂"。构建环境管理怎么设计？

**期望加分项：**
- 讲清"镜像即环境"的容器化思路：用 Docker agent / Kubernetes pod 按 Job 指定镜像（`agent { docker { image 'maven:3.9-eclipse-temurin-17' } }`），环境随流水线版本化，杜绝 agent 漂移
- 说明标签（label）分层：静态 agent 按能力打 label（`java17 && docker`），容器 agent 按镜像维度提供，避免"版本全装一台机器"
- 资源隔离与配额：Docker 容器限制 CPU/内存（`--cpus`/`-m`）、K8s 用 pod 的 requests/limits 防止单构建吃光节点；同 agent 上并行 Job 用容器隔离避免污染
- 讲清取舍：静态 agent（成本低、启动快，但环境易漂移）vs 动态容器（环境纯净、可扩展，但镜像构建与拉取有开销、调试复杂）
- 有工具链管理：全局工具配置（JDK/Maven 自动安装）、`tools` 块按 Job 指定，避免每台 agent 手动装版本
- 有量化或事故佐证：环境不一致导致的"能过不能挂"排查案例、构建密度提升数据

**减分项：**
- 只会说"给 agent 装齐所有版本"，不知道这正是环境漂移的根源
- 说不出 Docker agent / K8s pod 的用法和镜像选择策略
- 不知道容器资源限制（CPU/内存配额），一个构建 OOM 拖死整台 agent
- label 乱打或不用，调度靠运气
- 无事故佐证，讲不出"环境不一致"的真实代价

**解答：**

核心原则是"环境必须跟随 Job 定义，而不是跟随某台机器的巧合"。首选方案是容器化 agent：`agent { docker { image 'maven:3.9-eclipse-temurin-17' } }` 让每个构建在独立容器里跑，镜像 tag 即环境版本——需要 JDK17 就指定 `eclipse-temurin:17`，需要 Node 20 就指定 `node:20-slim`，镜像由运维团队统一维护（基础镜像 + 预装工具），版本演进走镜像 tag 而不是"在某台 agent 上装了个新版本"。K8s 插件更进一步：流水线里用 `agent { kubernetes { yaml '''...''' } }` 定义 pod，requests/limits 直接写在 pod 模板里——`resources: requests: cpu 1, memory 2Gi; limits: cpu 4, memory 4Gi`，从根上防止单构建吃光节点内存拖垮其他构建。标签分层管理：静态 agent 只按"能力维度"打标签（`java17`、`docker`、`windows`），流水线用 `label 'java17 && docker'` 表达需求，不要把具体版本号揉进 label（版本漂移时 label 会过期）。还要用全局工具配置：在"Manage Jenkins → Tools"里定义多版本 JDK/Maven 并勾选自动安装，流水线里 `tools { jdk 'jdk17' }` 声明式指定，避免每个 agent 手动装。容器化的取舍要讲：静态 agent 启动快（秒级）适合小团队，但环境是"累积污染"的——今天装一个、明天升级一个，最终没人说得清机器上是什么；动态容器环境纯净可复现，代价是镜像拉取 30-60 秒、排查问题要进容器。事故佐证：我们曾因两台 agent 的 Maven 版本不一致（一台 3.8 一台 3.9），同样的代码出现"一台能构建一台报依赖错误"，排查两天后统一到容器 agent，此类问题绝迹；容器化后每台宿主机的构建密度也从"同时 2 个"提到"按资源配额动态跑到 6-8 个"。

**延伸考点：** Docker agent 的镜像每次构建都拉取吗？容器 agent 里挂载了哪些目录（workspace、tool、凭据），需要注意什么？

---

### Q17. 测试报告在 Jenkins 上看不到、覆盖率数字对不上、失败测试没人管，怎么集成和治理？

**问题：** 你们流水线里跑完测试就算完，构建页面上看不到测试结果和覆盖率，有 flaky 测试失败了也不影响合并。JUnit 报告、HTML 报告、覆盖率、失败重试这些怎么集成，怎么形成测试治理闭环？

**期望加分项：**
- 讲清报告采集标准路径：测试框架产出标准格式（JUnit XML、Surefire XML），流水线 `junit 'target/surefire-reports/*.xml'` 采集，构建页出现测试趋势图、失败/跳过明细、flaky 标记
- HTML 报告用 HTML Publisher 插件发布（`publishHTML`），注意 Jenkins 默认对 HTML 渲染的安全限制（CSP），用 `allowMissing`、`alwaysLinkToLastBuild` 等参数
- 覆盖率：JaCoCo/Coverage 插件从构建产物解析覆盖率数据，与 PR 评论/门禁联动（`recordCoverage` 后按阈值设门禁，如分支覆盖率 < 60% 构建失败）
- 失败重试的边界：`retry(2)` 只对"确实 flaky"的步骤（网络、资源竞争）合理，对"确定性失败"重试是掩盖问题；要区分"重试构建"（Rebuild）与"重试步骤"
- 治理闭环：测试失败必须能追溯到责任人（Jenkins 发通知给提交者）、flaky 测试要有自动标记与隔离机制（如每轮重跑 3 次的 flaky 检测 Job）、覆盖率下降触发告警
- 有量化：接入后"测试通过但线上出 bug"下降、"有测试但无人看"的问题被暴露

**减分项：**
- 只会说"加个 junit 步骤"，说不出报告格式从哪来、趋势图怎么来的
- 覆盖率只说"插件能算"，说不清怎么设门禁、谁为下降负责
- 对 flaky 测试"失败就重试"一刀切，掩盖确定性 bug
- 不知道 HTML 报告在 Jenkins 里的 CSP/渲染坑
- 无治理闭环：报告有了但失败无人跟进

**解答：**

测试集成分四步走。第一步"采集"：测试框架要能产出标准报告——Maven Surefire/JUnit 产出 XML（`target/surefire-reports/*.xml`），Jest/Go 等要配 reporter 输出 JUnit 格式；流水线里 `junit 'target/surefire-reports/**/*.xml'`（注意 include 路径），构建页就出现测试总数/失败/跳过的卡片和跨构建趋势图，失败用例点开能看堆栈和日志。第二步"HTML 与覆盖率"：HTML 报告用 HTML Publisher——`publishHTML(target: [reportDir: 'target/site/jacoco', reportFiles: 'index.html'])`，但 Jenkins 默认 CSP 会拦截内联脚本导致页面白屏，要么配系统 CSP 白名单、要么报告生成时禁用内联；覆盖率用 JaCoCo 插件 `recordCoverage(tools: [[parser: 'JACOCO']])`，数据来自构建产物里的 `jacoco.xml`，关键是把覆盖率变成门禁而不是摆设：`coverage = jacoco 分支覆盖率`，低于阈值（如分支 < 60%、行 < 70%）直接 `error()` 失败或发 PR 评论要求补齐。第三步"失败重试的正确姿势"：`retry(3)` 包住"确认为 flaky"的步骤（如拉外部依赖、申请测试环境），但对确定性失败（编译错误、断言失败）绝不能重试掩盖——判定方法是看失败是不是稳定复现；更系统的做法是"重跑失败用例"（`retry` + 只跑失败集）和 flaky 检测 Job（同一用例连跑 5 次，出现间歇失败自动标记并通知 owner）。第四步"治理闭环"：失败必须有人认领——测试失败时通知提交者（`emailext` 的 `culprits` 或 Git 插件按 commit 定位作者）；覆盖率下降 5% 以上告警到团队群；每周例会过一遍"上周 flaky 测试 Top 10"。量化佐证：接入 JUnit + 门禁后，曾经"测试全绿但合并后立刻炸"的几起事故，追查都是没跑进流水线的测试分支；覆盖率门禁上线后核心模块分支覆盖率从 40% 提到 65%+。

**延伸考点：** 流水线跑多套测试（单测/集成/E2E）报告都叫 `**/*.xml` 会重复采集吗，怎么分区展示？HTML Publisher 的 `alwaysLinkToLastBuild` 和 `keepAll` 参数什么时候用？

---

### Q18. 构建失败只靠"有人偶然看一眼"，通知和可视化怎么做才不吵又不漏？

**问题：** 你们 Jenkins 构建失败没人知道，要么是开发自己刷页面发现的，要么是有人配了个邮件通知结果所有人都收、没人看。通知策略、Blue Ocean、构建状态可视化怎么设计？

**期望加分项：**
- 讲清通知分层的本质：失败通知 vs 恢复通知 vs 例行成功通知分开处理——失败立即通知责任人、恢复时通知"已修复"、成功一般不通知（或只通知关键发布），避免告警疲劳
- 邮件用 Email Extension 插件：配 `emailext` 按构建结果分主题（`subject: '$DEFAULT_SUBJECT'`）、`culprits` 定位失败引入者、`recipientProviders` 按角色取收件人
- 即时通讯：Slack/企业微信/钉钉用插件发 `slackSend`/`dingtalk`，post 块里 `failure` 发失败 + 构建链接 + 失败摘要（日志尾部），`fixed` 发恢复；按 Job 重要度分级（核心发布 Job 全渠道、普通 Job 静默）
- 可视化：Blue Ocean 看流水线 stage 视图（比经典 UI 直观）、构建趋势插件（Build Trend）/Test Results Analyzer 看稳定性曲线；`timestamps()` 让日志可读
- 有治理：通知模板统一（共享库封装 `notify()` 函数）、渠道/收件人配置化、夜间构建失败降噪（合并为晨报）
- 有量化或事故：告警疲劳导致"狼来了"后真故障被忽略的教训

**减分项：**
- 只会说"配个邮件通知"，说不出失败/恢复/成功怎么分级
- 全员收邮件，不区分责任人，等于没人看
- 不知道 `culprits`、`recipientProviders` 这类"找责任人"的机制
- 只有邮件没有即时通讯，手机上看不到构建状态
- 通知与 post 块脱节，构建链接、失败摘要没带出来

**解答：**

通知设计的原则是"失败秒级触达责任人、恢复主动闭环、成功默认沉默"。第一层邮件：Email Extension 插件配 `emailext subject: '$DEFAULT_SUBJECT: ${BUILD_STATUS}'`，关键是收件人选择——`culprits: true`（最后导致构建变红的人）、`recipientProviders: [requestor(), developers()]`（触发者和代码作者），让邮件精确打到"该负责的人"而不是全员；内容带构建 URL、失败 stage、日志尾部关键信息（`${BUILD_LOG, maxLines=50}`），让人在手机邮件里就能判断要不要管。第二层即时通讯：Slack/企业微信/钉钉插件，post 块 `failure { slackSend color: 'danger', message: "构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}" }`、`fixed { slackSend color: 'good', message: '已恢复' }`，比邮件快得多，适合线上值班；按 Job 分级：核心发布 Job 走"邮件 + 群 + @责任人"全渠道，日常 CI Job 只在持续失败（如连续 2 次）时发，避免每失败一次就刷屏。第三层可视化：Blue Ocean 把流水线渲染成 stage 泳道，一眼看出卡在哪个 stage；配合 Build Trend / Test Results Analyzer 插件看成功率、测试稳定性的跨构建趋势；`timestamps()` 插件让每个日志行带时间，排障时能对齐"哪一步花了几分钟"。第四层治理：把通知封装成共享库函数 `notify(result, extra)`，模板统一、收件人按 Job 参数注入，避免每个 Job 各写一套；夜间（22:00-8:00）构建失败不进群，合并成早上 9 点一份"昨夜构建失败清单"日报。教训案例：某个团队"任何失败都 @所有人"，三个月后群被静音，一次生产发布 Job 挂了 6 小时没人处理——整改成"责任人 + 分级 + 恢复闭环"后，发布 Job 失败平均响应时间从小时级降到 10 分钟内。

**延伸考点：** `culprits` 判定"谁把构建搞红"的算法依据是什么？Blue Ocean 与经典 UI 同时存在时，Rest API 的差异会影响脚本吗？

---

### Q19. 构建卡住不动、agent 掉线、Jenkins 内存飙高，怎么系统化排障？

**问题：** 线上出过几类问题：某个构建卡在"等待下一个可用执行器"半天没动、agent 显示离线、Jenkins 主进程内存涨到 6G 卡死、重启后 Job 状态异常。你会按什么顺序排查？有哪些典型的"坑"和处理手段？

**期望加分项：**
- 讲清排障分层：先看队列/执行器 → 再看 agent 状态 → 再看日志（`/log/all`、`/systemInfo`、agent 日志）→ 最后看系统资源（JVM 内存、磁盘、GC）
- "卡住不动"三类根因：等 executor（队列堆积，看 executor 数和 agent 状态）、等待外部资源（`input` 挂起、`lock` 未释放、`waitForQualityGate` 没回包）、步骤本身 hang（`timeout` 没配）；对应手段：加 executor/查 lock 占用/手动终止或 `timeout`
- agent 掉线排查：SSH 超时/JNLP 连接断开、Master 重启后 agent 未重连、时钟偏移、磁盘满；处理：标记离线、重启 agent、检查 agent 日志
- 内存/GC 问题：Jenkins Master 跑太多构建或队列对象堆积、插件泄漏；手段：JVM 参数调优（`-Xmx`、G1GC）、定期清理构建记录与日志、用 thread dump（`/threadDump`）定位卡死点
- restart 的正确姿势：`/restart` 优雅重启（等运行中构建到检查点）vs 直接 kill 的风险；重启前备份、重启后检查 agent 重连和队列恢复
- 有排障实例：如"构建卡在 waitForQualityGate"、"agent 离线是磁盘满了"这类具体案例

**减分项：**
- 一上来就"重启 Jenkins"，不定位根因
- 不知道怎么看队列、executor、系统日志，排障靠猜
- 分不清"等 executor"和"步骤 hang"两类卡住，乱杀构建
- 不知道 thread dump、`/systemInfo`、GC 日志这些诊断手段
- 无真实案例，只背流程

**解答：**

排障要有顺序，我的固定路径是"队列 → agent → 日志 → 系统资源"。第一步看队列和 executor："构建卡住"先到 `buildQueue` 看是不是"等待下一个可用执行器"——如果是，查 agent 数、executor 数和运行中的构建，可能有 Job 占着 executor 在 sleep（没配 `timeout`），处理是加 executor 或找到占用的 Job；如果不在等 executor，是"执行中但不动"，进第二步。第二步看 agent：agent 显示离线，查连接方式——JNLP agent 看 Master 的 JNLP 端口和 WebSocket、SSH agent 看 ssh 连接和 agent 日志，常见根因是 Master 重启后 agent 没自动重连、agent 磁盘满、时钟偏移过大导致握手失败；处理是手动重连或重启 agent 并检查 `JENKINS_HOME` 所在磁盘。第三步看日志：`Manage Jenkins → System Log` 的 `/log/all` 和 agent 日志是主战场，构建 hang 在哪个 stage 就看那一步的完整输出和周围的系统日志。第四步系统资源：内存飙高先看是不是队列对象/构建记录无限堆积（`buildDiscarder` 没配），用 `/threadDump` 抓现场——能看到所有线程栈，卡住的线程栈直接告诉你它挂在哪个调用（比如 `WaitForQualityGate` 在等 Sonar 回包、`lock` 在等锁）；JVM 层面 `-Xmx` 调大 + 换 G1GC + 定期重启治理，GC 日志（`-Xlog:gc`）能证明是内存泄漏还是配置不够。关于重启：`/restart` 是优雅重启，让运行中的流水线先到检查点再重启，能续跑；直接 kill 进程会丢构建状态、可能导致 `shutdown` 类损坏。经典案例：一次"构建卡 3 小时不动"排查，thread dump 显示线程全部挂在 `hudson.plugins.sonar.QGStatus` 上——Sonar 的 webhook 没回调，卡死在 `waitForQualityGate`，解法是给该步骤加 `timeout`；另一次 agent 集体离线，根因是 `/tmp` 被构建垃圾占满，agent 进程写不了临时文件。

**延伸考点：** `waitForQualityGate` 在 Sonar webhook 丢失时怎么防卡死？优雅重启期间队列里的构建和正在运行的构建分别是什么状态？

---

### Q20. 公司要砍自建 Jenkins 迁到 GitHub Actions，划不划算？怎么迁？

**问题：** 你们自建 Jenkins 维护成本越来越高：两台 Master、30 台 agent 要人管，插件升级、磁盘、证书全是事。有人提议整体迁到 GitHub Actions，有人反对说"迁移成本高、还有 K8s 部署和私有制品库的依赖"。你怎么评估选型？真要迁怎么迁不翻车？

**期望加分项：**
- 能列出对比维度而不是情绪化结论：成本（自建服务器+运维人力 vs 托管按分钟计费）、生态（Jenkins 插件 2000+、Actions 的 marketplace action）、扩展性（自建可深度定制、Actions 天然并行矩阵、免费开源仓库额度）、锁定与合规（代码在 GitHub 的合规/私有化要求）、性能（runner 规格、缓存、区域）
- 讲清典型迁移决策：开源项目/团队规模小/无私有化要求 → Actions 省心；强合规（金融、私有化部署）、深度定制、内网隔离 → 自建 Jenkins 或自托管 runner
- 迁移策略：Jenkinsfile → workflow 映射（`agent`→`runs-on`、`stage`→`job`、`post`→`if`/`on` 的 always、`withCredentials`→`secrets`/环境变量）、自托管 runner 连接内网资源（制品库/私有集群）
- 讲清"混合过渡"：新项目走 Actions、存量 Job 分批迁移（Jenkins 保留只跑存量），迁移期双跑对比产物；或 Actions + 自托管 runner 保留内网能力
- 有量化：迁移后 CI 时长对比（Actions 矩阵并行 vs Jenkins 串行）、成本对比（server 采购/维护 vs Actions 分钟数账单）、人效
- 有风险意识：GitHub Actions 的可用性（偶发故障）、API 限流、私有仓库分钟数成本上升

**减分项：**
- 只会说"Actions 免费"或"Jenkins 插件多"，给不出对比维度和量化依据
- 不知道 Jenkins 的存量能力（自托管 runner 在内网、K8s 插件、私有制品库集成）在 Actions 里怎么对等
- 迁移一把梭，没有双跑对比和回退计划
- 忽略合规/私有化约束，把代码和 CI 交给第三方毫无风险意识
- 无成本或时长数据，纯拍脑袋

**解答：**

选型要按"对比维度 + 迁移路径"两条线答。对比维度：一是成本——自建 Jenkins 的账要算全：两台 Master + 30 台 agent 的机器成本、运维人力（插件升级、证书、磁盘、7×24 值班）、备份恢复；Actions 是按分钟计费的托管服务，开源仓库免费、私有仓库按量付费，大团队私有仓库用量大时账单可能超过自建服务器成本，需要拿过去一年的构建分钟数实测估算。二是生态与能力——Jenkins 2000+ 插件覆盖几乎所有内部系统（内网制品库、堡垒机、K8s），Actions 的 marketplace 也在追赶但内网集成本质上要靠"自托管 runner"打通，所以内网隔离、强合规（金融/政务私有化）、深度定制场景，自建仍占优；反之，纯开源/公网代码、团队规模小、要快速上手和免运维，Actions 明显更省。三是性能——Actions 原生矩阵并行（同一 stage 多版本多平台并行跑）比 Jenkins 手动 parallel 更省心，公共 runner 规格大（如 4 核 16G）且免维护，但要考虑网络延迟（runner 在 GitHub 侧，拉内网制品库慢）和偶发可用性波动。迁移路径：先做能力映射——`agent { label }` → `runs-on` + 自托管 runner 标签、`stage/parallel` → `job` + `matrix`、`post` → workflow 的 `if: always()` 与 `on` 触发器、`withCredentials` → GitHub Secrets 注入环境变量、Jenkinsfile 里的共享库 → composite action 或 reusable workflow；存量迁移别一把梭：新项目直接走 Actions，存量 Jenkins Job 按"低依赖优先"分批迁，迁移期两个 CI 对同一分支双跑、对比构建产物 hash 一致再切流，Jenkins 保留一段时间兜底可回退。最后给结论：如果公司代码本来就全在 GitHub 且无私有化要求，Actions + 自托管 runner（连内网制品库和 K8s）是性价比最优解；如果有强合规或深度定制（我们当时评估过迁移 30 个 Job 的改造工时约 3 人周，且合规要求 CI 不出内网），继续自建 Jenkins 并做好 HA 治理也完全合理——选型没有对错，只有"哪个成本结构适合你们"。

**延伸考点：** GitHub Actions 自托管 runner 与 Jenkins agent 在安全模型（谁能在 runner 上执行代码）上的差异是什么？迁移后流水线里依赖的内网服务（制品库、Sonar）怎么访问？

---

### Q21. 构建频繁失败且越来越慢，从 Jenkins 侧怎么整体排查与治理？

**问题：** 你们 Jenkins 最近一个月构建失败率翻倍、单次构建时长越来越长，团队已经开始"失败就点重跑"靠运气。从 Jenkins 侧做一次整体排查与治理，你会按什么顺序入手，覆盖 agent、资源、流水线设计、缓存哪些层面？

**期望加分项：**
- 先量化再动手：用 Build Trend、Test Results Analyzer 和队列/时长统计确认失败率、平均时长、排队时长的基线，区分是"个别 Job 劣化"还是"全局系统性劣化"，避免凭感觉加机器
- 建立四层排查框架：资源层（executor 数、磁盘、内存、网络）→ agent 层（环境漂移、某台 agent 过载/掉线）→ 流水线层（串行步骤、全量重建、缺 timeout）→ 缓存层（缓存失效、无缓存）
- 资源治理：动态 agent（K8s/Docker）按需扩容、executor 配额与并发上限（`disableConcurrentBuilds`）、磁盘清理（buildDiscarder + workspace 清理）
- 流水线设计：可并行 stage 用 `parallel` 拆开、增量构建（只编译变更模块、测试只跑变更）、共享库统一封装标准阶段
- 缓存策略：Maven/Gradle/npm 依赖缓存持久化（agent 本地目录或挂载卷）、缓存键按锁文件设计、命中率监控
- 有量化佐证：治理前后失败率、平均时长、排队时长的对比数字

**减分项：**
- 一上来"加 agent、换大机器"，不先量化定位瓶颈在资源还是流水线
- "失败就重跑"，不区分环境问题与代码问题，掩盖根因
- 只优化单点（只加缓存或只调 executor），无整体排查框架
- 忽略磁盘、网络、agent 漂移等基础设施因素
- 无治理前后数据对比，收益说不清

**解答：**

治理的第一原则是先量化、再分层排查，而不是一上来就加 agent。先用 Build Trend 和 Test Results Analyzer 拉出基线：失败率、平均构建时长、队列等待时长，以及失败集中在哪几个 Job——失败集中在特定 Job 说明是流水线/代码问题，全面劣化才是资源或基础设施问题。第二步按四层排查：资源层看 executor 是否耗尽（队列积压）、磁盘是否被构建产物和 workspace 塞满（磁盘满会导致构建写到一半报错）、Master 内存与 GC 是否正常；agent 层看是不是某几台 agent 环境漂移（"这台过那台挂"）、是否过载或掉线；流水线层看是否存在全量重建、测试全跑、无 timeout 的低效写法；缓存层看依赖是否每轮重新下载。治理动作按杠杆排序：先做"救活失败"的——磁盘清理（buildDiscarder 设构建保留策略 + 定时清理 workspace）、给高危步骤加 `timeout` 防卡死、统一 agent 环境（容器化 agent 彻底消除漂移）；再做"让构建变快"的——依赖缓存持久化（Maven 的 `~/.m2`、Gradle 的 `~/.gradle`、npm 缓存目录挂到 agent 本地或持久卷，缓存键绑定锁文件，命中率目标 85%+）、并行化（`parallel` 拆可并行阶段、测试按模块分 Job）、增量构建（增量编译 + 只跑变更模块测试）；最后做"防止再变慢"的——动态 agent 按队列深度扩容、给高并发 Job 设 executor 配额。我们团队一次治理的量化结果：失败率从 12% 降到 2% 以内（一半根因是磁盘满和 agent 漂移），平均时长从 26 分钟降到 11 分钟，失败处理从"靠重跑碰运气"变成"有明确的排查路径"。

**延伸考点：** 同一批构建时快时慢、偶发失败，如何用数据区分"agent 环境漂移"与"代码/依赖问题"？依赖缓存命中率低常见是哪些原因（缓存键、平台差异）？

---

### Q22. 老团队 Jenkins 上散落几百个自由风格 Job 且无规范，怎么治理与重构？

**问题：** 你们接手一个老团队的 Jenkins：几百个自由风格 Job 散落在根目录，命名混乱，权限是"人人都能改"，不少 Job 没人说得清是干什么的。你要做一轮治理与重构，方案怎么设计？

**期望加分项：**
- 盘点先行：导出全部 Job 配置（Job 名、最后构建时间、触发方式、owner、是否被下游引用），按"核心 CI/发布链、定时任务、一次性脚本、僵尸 Job"分类，僵尸 Job 先停用观察
- 目录与权限治理：Folder 插件按团队/项目建文件夹，矩阵授权收敛——admin 全权、各团队只能管自己文件夹下的 Job 与 view，凭据按文件夹授权
- 分阶段迁移：新建 Job 一律 Jenkinsfile（配置即代码），存量按价值排序用"影子运行"迁移（新旧并行对比产物），Job DSL 把存量 Job 定义代码化
- 清理策略：僵尸 Job"先禁用后删除"（禁用观察 1-2 个发布周期确认无依赖），构建记录用 buildDiscarder 设保留策略
- 规范固化：命名规范（项目-阶段-用途）、参数化、共享库提供标准阶段模板、准入规则（新 Job 必须走 PR）
- 有量化：治理后 Job 数量、配置漂移事故、权限问题的变化

**减分项：**
- 上来就批量删除，没有盘点与 owner 确认，删错核心 Job 酿事故
- 只搬 UI 配置不改权限，治理完还是"人人可改"
- 全量迁移一把梭，无试点无回退
- 无命名规范与准入机制，治理完半年又乱回去

**解答：**

治理的核心是"先盘清楚、再分类处理"，忌讳上来就删。第一步盘点：把几百个 Job 的配置导出（config.xml 或 Rest API），逐条记录 Job 名、最后构建时间、触发方式（手动/SCM 轮询/定时）、owner 和下游依赖，做成清单。第二步分类：核心发布链（build→test→deploy）和定时任务属于"必须保"；一次性脚本类看是否还有人用；长期无构建、无 owner 的标记为僵尸 Job。第三步目录与权限：用 Folder 插件按团队/项目建文件夹，把 Job 全部归入对应文件夹，再用矩阵授权收敛——admin 全权，团队成员只能在自己文件夹里创建和配置 Job，其他文件夹只读或不可见，凭据按文件夹授权，从根上解决"人人可改"。第四步分阶段迁移：新建 Job 从这一刻起一律 Jenkinsfile 进仓库，禁止再在 UI 里建；存量高价值 Job 用"影子运行"迁移——新 Jenkinsfile Job 与旧 Job 并行跑同一分支，对比产物 hash 和测试结果一致后再停旧的；僵尸 Job 先"禁用"观察 1-2 个发布周期，确认没人报"这个构建怎么没了"再归档删除，同时用 buildDiscarder 给存量 Job 设构建记录保留策略，控制磁盘。第五步防回潮：命名规范（项目-阶段-用途）、共享库提供标准流水线模板、Jenkinsfile 必须走 PR 评审，把"规范"变成"机制"，而不是靠自觉。我们团队当时 400 多个 Job 收敛到 120 个有效的（60 多个确认是僵尸），核心 30 条迁移到流水线，权限事故（误改别人 Job）从每月几起归零。关键教训：删 Job 前一定先禁用观察，我们有一次禁用 3 天后才发现有一个是财务对账的定时任务——好在是禁用不是删除。

**延伸考点：** "没人知道干什么"的历史 Job，确认可删的手段还有哪些（调用链分析、访问日志）？迁移到 Jenkinsfile 后如何防止"又有人在 UI 建 Job"回潮？

---

### Q23. Jenkins 迁移上 Kubernetes 的高可用改造怎么做：无状态化、外部存储、动态 agent、备份迁移？

**问题：** 你们 Jenkins 还是单机跑的"宠物"，宕机一次恢复要半天，现在要迁到 Kubernetes 上做高可用改造。方案怎么设计？无状态化、外部存储、动态 agent、备份迁移各有哪些要点和坑？

**期望加分项：**
- 架构定位：K8s 上 Jenkins Master 用 Deployment 单副本部署，"高可用"的正确理解是"快速重建"而非多副本负载均衡（官方无原生集群，多副本共享 JENKINS_HOME 是伪 HA）
- 无状态化三件套：构建下沉 K8s 动态 agent（Kubernetes 插件按需弹 pod）、配置代码化（Jenkinsfile + CasC + 共享库）、产物/报告外部化（制品库），Master 只剩"壳"
- 外部存储：JENKINS_HOME 用 PVC 持久化（单副本 RWO），工作区不持久化（emptyDir 构建完即销毁），依赖缓存目录单独挂载
- 动态 agent 细节：Pod 模板（镜像、资源 requests/limits）、容器里跑 docker build 的取舍（DIND 特权 / kaniko / 挂载 docker.sock）、label 与 Pod 模板映射
- 备份迁移：迁移前全量备份（config.xml、credentials.xml + master.key、插件清单、jobs 配置），新环境先 restore 演练再切流；上线后 thinBackup 定期备份 + 恢复演练
- 有实战坑佐证：PVC 权限（非 root 运行、fsGroup）、时区/DNS、镜像拉取慢、JNLP 连接、资源配额打爆节点

**减分项：**
- 以为上 K8s 就是高可用，上多副本共享 PVC（RWO 单写、并发写冲突）直接翻车
- 状态没外移：工作区、构建产物还留在 Jenkins 里
- 迁移不带 master.key，恢复后凭据全部解不开
- 动态 agent 的镜像/资源/Docker 能力考虑不全，构建在 pod 里跑不起来

**解答：**

先立认知：K8s 上的 Jenkins"高可用"不等于多副本集群。Jenkins 官方没有原生集群，JENKINS_HOME 是单一事实源，多副本共享同一个 PVC（RWO 模式）会导致并发写冲突、插件状态不一致，是伪 HA。正确思路是"无状态化 + 快速重建"：Master 变成随时可推倒重建的薄壳，可用性靠 Deployment + 就绪探针 + 资源限额保证，故障恢复目标从"半天"降到"几分钟"。无状态化分三步：一是构建执行全部下沉到动态 agent——Kubernetes 插件在构建请求到达时按 Pod 模板弹 pod，构建完自动回收，Master 不再持有构建状态；二是配置代码化——Jenkinsfile 进仓库、系统配置用 CasC 固化、共享库进 git，Master 丢失后从 git 一键重建；三是数据外部化——构建产物和测试报告进制品库，工作区不持久化（emptyDir 构建完即销毁），Jenkins 里只留 Job 配置与凭据。外部存储：JENKINS_HOME 挂 PVC（注意单副本 RWO，多副本场景要么上锁方案要么干脆单活），依赖缓存可以单独挂一个 PVC 或放 agent 本地，避免每轮重新下载。动态 agent 的坑集中三处：容器里跑 docker build 的三条路——DIND（特权容器，安全弱）、kaniko（无特权，推荐）、挂载 docker.sock（有权限扩散风险）；Pod 模板必须给 requests/limits，否则单构建吃光节点拖死其他 pod；label 要映射到对应 Pod 模板，别让调度靠运气。备份迁移是最后一道保险：迁移前全量备份 JENKINS_HOME，特别强调 `master.key` 和 `secrets/` 必须成套备份，否则恢复后凭据解不开；新环境先做 restore 演练（Job 能跑、凭据能解、流水线能执行）再切流；上线后 thinBackup 定期备份 + 每季度恢复演练，量化 RPO/RTO。我们迁移时踩过最痛的坑：镜像用非 root 用户跑，PVC 权限不足导致 JENKINS_HOME 写不进去，排查半天——PVC 的 fsGroup 和镜像用户的 UID 必须对齐。

**延伸考点：** K8s 上 Jenkins Master 的 PVC 备份如何与云厂商快照/对象存储结合？动态 agent 的大镜像拉取延迟怎么优化（镜像预热、nodeSelector、缓存）？

---


