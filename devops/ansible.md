# DevOps · Ansible（面试题库）

本文件考察候选人在 Ansible 上的真实落地能力：无 agent 架构与控制节点/受管节点模型、Inventory 与动态清单、Playbook/任务/模块的幂等设计、变量与事实、Jinja2 模板、角色复用、vault 与敏感数据治理、forks/serial 大规模执行与滚动更新、以及提权与 SSH 信任模型下的安全边界。题目全部为场景化提问，不考八股文——重点看候选人能否给出量化依据、说清取舍、引用线上事故与排障过程、主动覆盖边界条件，难度从实践基础渐进到架构级设计。

---

### Q1. 新入职接手一套 200 台机器的配置管理，第一反应为什么是 Ansible？

**问题：** 你新入职一家公司，生产环境有 200 台 Linux 机器分散在三个云账号，此前靠运维手动 SSH 改配置，已经出过多次"漏改一台导致线上事故"。你评估要不要引入配置管理工具，为什么最终选了 Ansible（而不是 SaltStack/Puppet/Chef）？请先讲清它的核心架构：控制节点、受管节点、无 agent 是怎么工作的？

**期望加分项：**
- 一句话说清无 agent 的本质：控制节点通过 SSH 连接受管节点，执行完把 Python 模块代码推送过去临时运行即收走，受管节点无需常驻进程，省去 agent 升级与证书续期负担
- 讲清模块（module）是执行单元、幂等是模块级契约：每个模块实现"描述目标状态，只在有差异时才变更"，能举 file/lineinfile/copy 等例子说明
- 说明控制节点的部署成本低：一台机器装 ansible + SSH 即可起步，适合"从手动运维过渡"的现实约束
- 对比时给出取舍而非贬低：Puppet 有 agent 与 DSL、适合长期收敛；Ansible 强调过程式编排 + 可读性，与 CI/CD 流水线配合更直接
- 提及 ansible.cfg、模块库、inventory、playbook 四大组件的职责边界
- 能指出无 agent 的代价：SSH 往返开销、每轮全量事实收集、依赖 Python 环境，这引出后续大规模性能问题

**减分项：**
- 只会背"无 agent、走 SSH"，讲不清模块如何被推送执行、幂等如何实现
- 把"无 agent"说成"不用装任何东西"，忽略受管节点需要 Python 解释器
- 对比工具时说不出任何量化理由或场景差异，只是报菜名
- 不知道 ansible.cfg / inventory / playbook 各自管什么
- 没有意识到 200 台规模下 SSH 往返与事实收集的性能问题

**解答：**

Ansible 的架构可以概括为"控制节点 + SSH + 模块"：控制节点（管理机）持有 inventory 清单、playbook 和模块库，通过 SSH 连到每个受管节点，把对应的 Python 模块代码经管道推送到目标机器临时执行，执行完进程退出、不留任何常驻 agent。这是它最大的工程红利——不需要在 200 台机器上预装 agent、维护 agent 版本、处理 agent 与主控的证书配对，新机器只要能被 SSH 登录就能纳入管理，这对"存量环境迁入"极为友好。核心概念有三个：inventory 描述"管谁"（主机与分组），playbook 描述"怎么管"（任务序列），模块是原子执行单元。幂等性就建立在模块契约上：比如 `file` 模块创建目录，目标已存在且属性一致时返回 ok 不做任何变更；`lineinfile` 只在文件里缺那一行才追加。所以同一份 playbook 跑一次和跑一百次，最终状态一致、只有首次有 changed。选择 Ansible 的取舍在于：它的过程式模型直观可读、调试链路短（一条命令看报错），对"从手动运维迁移"的团队学习曲线最低；而 Puppet 的声明式收敛模型更强，但引入 agent 体系后运维成本高，适合需要长期漂移修复的场景。200 台规模下代价也随之显现：每次执行都要建 SSH 连接、收集 facts、传输模块，全量跑一轮常常超过十分钟——这为后来的 ControlPersist、pipelining 和 forks 调优埋下伏笔，面试中主动提到这一点说明你真的跑过大规模。

**延伸考点：** 受管节点 Python 版本过低（2.6/2.7）时 Ansible 会怎样处理？模块为什么要依赖受管节点的 Python 环境？

---

### Q2. 生产机新增 50 台，还要按机房/角色/灰度批次管理，Inventory 怎么设计？

**问题：** 你们要为 50 台新机器上线做配置管理：机器分布在华北、华东两个机房，分 Web/DB/Redis 三种角色，灰度发布还要区分批次（batch1/batch2）。你打算怎么组织 inventory？INI 和 YAML 两种格式各有什么取舍？"动态 inventory"什么时候才需要？

**期望加分项：**
- 讲清三种组织维度各用一类东西表达：物理位置用父组（group），角色用子组（group），灰度批次用变量或额外子组，避免把批次硬编码进主机名
- 演示嵌套组写法：`[web:children]` 语法，说明子组自动继承父组、子组的变量优先级高于父组
- 说清主机变量与组变量的写法与加载层级（host_vars/group_vars 目录 vs inventory 内联），以及默认行为：主机变量覆盖组变量
- 量化使用场景：静态清单只有几十台时 YAML/INI 足够；云上机器频繁伸缩时必须动态 inventory（AWS 用 `aws_ec2` 插件，按 tags/region 过滤），给出去 tag 建组的配置片段
- 会讲 `--limit` 与批次的关系：ansible-playbook 加 `--limit batch1` 只跑灰度批次
- 主动提 meta 分组 `all`/`ungrouped` 和 `ansible_host` 别名、SSH 端口/用户名在 inventory 里的配置方式

**减分项：**
- 只知道 INI 写 `[group]`，说不出嵌套组语法
- 把灰度批次设计成"复制三份相同配置的组"，体现不出抽象能力
- 分不清组变量与主机变量的优先级
- 静态清单几百台机器还手写，没意识到动态 inventory 的价值
- 不知道 host_vars / group_vars 目录约定，变量全堆在 inventory 文件里

**解答：**

Inventory 的本质是"主机 → 分组 → 变量"的三层组织，设计目标是把机器拓扑和业务维度解耦。以你的场景为例，机房用父组 `[cn-north]`/`[cn-east]`，角色用 `[web]`/`[db]`/`[redis]`，批次用 `[web-batch1]`/`[web-batch2]` 这类子组，并用 `:children` 语法把角色与批次挂到机房父组下，例如 `[cn-north:children] web db redis`——这样执行 playbook 时可以按任意维度圈选主机。变量分层上，约定 `host_vars/<hostname>.yml` 与 `group_vars/<group>.yml` 目录，Playbook 同目录下自动加载；优先级是主机变量 > 组变量 > inventory 顶层变量，父组变量会被子组覆盖，这个"就近覆盖"规则必须背下来，否则会出现"改半天不生效"。文件格式上，INI 紧凑适合手写少量机器；YAML 的 `all:`/`children:`/`hosts:` 结构更适合表达嵌套与变量内联，也更便于后续转成动态清单。当机器数量大、生命周期短（云上扩容缩容频繁）时，静态文件必然漂移，这时用动态 inventory 插件：以 AWS 为例，`ansible-inventory --list` 通过 `aws_ec2` 插件按 tag 自动生成分组，机器打 tag 即入组、销毁即消失，配合 `--limit` 就能做"只对 batch1 灰度"。线上工程里还有个实用细节：IP 变化时用 `ansible_host` 指向真实地址、inventory 里的名字只当逻辑别名，否则 IP 一变整个 host_vars 目录全部失效。

**延伸考点：** `--limit` 和 inventory 里的批次分组在灰度场景各有什么优劣？动态 inventory 缓存（cache）配置在什么场景会踩坑？

---

### Q3. 半夜批量改 30 台机器密码，你写 Playbook 还是敲 ad-hoc？

**问题：** 有两个任务同时落在你头上：一是临时给 30 台机器的某个用户批量改密码、顺手查看所有机器的磁盘使用率；二是要把一套新部署规范固化下来长期复用。你会分别用 ad-hoc 还是 Playbook？两者的适用边界怎么划？

**期望加分项：**
- 讲清 ad-hoc 的价值：`ansible <host-pattern> -m <module> -a "args"` 单条命令直达目标，适合一次性、只读诊断、快速验证模块行为
- 明确边界的判断标准：是否要复用、是否要编排顺序/条件/回滚、是否要审计留痕——一旦"需要重现"就该写 Playbook
- 举出自己用 ad-hoc 的真实场景：`ansible all -m ping` 探活、`-m shell -a 'df -h'` 批量看磁盘、`-m user -a 'password=xxx'` 批量改密、`-m apt -a 'name=vim state=latest'` 批量装包
- 说清 ad-hoc 的局限：单模块、无顺序控制、无失败处理、无 handler，改密码这类操作如果不带 `--become` 还会直接权限失败
- 会指出 ad-hoc 改密码的实践坑：直接明文写在 shell 历史里，正确做法是 vault 或 lookup 密码生成器，且 user 模块对已存在的用户要显式 `update_password: always` 才覆盖
- 能引出"能幂等的操作才有安全执行的价值"——ad-hoc 同样受模块幂等性保护

**减分项：**
- 只会背"一次性用 ad-hoc、复杂的用 playbook"，给不出判断标准
- 不知道 ad-hoc 不带 become 提权会失败、不知道常用模块的 -a 参数格式
- 把临时操作写成 playbook 文件再删除，反而多此一举
- 改密码直接明文上命令，毫无安全意识
- 不清楚 `ansible` 命令与 `ansible-playbook` 命令的差异

**解答：**

划分标准只有一个：这个操作"要不要被重现、被审计、被编排"。临时性的、单动作的、只读的，用 ad-hoc 最划算——`ansible web -m ping` 探活、`ansible web -m shell -a 'df -h | head'` 看磁盘、`ansible db -m apt -a 'name=postgresql state=present'` 补装依赖，一条命令批量打完收工，没有文件落地、不用维护。但 ad-hoc 有两个硬伤：一是只支持单模块单动作，无法表达"先停服务再改配置再启动"的顺序和失败依赖；二是不留痕，命令不在 git 里、不经过代码评审，出了事故无法复盘。所以凡是"会上线系统"的操作都要写 Playbook：部署、配置变更、用户管理这类会改状态的，写成 playbook 进版本库，review、跑测试、留审计，一劳永逸。改密码这个具体例子最能说明边界：用 ad-hoc 的 `user` 模块批量改密码技术上没问题，但要命的是参数必须带 `--become` 才能写 /etc/shadow，而且明文密码会留在 shell 历史里；工程化做法是 playbook + vault：密码存 vault 加密文件，task 用 `update_password: always` 显式覆盖（Ansible 默认对已存在用户跳过改密，很多人在这里踩坑——以为改了其实没改），执行时 `ansible-vault view` 或直接引用加密变量。我自己的习惯是：任何会在生产留痕的操作，即便只有一行，也先写成 playbook 挂在 git 里走评审，ad-hoc 只留给探活、诊断、临时排障三类场景。

**延伸考点：** `user` 模块改密码时 `update_password: always` 与默认 `on_create` 的行为差异，会带来什么线上隐患？为什么说 ad-hoc 的 `shell` 模块输出无法结构化解析？

---

### Q4. 新人写的 Playbook 顺序全乱了，你怎么讲解 play/task/module 的执行模型？

**问题：** 团队新人照着教程写了一个"先安装 Nginx、再写配置、再启动服务"的 playbook，跑完发现服务没起来，他不知道去哪查、也不知道执行顺序到底由什么决定。请你用他的例子，把 play/task/module 三层结构、执行顺序和执行结果的含义讲清楚。

**期望加分项：**
- 讲清三层模型：playbook 由若干 play 组成（每个 play 圈定主机和任务清单），play 由按顺序执行的 task 组成，每个 task 调用一个 module，task 的顺序是"书写顺序 + when/loop/handler 等控制流"共同决定
- 解释执行结果四种状态：ok（无变更）、changed（有变更）、failed、skipped，以及它们在调试里的意义
- 用新人例子点出关键坑：service 模块启动依赖"配置语法正确"，如果配置 task 里模板变量未定义导致生成的文件是错的，service 照样会报失败或起不来——所以排查顺序是逐 task 看 ok/changed/failed
- 演示 `ansible-playbook -v` / `-vvv` 看详细输出、`-C` 先 dry-run 验证，能定位到具体 task
- 讲清 hosts/name/tasks 的语法骨架，强调每个 task 必须 name（可读性即可维护性）
- 会提 handlers 在 play 末尾统一触发的时序语义，顺带说明"服务没起来"也可能是 handler 未触发

**减分项：**
- 说不清 play 与 task 的层次关系，把"执行顺序"简单归结为"从上到下"
- 不知道 ok/changed/failed/skipped 四种结果各代表什么
- 遇到失败不会用 -v 定位到具体 task，只会贴报错
- 以为 task 的 when 条件不影响顺序、以为 handlers 会立即执行
- 没有 name 命名的习惯，playbook 可读性差

**解答：**

先把三层结构钉死：Playbook 是"要做什么"的总纲，里面可以包含多个 play；一个 play 定义"对哪些主机（hosts）+ 依次做哪些任务（tasks）"；每个 task 调用一个模块（module）完成一个原子动作。执行顺序是逐 play、逐 task 从上到下推进，但真正决定行为的是控制流——task 里的 `when` 决定"这步跳不跳"、`loop` 决定"重复几次"、handler 决定"变更后回调哪个动作"，这些叠加上才是完整的顺序语义。执行结果的四种状态是调试的罗盘：`ok` 表示模块检查后目标已符合状态、没做任何事（幂等）；`changed` 表示产生了实际变更；`failed` 表示报错；`skipped` 表示被 when 条件跳过了。新人那个"Nginx 没起来"的例子，最可能的剧本是：install 的 apt 模块 ok、template 的配置 task 也显示 ok，但因为变量拼错，生成的 nginx.conf 里 server 段是坏的，service 模块执行时 start 失败——或者更隐蔽的：配置没变（ok）时 handler 不会触发，服务用的还是旧配置。正确排查路径：`ansible-playbook -C nginx.yml` 先干跑一遍看哪些会是 changed，再 `-v` 看每个 task 的模块输出定位失败点，最后上机器 `nginx -t` 验证配置、`systemctl status nginx` 看实际状态。所以给新人的第一课是：不要盯着"最终失败"看，把执行输出按 task 拆开，ok/changed/failed 就是你要的答案。这也顺带解释了为什么每个 task 必须写 name——报错、日志、审计全部依赖它来定位。

**延伸考点：** 同一台主机在多个 play 中出现时，facts 收集的时机对后续 play 有什么影响？handler 为什么一定在 play 末尾执行而不是触发即执行？

---

### Q5. 同一条命令在 10 台机器上结果各不相同，你的 Playbook 里装依赖、写文件、管服务分别该用哪些模块？

**问题：** 你们从 CentOS 迁移到 Ubuntu，发现原来手写的"装包、改配置、起服务"shell 脚本在新系统上跑挂了一半——yum 没了、路径变了、判断逻辑到处都是 if。如果改成 Ansible，file/copy/command/shell/apt/yum/service/user 这些模块各自该在什么场景用？它们是怎么保证幂等性的？

**期望加分项：**
- 按"声明式优先"原则给出模块选择清单：装包用 apt/yum（自带幂等，包已装则 ok）、写配置用 copy/template/lineinfile、管服务用 service/systemd、建用户用 user、纯诊断或别无他法才用 command/shell
- 说清 command vs shell 的差异：command 不经过 shell 解释（无管道/重定向/变量展开），shell 才走 /bin/sh——能用 command 就不用 shell，shell 是"退出合规手段"
- 强调 file 模块的幂等语义：state 参数决定目标（directory/file/link/absent），已满足即 ok；copy 用 checksum 判断是否需覆盖，避免无条件重写
- 能举出每类模块的典型坑：apt 加 `update_cache: yes` 的时机与 cache 失效、service 的 enabled/state 都要显式声明、user 模块默认不更新已有用户密码、file 的 mode 权限八进制写法
- 提到 replace/lineinfile 做"存在性修改"比 shell+sed 更可审计、更幂等
- 会主动说"模块不存在时才考虑 command/shell"，并给出判断：该操作有没有对应用模块

**减分项：**
- 一上来全用 shell 写，把 playbook 写成"带注释的 shell 脚本"，完全没有利用模块幂等性
- 说不清 command 和 shell 的区别
- 不知道 copy 默认按 checksum 判断、不知道 service 需要同时管 enabled 和 state
- 装包时从不考虑 update_cache，在 Ubuntu 上装新包时反复失败
- 用 file 模块改权限时 mode 写错成字符串或忘加引号

**解答：**

核心原则一句话：能用声明式模块表达"目标状态"的，绝不用命令式 shell。按场景拆开：装包用 `apt`（Debian 系）或 `yum`，模块内部先查包是否已安装，已装则 ok——这是天然幂等；注意 Debian 系装新包前要 `update_cache: yes`，但每次跑都 update 又拖慢全流程，工程上常把 update_cache 单独放一个 task 或用 `apt update` handler 控制。写文件分三种：整体覆盖用 `copy`（本地文件推上去，Ansible 比较校验和，内容一致就不动）；内容里有变量用 `template`（Jinja2 渲染）；只增删某几行用 `lineinfile`（按正则匹配，存在则跳过）——这三种都比 `shell + sed` 可审计得多，因为输出明确告诉你"改没改"。管服务用 `service` 或更精确的 `systemd`，必须同时声明 `state: started` 和 `enabled: yes`，很多线上事故就是"服务起来了但重启后没自启"。建用户用 `user`，创建、设密码、加组、上锁都能做，但要注意它默认对已存在的用户跳过密码更新，需要 `update_password: always`。`command` 与 `shell` 的差别：command 直接把参数交给模块执行、不经 shell，所以不支持管道和重定向，但更安全可控；shell 才走 `/bin/sh`。选择标准是：这件事有没有对应模块？比如"改内核参数"有 `sysctl` 模块、"配 cron"有 `cron` 模块，都不要用 shell 硬写。我见过的反面教材，是把全套部署脚本原样抄进 shell task，结果既不幂等也不可调试，变更全靠肉眼对比——这恰恰是 Ansible 要解决的问题。

**延伸考点：** `apt` 模块的 `state: latest` 与 `state: present` 在生产上的取舍是什么？为什么尽量别用 latest？copy 与 template 都支持校验和判断，它们判断"是否需要更新"的机制有何不同？

---

### Q6. 同一个变量在 8 个地方定义了，Nginx 配置里的端口怎么就对不上？

**问题：** 你的 playbook 里既有 group_vars、host_vars、play 级 vars，又有 register 的结果和 setup 自动收集的 facts，某个 Nginx 端口变量在配置文件里渲染出来和预期不符。你打算怎么定位？请把 Ansible 的变量体系与优先级讲全。

**期望加分项：**
- 讲清变量来源分层：inventory 内联/group_vars/host_vars、play 的 vars/vars_files、task 的 register/set_fact、facts（setup 自动收集）、命令行 -e 额外变量
- 熟练背出关键优先级结论：命令行 `-e` 覆盖一切；`set_fact` 与 `register` 属于 host 级运行时变量，优先级高于大多数静态定义；host_vars 高于 group_vars；play vars 高于 inventory 变量
- 会用 `ansible <host> -m debug -a 'var=ansible_facts'` 或 playbook 里 `debug` 模块打印变量实际值来定位，而不是猜
- 说明 facts 的收集机制：setup 模块默认在每个 play 开始时对每台主机执行，可用 `gather_facts: no` 关掉提速；缓存 facts（jsonfile 缓存插件）是大规模优化手段
- 讲清 `{{ }}` 模板求值时机与变量未定义（`ansible_facts` 命名空间、`undefined`）的处理：default 过滤器、`{{ var | default('x') }}`
- 提到变量名规范与 namespace 隔离：prefixed（如 nginx_port）避免冲突

**减分项：**
- 只知道"优先级表"但背不完整或背错，答不出 -e 覆盖一切、register 覆盖 play vars
- 排查时不会用 debug 模块看实际值，只会逐文件翻
- 不知道 gather_facts 默认开、不知道 setup 模块的存在
- 把 `{{ }}` 用在 when 里还加引号导致求值异常（如 `when: "{{ x }} == 1"` 的反模式）
- 说不出 register 的作用范围是"该主机、该 play 之后的所有 task"

**解答：**

Ansible 的变量体系可以分成静态定义与运行时数据两条线。静态定义按"离主机越近越优先"的直觉排列：`-e` 额外变量 > 命令行 > play 级 `vars` > host_vars > group_vars > inventory 内联变量 > 角色 defaults（defaults 优先级最低，这就是"角色默认值可被一切覆盖"的设计）。运行时数据是另一条线：`register` 把上一个 task 的结果存进变量（作用域是该主机、该 play 之后的 task）、`set_fact` 显式创建/覆盖主机级变量，这两者的优先级高于绝大多数静态定义——所以"端口对不上"最常见的真凶，就是某个 register 或 set_fact 悄悄覆盖了你在 group_vars 里定义的值。facts 是 setup 模块自动收集的主机信息（IP、OS、内存、网卡等），挂载在 `ansible_facts` 命名空间下（2.9 后推荐 `ansible_facts.xxx` 而非裸 `ansible_xxx`），默认每个 play 开头收集一次，200 台机器光收集 facts 就要一两分钟，所以性能敏感场景会 `gather_facts: no` 只在实际需要时手动 setup 或用 `gather_subset` 裁剪。排查这种问题只有一条路：打印出来看。在 playbook 里插一个 `- name: 打印端口 debug: var=nginx_port`，或在命令行 `ansible web -m debug -a 'var=nginx_port'`，看真实值是多少、再倒查它在哪一层被定义/覆盖。模板求值还有两个经典坑：`when` 条件里不要再套 `{{ }}`（写成 `when: nginx_port == 8080`），变量未定义时用 `{{ nginx_port | default(8080) }}` 兜底，避免整个 task 因未定义直接失败。

**延伸考点：** `-e` 传入的变量为什么能覆盖 set_fact？facts 缓存在什么规模下才值得开启、有什么一致性风险？

---

### Q7. 把 Nginx 配置里的 20 处硬编码改成模板，Jinja2 一渲染就出错，怎么调试？

**问题：** 你接手一套 30 个 Nginx vhost 的配置，以前每台机器手改，现在想统一用 template 模块渲染。第一次渲染就发现变量全空、有的行多出空格、循环生成的 server 块顺序不对。请讲讲 Jinja2 在 Ansible 里的正确打开方式：模板语法、过滤器、条件与循环，以及渲染出错的排障手段。

**期望加分项：**
- 讲清 template 模块的用法：`template: src=xxx.j2 dest=xxx`,变量用 `{{ }}`，控制逻辑用 `{% %}`，注释用 `{# #}`，两者不要混用
- 熟练演示过滤器链：`{{ port | int }}`、`{{ ips | join(',') }}`、`{{ name | lower }}`、`{{ var | default(x) }}`、`| to_json`/`| b64encode` 等，说清过滤器是纯函数式转换
- 循环典型场景：`{% for vhost in nginx_vhosts %}` 生成多段 server 块，`{% if %}`/`{% endif %}` 控制条件渲染，`{% endfor %}` 闭合
- 会处理"空列表渲染出多余内容"：用 `{% if vhosts is defined and vhosts %}` 或 `| default([])` 包一层
- 排障方法：`ansible -m template` 不行就本地用 `ansible localhost -m template -a 'src=... dest=/tmp/out'` 渲染看产物、或直接在模板里 `{{ var }}` 前加 debug；模板错误信息会精确到行号，用它定位
- 提醒安全点：模板里不要执行任意外部命令（Jinja2 不默认允许）、不要渲染未经转义的用户输入进 shell 上下文、`{{ lookup('pipe', ...) }}` 之类的能力要谨慎使用

**减分项：**
- 分不清 `{{ }}` 与 `{% %}`，把 if/for 写进插值
- 不知道过滤器机制，遇到大小写/拼接/默认值全靠模板里硬写逻辑
- 不会处理变量未定义或空列表，一渲染就崩
- 排障全靠"瞎改重试"，不知道本地渲染调试的手段
- 忽略模板内容的安全边界，把用户可控输入直接渲染进可执行文件

**解答：**

Jinja2 模板的三类语法必须一眼分清：`{{ expr }}` 是插值（把变量/表达式的值渲染出来），`{% stmt %}` 是控制语句（for/if/set 等），`{# #}` 是注释。Nginx 多 vhost 的典型模板是这样组织的：外层 `{% for vhost in nginx_vhosts %}` 包住整段 server 块，块内用 `{{ vhost.server_name }}`、`{{ vhost.listen_port | default(80) }}` 取属性，`{% if vhost.ssl %}` 决定是否渲染 443 段，最后 `{% endfor %}` 收尾。过滤器是处理"渲染前要不要加工"的唯一正确手段：端口可能从 inventory 里读成字符串，`{{ vhost.port | int }}` 转整数；多 IP 列表 `{{ vhost.backends | map('regex_replace', 'http://', '') | join(',') }}` 生成 upstream 一行。两个高频坑先说在前头：一是变量未定义，整个模板渲染直接失败——防御写法 `{{ nginx_port | default(8080) }}` 或列表类 `| default([])`；二是空列表，`{% for %}` 对空列表不报错但会留下一个空 server 块，模板里加 `{% if vhosts %}` 包一层即可。排障方法论：模板报错会精确到"文件第几行第几列、具体哪个表达式未定义"，先读报错别重试；然后 `ansible localhost -m template -a 'src=nginx.conf.j2 dest=/tmp/nginx.conf'` 把渲染产物落地到本地，`cat /tmp/nginx.conf` 直接看渲染结果，比在远程机器上反复试高效得多；最后再 `nginx -t` 验证语法。安全上有两条红线：模板只做展示层转换，不要在模板里调用 `lookup('pipe', ...)` 执行任意命令（Jinja2 默认也不允许直接执行）；任何来自用户输入/URL 参数的内容进入模板前必须转义，防止注入到 shell 或配置语法里——这属于面试里区分"会用"和"用得安全"的关键点。

**延伸考点：** template 渲染结果和 copy 一样会做校验和判断，那"模板文件更新但渲染结果不变"时会触发 handler 吗？`| to_json` 在生成配置文件时和直接写 JSON 模板的取舍是什么？

---

### Q8. 同一个 task 要在"部分机器跳过、部分机器重复执行"，when 和 loop 怎么配合？

**问题：** 你要给一批机器装监控探针：Nginx 机器装 nginx_exporter、Redis 机器装 redis_exporter，而且探针可能要装多个（有的机器同时有多个 Redis 实例）。同一份 playbook 里怎么用 when 和 loop 表达这种差异？block 在什么场景下有用？遇到"列表为空导致任务报错"怎么处理？

**期望加分项：**
- 讲清 when 是"按主机条件过滤"：`when: "'nginx' in group_names"` 或按事实/变量判断，被跳过的主机显示 skipped
- 讲清 loop 是"对列表逐项执行"：`loop: "{{ redis_instances }}"`，循环内用 `item` 引用当前项，可以 loop 套 when 做组合筛选
- 给出组合方案：`loop: "{{ exporters }}"` + `when: exporter 对应当前主机角色`，一个 task 兼容多类机器；或拆成多个 task 各自 when，讲清两种写法的取舍（可读性 vs 精简）
- 演示 block 的两种用法：`block + when` 把多步逻辑包成组（整组跳或整组跑）；`block/rescue/always` 做 try/catch 式的失败处理
- 处理空列表：loop 遇到空列表默认直接跳过不报错（老版 with_items 也类似），但"动态生成的列表未定义"会炸——用 `| default([])` 兜底，或用 `loop: "{{ list | default([]) }}"`
- 会提到 `loop_control` 与 loop 的 item 嵌套命名（loop_var），多循环嵌套时避免 item 冲突

**减分项：**
- 只会用 when 写死主机名，不会用 group_names / 变量做通用判断
- 不知道 loop 和 with_items 的关系与区别（loop 用列表表达式、可嵌套、更现代）
- 说不出 block/rescue/always 的存在
- 空列表/未定义列表导致 playbook 直接失败，没有兜底手段
- 多层循环嵌套时不会用 loop_var 重命名 item，变量互相覆盖

**解答：**

when 和 loop 解决的是两个正交问题：when 决定"这台机器要不要做"，loop 决定"要做的话做几遍"，两者可以叠在同一个 task 上。典型写法：`- name: 安装 redis_exporter\n  apt: ...\n  loop: "{{ redis_instances }}"\n  when: "'redis' in group_names"`——机器不在 redis 组直接 skipped，在的话对每个实例循环执行。判断主机归属除了 `group_names`，还可以用 `ansible_facts['os_family']`、自定义变量或 facts 里的字段，原则是"用事实判断而非硬编码主机名"，这样新机器加入组后 playbook 无需改动。当循环逻辑复杂到"不同角色装不同包"，我倾向拆成两个 task 各自 when，虽然行数多点但输出更清晰、排障时一眼看到是哪类机器；一个 task 里写复杂映射（如 exporters 列表带 target 字段再 when 过滤）适合列表结构规整的场景，可读性取决于数据设计。block 的价值在于"整组复用条件"和"失败兜底"：`block: ... when: ...` 让 block 内所有 task 共享同一个 when，不必每行都写；`rescue: ...` 捕获 block 中任意 task 的失败执行补救逻辑、`always: ...` 无论成败都执行清理动作——这是实现"出问题自动回滚"的最小骨架，比逐 task 写 ignore_errors 靠谱得多。空列表是个经典坑：`loop` 对空列表是安全的（直接跳过），但列表变量未定义会直接报 undefined——`loop: "{{ redis_instances | default([]) }}"` 一行解决；更隐蔽的是循环体内引用了 item 的属性，列表里某一项缺该字段照样炸，这时配合 `item.field | default(...)` 防御。最后，嵌套循环时默认 item 会被内层覆盖，记得给外层加 `loop_control: loop_var: outer_item`。

**延伸考点：** `block/rescue/always` 和 `any_errors_fatal`、`ignore_errors` 在失败传播语义上有什么本质区别？`with_items` 和 `loop` 在"传列表的列表"时行为有什么不同？

---

### Q9. 改完配置服务没重启，你写的 handler 为什么没触发？

**问题：** 你在 playbook 里用 template 生成 Nginx 配置，配了 `notify: reload nginx`，但执行完发现配置文件确实是新的、服务却没重载。请解释 handlers 的完整机制：什么时候触发、什么时候不触发、怎么强制触发？"监听"模式（listen）解决什么问题？

**期望加分项：**
- 讲清 handlers 的触发条件：只有被 notify 的 task 产生 changed 结果时才触发，ok（无变更）不触发——这是幂等设计的核心约定，也是"没重载"最常见的原因
- 讲清执行时机：所有 handler 在 play 末尾（所有 task 跑完后）统一执行，且同名 handler 只执行一次；notify 可以发生在多个 task
- 会演示正确写法：`template: ...\nnotify: reload nginx` + handlers 段 `- name: reload nginx\n  systemd: name=nginx state=reloaded`，强调用 reload 而非 restart（restart 会断流量）
- 给出"配置变了但想确保生效"的兜底：`meta: flush_handlers` 在 play 中间强制执行 handler；或 `handlers` 里用 `listen: "nginx config changed"` 让多个 handler 被一个通知触发
- 能讲清多 host 语义：handler 每台主机独立触发，与 changed 状态绑定，不是"全局跑一遍"
- 会指出线上常见坑：notify 拼写与 handler name 不一致时静默不执行（Ansible 不报错）、task 里 ignore_errors 导致 changed 语义错乱、重启类 handler 和重启后检查的时序

**减分项：**
- 以为 notify 一定触发 handler，不知道"只有 changed 才触发"
- 以为 handler 是立即执行，说不清"play 末尾统一执行"
- 不知道 `meta: flush_handlers` 和 listen 的存在
- restart 和 reload 分不清，把优雅重载写成 restart 制造抖动
- 遇到"服务没重载"只会重复跑 playbook，不会分析 changed 状态

**解答：**

Handler 机制一句话：handler 是"被通知后才执行的任务"，而通知的触发条件是"上游 task 的结果是 changed"。你改了模板、文件确实写进去了，但 template 模块在校验和对比后发现内容没变化（比如变量没真变），结果就是 ok 而不是 changed，notify 就不会发出，服务自然不重载——这是"没触发"的头号原因，排查时先看执行输出的 changed/ok 分布。第二个原因是时机：handler 不会在 notify 时立即执行，而是等当前 play 的所有 task 跑完后统一执行，且同名 handler 在一轮 play 中只执行一次，即使被 notify 三次。这种延迟语义的好处是：多个 task 改同一个服务的配置，最后只重载一次，避免重复抖动。工程上的正确姿势是 reload 而非 restart：`systemd: name=nginx state=reloaded` 是平滑重载、不断连接，`restarted` 会中断现有请求——高流量时段一个 restart 就能引发一次事故。如果配置变更后希望"立即生效再继续后续任务"，在关键位置插入 `- meta: flush_handlers`，它会强制执行已通知的 handler 再继续。listen 模式解决"多个 handler 绑同一个事件"的扩展问题：task 里 `notify: "reload web stack"`，handlers 段里多个 handler 各自声明 `listen: "reload web stack"`，新增服务时只需加一个 listen 的 handler，不用改所有 notify 点。还有两个隐蔽坑：notify 名字和 handler 名字对不上时 Ansible 不报错、只是静默不触发，建议通知名与 handler name 保持一致；以及 handler 里用了 `restart` 且没有 `started` 兜底，服务本来没运行时 handler 反而把状态搞乱。

**延伸考点：** 一个 task notify 了 handler、但后续 task 失败导致 play 中断，handler 还会执行吗？handler 里的任务是否也要保持幂等？如何处理？

---

### Q10. 三套环境都要部署同一套 Nginx+Redis+监控，你怎么用 Roles 组织才能不复制代码？

**问题：** 你们有 dev/staging/prod 三套环境，都要部署"Nginx + Redis + 监控探针"这套组合，但每套环境的版本、端口、监听地址都不一样。你决定用 roles 重构，请讲清 roles 的标准目录结构、defaults 与 vars 的分工、以及 ansible-galaxy 的角色复用方式。

**期望加分项：**
- 默写 roles 标准结构：`roles/<name>/{tasks,handlers,defaults,vars,templates,files,meta}.yml 与目录`，讲清每个目录的职责（templates 存 .j2、files 存静态文件）
- 讲清 defaults/main.yml（默认值，优先级最低、可被一切覆盖）与 vars/main.yml（固定内部变量，优先级高）的分工：环境差异全部放 defaults + 上层覆盖，不要在角色内写死
- 演示角色在 playbook 里的调用：`roles: [nginx, redis, monitor]`，以及 roles 顺序执行；`ansible-playbook site.yml` 多角色编排
- 讲清三套环境的差异通过什么注入：host_vars/group_vars 按环境分组（`[dev]`/`[staging]`/`[prod]`），变量按组覆盖角色的 defaults
- 会提 ansible-galaxy：`ansible-galaxy install geerlingguy.nginx` 装社区角色、`requirements.yml` 声明依赖与版本、私有 galaxy/自建仓库的依赖管理
- 能说出 roles 的常见反模式：把环境特定值写进角色 vars、角色过大耦合业务逻辑、滥用 include_role 嵌套导致调试困难

**减分项：**
- 说不出 roles 的目录结构，或把 defaults 和 vars 混为一谈
- 环境差异写死在角色里，三套环境要维护三份角色
- 不知道 ansible-galaxy 的存在或只知道概念没用过
- 角色与 playbook 职责不清：把所有内容都塞进 playbook 的 tasks
- 忽略 meta/main.yml 的依赖声明（dependencies）功能

**解答：**

Roles 是 Ansible 的"标准化代码复用单元"，目录结构是约定大于配置：`roles/nginx/tasks/main.yml`（核心任务）、`handlers/main.yml`（本角色 handler）、`defaults/main.yml`（默认变量）、`vars/main.yml`（角色内部常量）、`templates/`（Jinja2 模板）、`files/`（静态文件）、`meta/main.yml`（依赖与其他角色元信息）。关键分工在 defaults 和 vars：defaults 定义"可被覆盖的默认值"，优先级在所有变量里最低——调用方在 group_vars、play vars、命令行都能覆盖它；vars 定义"角色内部不想被外部改的常量"，优先级高。三套环境的差异化做法是：角色里只用 defaults 暴露端口、版本、地址等参数，环境差异全部通过 inventory 的组织表达——dev/staging/prod 各建一组，group_vars 里各自覆盖 `nginx_port`、`redis_version` 等，同一套角色零拷贝跑三套环境。playbook 侧则是一层薄的编排层：`- hosts: nginx_servers\n  roles: [nginx, monitor]`，角色按声明顺序执行；跨角色共享的状态用 `include_role`/变量传递，而不是在角色内部隐式依赖别的角色目录。社区复用是角色生态的另一半：`ansible-galaxy install geerlingguy.nginx` 装通用角色，`requirements.yml` 用 `version:` 锁版本、实现可复现的依赖安装；注意社区角色的质量参差，装之前看维护活跃度与支持的 Ansible 版本，内部推广时优先沉淀自己的标准角色。反模式要主动避开：一是把环境 IP、密码写进 vars/main.yml，等于把三套环境写死；二是角色越写越胖，从"部署 Nginx"膨胀到"整机初始化"，角色应该保持单一职责。

**延伸考点：** 角色里 `meta/main.yml` 的 dependencies 声明的依赖角色，和 playbook 里显式列出两个角色，在执行顺序和变量优先级上有什么差异？`include_role` 与 `import_role` 的差异（动态 vs 静态）影响哪些能力？

---

### Q11. 50 台 Web 分批滚动升级，全部一起跑会有什么后果？

**问题：** 你要给 50 台生产 Web 服务器升级应用版本，按你的经验"全部并行跑"会造成请求闪断、负载不均，而"一台一台跑"又要 50 个轮次。Ansible 的并行机制（forks、serial、strategy）分别管什么？你怎么设计一个滚动更新策略？

**期望加分项：**
- 讲清三个参数的分工：`forks` 是"控制节点并发连接数"（默认 5），决定同一时刻最多几台机器在跑；`serial` 是"play 内分批跑"（如 `serial: 5` 或 `serial: "20%"`），每批跑完才进下一批；`strategy` 是"任务层面的推进方式"
- 讲清 strategy 差异：默认 `linear` 是所有主机同步推进（本批所有机器完成 task1 才进入 task2），`free` 是每台机器独立推进互不等待，`batch`/`debug` 是进阶选项
- 给出滚动更新方案：`serial: 5` + 先 `pre_tasks` 摘流量（如设置 LB 维护模式/注销节点）→ 升级 → `post_tasks` 验活（健康检查通过才恢复），失败时中止后续批次
- 会用 `max_fail_percentage` 控制"本批失败超过阈值就整个 play 中止"，配合 `any_errors_fatal: true` 避免脏批次继续扩散
- 量化权衡：serial 太小则总耗时 = 批次 × 单批耗时，太大则闪断窗口大；结合 LB 摘流量能力决定批次大小，通常 10%-20%
- 提到 `strategy: free` 适用场景：无依赖的独立机器（如缓存节点批量清理），但要承担"结果不一致、无法保证全组同状态"的代价

**减分项：**
- 把 forks 和 serial 混为一谈，说不清哪个管并发连接、哪个管分批
- 不知道 linear 是默认 strategy，也不知道 free 的存在
- 滚动更新不考虑流量摘除与健康检查，直接"升级完就完事"
- 不知道 max_fail_percentage / any_errors_fatal
- 没有量化概念，答不出批次大小怎么定

**解答：**

三个参数管的层次完全不同：forks 控制控制节点同时建立的 SSH 连接数（默认 5），是"并发天花板"；serial 控制一个 play 按"多少台一批"推进，每批全部完成后才开始下一批；strategy 控制的是任务级别——默认 `linear` 下，同一批内所有主机必须"同步"过完当前 task 才一起进入下一个 task，谁慢谁拖全组；`free` 下每台主机独立推进，快的先跑完，不互相等待。滚动升级的标准姿势是把三者组合：`forks: 20`（连接够用）+ `serial: 5`（每批 5 台，占总量 10%）+ 保持默认 linear 保证批次内状态一致。批次内容用 pre_tasks/post_tasks 包起来：pre_tasks 先把该批节点从负载均衡摘掉（注销注册中心、把 LB 权重置 0、或返回 503 维护页），再跑升级任务，post_tasks 做健康检查——`uri` 模块打本机健康端口、验证通过后把节点加回 LB；检查失败则该批标记失败。失败控制是滚动的灵魂：`max_fail_percentage: 25` 表示单批失败占比超过阈值整个 play 中止，避免"每批都坏还继续往下滚"；单台失败默认只影响那台主机，后续批次继续，这可能把坏配置扩散到全集群——所以升级类 play 我通常会加 `any_errors_fatal: true` 配合 serial，任何一台失败立即叫停，回滚再上。批次大小的量化逻辑：批次过大 → 单批内流量瞬时被摘掉太多，剩余机器扛不住；批次过小 → 总耗时线性上升。结合 LB 容量按 10%~20% 选，同时把"摘流-升级-验活-回流"这四步的耗时实测出来，才能给出总窗口预估——面试里能说出"serial 5、每批约 3 分钟、全量约 30 分钟"这种数字，说明真跑过生产滚动。

**延伸考点：** serial 和 strategy: free 组合会出现什么诡异现象？`run_once` 与 `delegate_to` 在滚动场景里怎么配合做"只执行一次的初始化"？

---

### Q12. 只修一台机器，全量 playbook 跑 10 分钟，Tags 和 --limit 怎么救你？

**问题：** 你的部署 playbook 里有 40 个 task：装包、建用户、写配置、启动服务、刷 cron。现在生产环境一台机器上的 cron 被手改坏了，你想"只重写这台机器的 cron、别动其他任何东西"，但又不想新建一个只干这一件事的 playbook。怎么用 tags 和命令行选项做到精确执行？

**期望加分项：**
- 讲清 tags 的用法：task 或 play 级声明 `tags: [cron]`，执行时 `ansible-playbook site.yml --tags cron` 只跑带该 tag 的任务、`--skip-tags` 反向排除；tag 支持组合和特殊 tag（always/never）
- 讲清 --limit 的用途：`--limit web-03` 只对指定主机跑，与 tags 正交（tags 管任务维度、limit 管主机维度）
- 讲清 --start-at-task 与 --step 的适用场景：从指定 task 名继续执行、逐 task 确认，适合"全量中途失败后的续跑"
- 设计 tag 规范的工程建议：按变更域分组（install/config/service/cron/security），多 tag 用逗号分隔，`tags: always` 用于"无论选什么 tag 都要跑的依赖任务"（如事实收集）
- 提醒坑：--tags 与 handlers 的关系（默认 handler 会被跳过，需用 include 或 `handlers: true` 相关配置）；tag 与 when 组合时的执行语义
- 会讲"标签不是万能"：过度打 tag 会让 playbook 维护成本上升，tag 规划要在写 task 时就同步定

**减分项：**
- 只知道 --tags 存在，说不清 --skip-tags、--limit、--start-at-task 的区别与组合
- 没有 tag 规划意识，40 个 task 零散打标，等于没打
- 不知道 --limit 只影响主机维度，以为它能筛任务
- 不知道 tags 与 handler、when 的交互坑
- 用"注释掉 task"或"复制 playbook"来绕过，完全没利用选择机制

**解答：**

三个命令行选项解决三个不同维度的问题：tags 选任务、--limit 选主机、--start-at-task 选起点，可以组合使用。先讲 tags：给 task 打 `tags: cron`，执行 `ansible-playbook site.yml --tags cron` 就只跑打标为 cron 的任务（未打 tag 的全部跳过，除非打了 `always`）；`--skip-tags security` 则跳过所有 security 标。`always` tag 是个隐蔽但重要的设计：事实收集、日志初始化这类"无论选什么任务都必须先跑"的依赖，打 `tags: always`，否则你单独 `--tags cron` 时会因为没有 facts 而报错。工程上 tag 要有规划：按变更域分（`install/config/service/cron/security/reboot`），写 task 时同步打标，别等 40 个任务写完再回头补——打标成本极低但收益极高，排障时"只重刷 cron"这种需求一句话就能精准执行。--limit 管的是主机维度：`--limit web-03` 只对 web-03 生效，和 tags 完全正交，`--tags cron --limit web-03` 就是"只修这台机器的 cron"。--start-at-task 用于长 playbook 中途失败后的续跑：`--start-at-task "重写 cron 配置"` 从该任务继续，前面已完成的不重跑——注意它依赖 task name 精确匹配，再次证明给 task 起规范名字的重要性。--step 则逐 task 交互确认，适合完全不熟悉的新 playbook 首跑。坑要提前讲：tags 默认不会连带执行 handlers（handler 若被打 tag 需要显式匹配；未打 tag 的 handler 在 --tags 下会被跳过，可能导致"配置改了但重载没跑"）；tag 和 when 同时存在时，两者是"与"关系，跳过任何一个都不执行。最后的忠告：tags 是手术刀不是瑞士军刀，tag 打得太细维护成本反而上升，保持每个 tag 对应一个明确的运维动作边界。

**延伸考点：** `--tags always` 之外，`never` tag 的使用场景是什么？`ansible-playbook --list-tags --list-tasks` 在审计和交接场景的价值？

---

### Q13. 密码和私钥明文躺在 git 里，你被安全团队点名了，怎么整改？

**问题：** 安全团队扫描发现你们的 playbook 仓库里有明文数据库密码、SSH 私钥、云密钥，要求立即整改。请讲一套完整的敏感数据治理方案：ansible-vault 怎么用、no_log 怎么配、随机密码怎么生成、密钥分发怎么落地？

**期望加分项：**
- 讲清 ansible-vault 三件事：`ansible-vault create/edit/view/reencrypt` 管理加密文件、`ansible-vault encrypt_string` 加密单个变量、`ansible-playbook --ask-vault-pass` / `--vault-password-file` / vault-id 多密钥支持
- 给出工程化落地：secrets 单独建文件（group_vars/all/vault.yml）只存敏感项，普通变量与加密变量分离，加密文件进 git 但密钥不进
- 讲清 no_log 的用法：对可能输出敏感信息的 task 加 `no_log: true`（模块输出、debug 输出都会被屏蔽），`loop_control.label` 隐藏循环 item 里的敏感字段
- 演示随机密码生成：`lookup('password', '/path/file chars=ascii_letters,digits length=16')` 首次生成并落盘、后续复用，避免"人肉定密码"
- 讲清密钥分发：SSH 私钥用 copy 模块 + `mode: 0600` 指定 owner、或 ssh_key 模块生成并授权；私钥内容本身经 vault 保护
- 提醒审计面：git 历史里的明文无法靠"删文件"抹掉，需 `git filter-repo` 清洗或轮换密钥；vault 密码本身要交给密码管理器/CI secret 而非再放 git

**减分项：**
- 只知道 `ansible-vault encrypt` 一个命令，说不出 encrypt_string、多 vault-id、rekey
- no_log 只会背概念，举不出实际配置片段
- 密码靠人肉指定然后写进 vars，没有生成机制
- 明文私钥直接 copy 分发，不设置权限和 owner
- 忽略了"历史 commit 里的明文"这个安全盲区

**解答：**

整改方案分四层。第一层是加密静态敏感值：`ansible-vault encrypt group_vars/all/vault.yml` 把整个文件加密（加密后文件头是 `$ANSIBLE_VAULT;1.1;AES256`），`ansible-vault encrypt_string 'p@ssw0rd' --name db_password` 只加密单个变量、便于把加密变量内联进普通文件；解密密码通过 `--vault-password-file`（文件模式权限 600）或 vault-id 多密钥（不同团队用不同密钥解锁不同文件）注入，运行 `ansible-playbook` 时带上即可。工程规范上，我把 secrets 集中到独立的 `vault.yml`，普通变量和加密变量分文件存，playbook 引用变量名而不是加密内容——这样 review 时明文逻辑可读、敏感值全程不可见。第二层是运行时防泄漏：给"执行会输出敏感信息"的 task 加 `no_log: true`（比如 shell 里拼了密码、copy 私钥、debug 打印连接串），模块输出整体屏蔽；循环里 item 含密码时用 `loop_control: label: "{{ item.name }}"` 只显示名字不显示值——这一步防的是 playbook 运行日志和 CI 控制台泄密。第三层是密码生命周期：用 `lookup('password', '/etc/ansible/secrets/db_pass chars=ascii_letters,digits length=16')` 首次生成随机强密码并落盘，之后每次引用同一文件内容不变——保证幂等且密码强度有底线，同时规避"人肉起名+写进仓库"的全链条错误。第四层是密钥分发：SSH 私钥经 vault 保护后在任务里 `copy` 到目标并强制 `mode: '0600'`、指定 owner，否则 sshd 会拒绝使用权限过宽的密钥。最后必须点破一个盲区：git 历史里的明文删除文件只是假整改，要么 `git filter-repo` 重写历史（团队可接受的前提），要么直接轮换所有已泄露的密码/密钥——我通常建议后者，历史清洗成本高且不可靠。

**延伸考点：** vault 文件在 CI（Jenkins/AWX）里怎么解锁而不暴露密码？`ansible-vault rekey` 与多 vault-id 在密钥轮换场景的配合？

---

### Q14. 手写脚本跑 50 次都不出错，为什么还要求"幂等"，你的 Playbook 怎么证明自己幂等？

**问题：** 面试官问你"Ansible 凭什么保证重复执行安全"，你举了模块幂等的例子，但他追问："你的团队里有人用 shell 模块写了 `echo xx >> file`，这根本不幂等，你怎么在设计层面保证整本 playbook 的幂等性？"请系统讲一遍：模块层面、任务层面、playbook 层面各怎么保证幂等，check mode 和 --diff 怎么用。

**期望加分项：**
- 讲清三层幂等：模块层（声明目标状态、比较后决定是否变更）、任务层（when 条件、changed_when 显式声明变更语义）、playbook 层（整体重复执行结果收敛到同一状态）
- 熟练演示 `changed_when` / `failed_when`：command/shell 没有固有幂等语义，需要自己声明——`changed_when: "'added' in result.stdout"`、`failed_when: result.rc != 0 or 'error' in result.stderr`，把输出解析成布尔
- 讲清 check mode 的机制：`--check` 让模块"预演"——有 check 支持的模块做干跑判断（如 file 报告 will create），shell/command 等不支持 check 的默认跳过、需要 `check_mode: no` 或配合 `changed_when` 处理
- 演示 `--diff`：配合 --check 看将要发生的文件级差异，`--check --diff` 是变更评审的黄金组合
- 会用 register + 条件把"非幂等操作"改造成幂等：先查状态再执行（如先 `command: psql -tAc` 查记录存在与否，when 条件决定是否插入）
- 主动说"幂等不是工具保证的，是作者意图的工程化表达"，能举自己线上把 shell 改成幂等的重构案例

**减分项：**
- 只停留在"模块自带幂等"层面，说不出 command/shell 为什么天然不幂等
- 不知道 changed_when / failed_when 的存在，或不知道它们怎么解析 rc/stdout
- 不知道 check mode 对不支持模块的处理行为（跳过 vs 报错）
- 用 `ignore_errors: true` 掩盖失败来"装幂等"，没有任何收敛逻辑
- 从没用过 --check --diff 组合

**解答：**

幂等是分层实现的，缺一层都会出事故。模块层：声明式模块（file/copy/template/apt/service）内置"查-比-改"逻辑，目标已符合就直接 ok，这是幂等的地基。但 command/shell 是例外——它们本质是"执行命令"，没有目标状态概念，重复执行结果取决于命令本身，所以任务层必须兜底：`changed_when` 把模块输出解析成"是否发生了实质变更"，比如 `shell: useradd foo` 配 `changed_when: "'already exists' not in result.stderr"`；`failed_when` 把"非零退出码但其实是成功"的边界救回来，如 grep 没匹配返回 1 时 `failed_when: false`。真正的工程难点是把"查询-判断-执行"三段式写成幂等任务：先 `register` 一个查询命令的结果（如 psql 查记录、id 查用户），再用 `when` 判断是否要执行变更操作——这是把任何非幂等命令改造成幂等的标准手法，也是"系统状态驱动"与"脚本过程驱动"的本质区别。playbook 层则靠执行模式来验证：`--check` 让支持 check 的模块预演（file 会报告 `would create`、template 会报告差异），不支持 check 的 shell/command 默认显示 skipped（若用 `check_mode: no` 会真执行，务必知道这个雷）；`--diff` 展示将要发生的文件内容级差异，与 --check 组合 `ansible-playbook --check --diff site.yml` 就是发布前的变更评审标准动作——CI 里可以把它作为门禁先跑一遍。实践纪律上，我把"所有 task 跑完必然收敛"作为 code review 的硬指标：新增 shell task 必须写 changed_when、必须能自证重复执行行为一致，谁提交不幂等的 task 谁负责解释。线上经验里，幂等最大的敌人是"时间相关的副作用"——追加日志、写时间戳、自增 ID，这类操作要么彻底避免，要么用目标状态反推（检查已存在则跳过）。

**延伸考点：** `--check` 模式下 handler 会被触发吗？配合 `check_mode: false` 的 task 会出现什么边界问题？为什么 CI 里常用"check 通过才允许 apply"的流水线模式？

---

### Q15. Terraform 刚建好一批 EC2，Ansible 立刻上去装软件，两者的边界在哪？

**问题：** 你们的 CI 流水线现在是：Terraform apply 创建云资源 → Ansible 在上面装软件改配置。面试官问"为什么不让 Terraform 用 user_data 全干了，或者干脆让 Ansible 自己调云 API 建机器？"请讲清配置管理与资源编排的本质区别，以及两者协作的标准模式。

**期望加分项：**
- 讲清职责边界：Terraform 管"基础设施生命周期"（创建/销毁/伸缩云资源、网络、存储，声明式+state 追踪），Ansible 管"机器内部状态"（装包、写配置、起服务、管用户，无 state、靠幂等）
- 讲清本质差异：Terraform 有 state 文件做资源追踪与增量 diff，能销毁重建；Ansible 不追踪资源生命周期，它的"状态"是目标机器上可收敛的期望值
- 给出协作模式：Terraform 产出 inventory（如 `terraform output` 生成 hosts 文件、或动态 inventory 插件直读 terraform state）→ Ansible 消费；或 user_data 只做 bootstrap（装 python、放 ssh key）剩下交给 Ansible
- 指出 user_data 全干的缺陷：不可重复执行（重启不再跑）、无幂等、无编排顺序、日志难审计——所以 bootstrap 之外都用 Ansible
- 讲清"Ansible 调云 API"的反模式：有 `amazon.aws` collection 能建 EC2，但那等于重复造 Terraform 的轮子，且丢了 state、丢了团队协作基础
- 会提 drift 处理：Terraform 检测基础设施漂移（有人手改安全组）、Ansible 收敛机器内配置漂移，两者配合覆盖漂移全集

**减分项：**
- 说不出 state 在两者中的关键差异（Terraform 有、Ansible 无，以及这带来什么能力差异）
- 以为 user_data 可以替代配置管理，说不清 bootstrap 与配置管理的分工
- 不知道 Terraform 和 Ansible 之间怎么传 inventory（terraform output / 动态 inventory 插件）
- 建议用 Ansible 建云资源却说不清为什么这是反模式
- 完全没有"漂移"概念，答不出两者在漂移处理上的分工

**解答：**

一句话边界：Terraform 回答"世界上该有哪些资源"，Ansible 回答"机器里该是什么状态"。Terraform 是资源编排（orchestration）：它维护 state 文件追踪每个云资源（EC2、VPC、SG、RDS）的 ID 与属性，plan 时对比 state 与配置做增量 diff，apply 时创建/修改/销毁，destroy 能整批回收——它天然具备"资源生命周期"能力。Ansible 是配置管理（configuration management）：没有 state 文件，它的契约是"把目标机器收敛到描述的状态"，重复执行收敛到同一结果，但它不追踪"这台机器是不是我建的"——所以它不适合创建/销毁资源。因此标准模式是接力：Terraform 负责把基础设施建好并输出 inventory——`terraform output` 导出 IP 清单生成 hosts 文件，或者更优雅地直接用动态 inventory 插件（如 `aws_ec2`，甚至可以配 `cloud.terraform` inventory 插件直读 tfstate），把"哪几台机器"这个事实交给 Ansible；Ansible 接棒做机器内的一切：装包、模板配置、启动服务、建用户。user_data 的定位要讲透：它是 bootstrap（开机引导），适合做"装 python、注入 Ansible 用的 SSH key、注册到 inventory"这三件一次性的事，但绝不能承担完整部署——因为 user_data 只在首次启动执行、重跑不生效、无幂等、无顺序控制、失败无重试逻辑，用它做配置管理等于回到 shell 脚本时代。反过来，Ansible 的 `amazon.aws.ec2_instance` 模块能建机器，但在有 Terraform 的团队里这是反模式：没有 state 就没有"这台机器是谁管的"记录，destroy 无从谈起，并发与依赖图管理也全面退化。最后补上 drift 的分工：有人手改安全组 → Terraform plan 报漂移；有人手改 nginx.conf → Ansible 重跑收敛。两把扫帚各扫一段，这就是 IaC 体系里最经典的配对。

**延伸考点：** 混合云（AWS+自建机房）场景下，Terraform 管不到的裸金属怎么纳入这套体系？`cloud.terraform` inventory 插件直读 tfstate 有什么隐患（比如 destroyed 资源没刷新）？

---

### Q16. 给新业务线写一套"上线即用"的部署 Playbook，Nginx+应用+cron+内核参数，怎么组织？

**问题：** 你负责给一条新业务线写"一键上线"的 playbook：装 Nginx 与应用依赖、部署应用包、建应用用户、配 systemd 服务、加 cron 清理日志、调内核参数、最后优雅重启让流量无缝切换。请讲出完整的 playbook 结构设计和每步的关键技术点，说明"上线"与"变更"如何共用同一套代码。

**期望加分项：**
- 给出分层结构：site.yml（编排入口）→ 环境变量注入 → 各 roles（nginx/app/cron/sysctl）按序执行，pre/post 里处理停服窗口与验活
- 讲关键步骤的模块与参数：建用户 `user`（system: yes、home、shell）、systemd 单元用 `template` 生成 + `systemd: daemon_reload`、cron 用 `cron` 模块（name/minute/job）、内核参数用 `sysctl` 模块（持久化 /etc/sysctl.d）
- 讲"优雅重启/无缝切换"的工程细节：先 `pre_tasks` 停流量（LB 摘除）→ 部署（copy/template + notify reload）→ 健康检查（uri/command 验证端口与接口返回码）→ 才放量；而非直接 restart
- 讲"上线与变更同构"：同一 playbook 重复执行即"变更"，上线只是首次执行——靠幂等模块与 handler 完成，新版本包 copy 上去文件差异触发 reload，无差异则 ok 不动
- 讲验证步骤：`--syntax-check` → `--check --diff` → 灰度（--limit batch1 + serial）→ 全量，每个阶段有明确退出条件
- 量化指标意识：总时长预估、变更窗口、失败回滚预案（备份旧包/配置，`ansible.builtin.copy backup: yes`）

**减分项：**
- 只有"安装→配置→启动"三步，没有流量摘除、健康检查、回滚设计
- systemd 服务改了 unit 文件不 daemon_reload、cron 任务重复添加、sysctl 不持久化——典型半吊子细节
- 用 restart 代替 reload 做发布，不考虑平滑切换
- 上线与变更维护两套代码，改动不同步
- 没有 dry-run/check 意识，直接往生产上冲

**解答：**

"一键上线"的 playbook 应该是一棵结构清晰的树：入口 `site.yml` 只做 hosts 圈选与角色编排（nginx → app → cron → sysctl），环境差异全部由 group_vars 注入，业务逻辑零拷贝。每个角色的关键点：建应用用户用 `user` 模块（`system: yes` 建系统用户、指定 home 与 shell），把"弱权限运行"做成默认；Nginx 与应用的 systemd unit 用 `template` 渲染（进程数、内存、JVM 参数都是变量），生成后必须 `systemd: name=xxx daemon_reload: yes`，否则 unit 改了不生效——这是高频翻车点；cron 用 `cron` 模块声明式管理，同名任务幂等更新，绝不用 shell 拼 crontab；内核参数用 `sysctl` 模块，带 `sysctl_file: /etc/sysctl.d/99-tune.conf` 持久化，重启不丢。优雅发布的核心是"摘流→部署→验活→回流"四步：pre_tasks 里从 LB 摘掉节点（或切维护页），部署阶段 copy/template 应用新版本（`copy` 支持 `backup: yes` 留旧包兜底），配置变更走 notify handler 做 `reloaded`（平滑、不断连接）而非 restart，post_tasks 里用 `uri` 打健康接口验证返回码，通过才加回 LB；任一步失败则本批次中止、人工或脚本回滚。上线与变更共构是这套设计最大的红利：playbook 对"首次上线"和"日常变更"是同一份代码，重复执行时模块发现无差异返回 ok、有差异才 changed 触发 reload——版本升级的本质就是"copy 了新包 → 文件 checksum 变了 → handler reload → 验活通过"。最后是发布流程的门禁：`--syntax-check` 过语法、`--check --diff` 看变更清单、`--limit batch1` 灰度一批验活、再 serial 滚动全量，每一步都有明确退出条件，任何一步失败都先回滚再排查——这套流程本身就是"发布工程师"和"会写脚本的人"的分水岭。

**延伸考点：** 应用需要先备份数据库再升级、且备份失败必须中止发布，这个"前置条件"用 block/rescue/always 怎么写？灰度批次验活通过后，如何用 Ansible 自动把节点加回 LB（要考虑幂等）？

---

### Q17. Playbook 在生产环境报 `permission denied`，你怎么把问题定位到根因？

**问题：** 你的 playbook 在测试环境跑得干干净净，一上生产就报错，错误各不相同：有的机器报 `permission denied`、有的报 `ModuleNotFoundError`、有的卡在 SSH 连接超时。请梳理一套"从执行输出到根因"的排查方法论：语法检查、dry-run、详细日志、权限、SSH 五个维度分别怎么查。

**期望加分项：**
- 给排查顺序：先 `--syntax-check` 排除 YAML/语法问题（报错会精确到文件行），再 `--check` 干跑排除逻辑问题，最后才真执行；真执行用 `-v/-vvv` 分级看输出
- 讲 become 权限排查：`permission denied` 九成是提权问题——检查 `become: yes`、`become_user`、`ansible_become_pass`/sudo 免密配置、目标机 sudoers 规则；用 `ansible <host> -m shell -a 'id' --become` 单独验证提权链路
- 讲 SSH 排查：`-vvv` 看 SSH 层输出定位是密钥拒绝、主机指纹（host key checking）还是网络超时；`ssh -i key user@host` 手工验证连接与密钥权限（私钥 600、authorized_keys 权限、selinux 上下文）
- 讲模块执行失败的分析：错误信息里的 `rc`、stdout/stderr 才是答案——比如 `ModuleNotFoundError: No module named ...` 说明目标机 python 环境缺依赖（如 python3-apt），需在 play 里装依赖或用 `ansible_python_interpreter`
- 会讲 Python 解释器问题：`ansible_python_interpreter` 显式指定、目标机 python2/3 混用导致模块语法不兼容
- 能举自己线上排障的真实案例（哪一步定位到根因、花了多久）

**减分项：**
- 只会贴报错、不会把报错拆成 rc/stdout/stderr 分析
- 遇到 permission denied 不验证 become 链路就瞎改权限
- 不知道 -vvv 能看 SSH 层、不会手工 ssh 验证
- 不知道 ansible_python_interpreter 的存在
- 不看 --syntax-check 直接把语法错误当成执行错误

**解答：**

排障要按"从输入到执行"的层次推进，每一层都有对应的工具。第一层语法：`ansible-playbook --syntax-check site.yml`，YAML 缩进错、模块参数错、when 表达式错都会在这一步精确报出文件和行号——很多人跳过这步直接跑，把语法错误混进执行错误里，徒增噪音。第二层逻辑：`--check` 干跑，看哪些 task 会 changed、哪些 skipped，逻辑错误（when 条件不生效、handler 没配对）在这一层暴露。第三层真执行：输出用 `-v`（模块返回值）到 `-vvv`（含 SSH 层的完整传输与连接日志）分级，默认输出只有结果没有原因。四类高频故障的定位法：一是 `permission denied`，先别怀疑权限文件，先验证提权链路——`ansible web-03 -m shell -a 'id' --become`，能跑通说明 become 没问题、是任务内具体命令缺权限；跑不通就查 sudoers（`ansible_become_pass` 没配、NOPASSWD 没设、become_user 的用户不存在）。二是 SSH 层问题，`-vvv` 里能看到具体拒绝原因：host key 校验失败（`Host key verification failed`，新机器未加入 known_hosts）与密钥拒绝（`Permission denied (publickey)`，私钥权限大于 600 或未加入 ssh-agent）是两种完全不同的修法；手工 `ssh -i key -p port user@host` 复现是黄金验证手段。三是模块执行失败，答案在 `rc`、stdout、stderr 三个字段：`rc=1` + stderr 里 `ModuleNotFoundError: No module named 'apt'` 说明目标机 python 环境缺 `python3-apt`——这类问题要么 pre_tasks 先装依赖，要么 `ansible_python_interpreter: /usr/bin/python3` 显式指定解释器（CentOS 老机器默认 python2，模块语法直接崩）。四是超时：网络层隔离、跳板机（需要 `ProxyCommand`）、SSH 连接复用参数没配导致握手过慢。方法论上记住：把报错当证据拆开看（rc/stdout/stderr/ssh 日志各记什么），而不是整段贴出来等人翻译——这是排障能力的分水岭。

**延伸考点：** `ansible_failed_task` 和 `debug` 模块怎么组合做"失败现场留档"？同一报错在 `--check` 下正常、真执行失败，最可能是哪类原因？

---

### Q18. 全量跑一次 30 分钟，老板嫌慢，怎么在不改业务逻辑的前提下把执行时间砍一半？

**问题：** 你们的 playbook 有 300 台机器、40 个 task，全量跑一轮要 30 分钟，每次发布都被人吐槽。老板说"业务逻辑别动，把时间砍下来"。你会从哪些层面优化？请重点讲 SSH 连接复用（ControlPersist）、Pipelining、facts 收集裁剪和并行度。

**期望加分项：**
- 讲清最大瓶颈是"每 task 一次的 SSH 连接 + 文件传输 + facts 收集"，优化优先级：pipelining > ControlPersist > forks > facts 裁剪
- 讲 pipelining 的机制：`pipelining = True` 把模块代码经 SSH 标准输入直接送入目标机 python，省掉"传临时文件→执行→删文件"的两次往返，能省 30%-50% 连接开销；注意前提是 sudo 需要 `requiretty` 关闭（RHEL 默认开，踩坑率高）
- 讲 ControlPersist：SSH 长连接复用，`ssh_args = -o ControlMaster=auto -o ControlPersist=60s`，同一主机多次连接复用同一 TCP 通道，任务级往返从"建连+握手"降为"复用"
- 讲 forks 调优：默认 5 太小，300 台机器建议 20-50，受控制节点 CPU/网络与目标机并发能力约束，给出压测依据（看控制节点负载与单机 sshd 进程数）
- 讲 facts 裁剪：`gather_facts: no` + 需要时 `gather_subset`（如 `!hardware,!network`）；或开 facts 缓存（jsonfile 插件配 cache 目录，按需失效）——facts 收集在大规模下往往占总耗时 30%+
- 量化意识：说出优化前后数字（如 30 分钟 → 12 分钟），用 `time ansible-playbook` 与 `-vvv` 观察每 task 耗时定位剩余瓶颈

**减分项：**
- 只知道"加大 forks"，说不清连接复用的机制
- 不知道 pipelining 的存在，或不知道它和 sudo requiretty 的冲突
- 不考虑 facts 裁剪/缓存，让 300 台机器每轮全量收集
- 优化不验证、不量化，给不出前后对比
- 忽略 ControlPersist 的坑：连接复用导致内存/句柄积累、长任务偶发断连

**解答：**

30 分钟里大头是"每 task 一次 SSH 往返 + 每 play 一次全量 facts 收集"，业务逻辑不用动，动的是传输层和收集策略。按性价比排序：第一是 `pipelining = True`（ansible.cfg 的 `[ssh_connection]` 段）：默认 Ansible 是把模块文件先 scp 到目标机再执行再删除，三步两次往返；开启后模块代码直接通过 SSH 标准输入管道喂给目标机 python，每 task 省掉 60%-70% 的连接开销——300 台 × 40 task，光这一项通常能砍 30% 以上。踩坑点必须提：RHEL/CentOS 的 sudo 默认 `requiretty`，pipelining 下 stdin 被占用会报 `sudo: sorry, you must have a tty`，需要关闭 requiretty 或改用免 sudo 策略。第二是 ControlPersist：`ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s` 让同一主机的 SSH 连接在任务间复用（保持 60 秒空闲存活），把"每 task 建连+密钥协商"降为"复用已有通道"，加上 `-C` 压缩传输还能进一步省带宽。第三是 forks：默认 5 意味着同屏最多 5 台机器在跑，300 台就是 60 个"5 台批次"的串行——调到 20~50 直观见效，但别无脑拉满：控制节点 CPU/内存、网络带宽、目标机 sshd 并发、云厂商 API 限流都会成新瓶颈，用 `top` 观察控制节点、`ps -ef | grep sshd | wc -l` 观察目标机来定上限。第四是 facts 收集：默认每 play 每台机器跑一轮 setup（内存、网卡、OS 全查），300 台往往占总耗时 30%+——`gather_facts: no` 关掉，只有需要 facts 的 play 才开，且用 `gather_subset: '!hardware,!network'` 只取需要的子集；再进一步配 facts 缓存（`fact_caching = jsonfile` + `fact_caching_connection = /tmp/ansible_facts`），非敏感场景下 facts 按时间失效复用。最后一定要量化：`time ansible-playbook site.yml` 记录基线，每调一项重测，把 30 分钟压到 10 分钟级别是正常水平——能报出"pipelining 省 8 分钟、forks 从 5 到 30 省 6 分钟、facts 裁剪省 5 分钟"这种分项账，面试官就信你真的调过。

**延伸考点：** ControlPersist 在长任务（单 task 执行几分钟）场景下偶发"连接断开"的成因和规避？pipelining 开启后 `become` 与 `sudo -u` 组合还有哪些隐藏约束？

---

### Q19. 安全团队要审计 Ansible 的权限模型：一台堡垒机怎么管 300 台生产机？

**问题：** 安全团队要求：Ansible 控制节点不能持有 root 密钥直连生产机、操作必须可审计、生产机上的提权路径要收口。你现在的实现是"控制节点用 root 的 SSH 密钥免密登录所有机器"，被直接打回。请设计一套满足最小权限 + 可审计 + 可轮换的提权与密钥方案。

**期望加分项：**
- 讲清最小权限的层次：控制节点用专用运维账号（非 root）SSH 登录 → 目标机上该账号无 root 权限 → 具体任务才 `become: true` 提权，提权口令/方式独立于 SSH 登录凭据
- 讲 SSH 密钥管理：专用运维用户各自的密钥、`ansible_ssh_private_key_file` 指定、密钥存 vault 或 CI secret 而非明文；`authorized_key` 模块管理目标机公钥；密钥定期轮换（cron/playbook 双写替换）
- 讲提权收口：目标机 sudoers 用 `NOPASSWD` 或限命令白名单（如 `ansible ALL=(ALL) NOPASSWD: /usr/bin/systemctl, ...`），`become_user` 最小化；禁 root 直接 SSH（`PermitRootLogin no`），避免"控制节点 = 全集群 root"
- 讲免 agent 的信任模型：信任边界是"SSH 密钥 + sudo 规则 + 控制节点安全"，因此控制节点本身要加固（防火墙、访问控制、审计），这是无 agent 架构必须承担的责任
- 讲审计闭环：`ansible.cfg` 开 `log_path` 或接 AWX/AAP 记录每次执行的 playbook/主机/结果；执行账号按人隔离（每人一把 key），不共钥，出事能追到人
- 会提 SSH 配置层加固：known_hosts 校验（`host_key_checking`）、禁用密码登录、`ssh_args` 里限制算法/超时

**减分项：**
- 认为"root 密钥直连"没问题，讲不出信任边界的代价
- 分不清 SSH 登录身份与 become 提权身份，把两者混为一谈
- sudoers 一刀切全放行或全 NOPASSWD，没有白名单意识
- 没有按人分钥/审计意识，出事无法追责
- 忽视控制节点本身的防护，只盯目标机

**解答：**

这套设计要回答"信任边界在哪"——无 agent 架构下，控制节点是天然的信任根，所以控制节点加固 + 凭据最小化 + 审计闭环三件事缺一不可。第一层是登录身份：控制节点上每人用专用运维账号（比如 `ops`），SSH 密钥按人分发（`authorized_key` 模块管理目标机上的公钥），执行时通过 `ansible_ssh_private_key_file` 或 ssh-agent 指定；绝对禁止 root 密钥直连，目标机 `PermitRootLogin no` 把 root 远程登录堵死。第二层是提权收口：ops 账号默认无特权，需要 root 的任务才加 `become: true`，提权通道与 SSH 登录通道分离——sudoers 不要全开 `NOPASSWD: ALL`，按需白名单（如应用发布账号只允许 `systemctl`、`cp`、`chown` 等具体命令），`become_user` 尽量用目标服务账号而非 root；这样即使某台机器被攻破，攻击者拿到的是受限的 sudo 能力，而非整个集群的 root。第三层是密钥生命周期：私钥不进 git、存 vault 或 CI 的 secret 管理（对应 Q13 的治理体系）；轮换用 playbook 自身完成——新钥写入 → 验证新钥可用 → 移除旧钥，全流程幂等可回滚；定期轮换频率要写进制度（如每季度）。第四层是审计：`ansible.cfg` 里 `log_path = /var/log/ansible/ansible.log` 记录执行人（LOGNAME）、playbook、主机与结果；更完整的做法是上 AWX/AAP——它自带 job 级审计、RBAC（谁能对哪些主机跑哪些 playbook）、操作留痕，安全审计报告直接导出。最后别忘了无 agent 的另一面：因为受管节点不装 agent、没有本地策略执行器，机器上的安全依赖"SSH 访问控制 + sudo 规则 + 文件完整性"，所以目标机的 `sshd_config` 加固（禁密码、禁 root、限来源 IP）也是这套方案的组成部分。能把这五层讲全并给出 sudoers 片段和轮换流程，说明你真在生产上扛过安全评审。

**延伸考点：** 如果运维账号的 SSH 密钥在目标机上被泄露，最小权限设计能把损失限制在什么范围？`become_user` 与 `become_method`（sudo/su）在不同场景（如无法用 sudo 的容器）怎么选？

---

### Q20. 公司有几百个 shell 部署脚本要"现代化"，你怎么说服团队迁到 Ansible，落地节奏怎么排？

**问题：** 公司沉淀了几百个运维 shell 脚本，散落在各团队，很多脚本只有原作者看得懂，出过一次"脚本在 A 机器正常、B 机器跑挂"的事故。你被任命牵头"脚本现代化"，怎么定迁移优先级、怎么设计落地路径，以及最终要不要上 AWX/AAP 这类平台？

**期望加分项：**
- 讲迁移优先级的标准：按"影响面 × 风险 × 复用度"排序——高频变更、跨机器执行、影响面大的先迁；一次性脚本不迁（用 ad-hoc 或保留）；给出具体例子（如先迁"发布部署"再迁"日志清理"）
- 讲迁移的对应关系：shell 的 if 分支 → when/block、函数 → roles、变量定义 → group_vars/vault、`set -e` → any_errors_fatal/failed_when、注释 → task name，强调"迁移是语义重构不是翻译"
- 讲"先翻译再收敛"的节奏：第一批先用 command/shell 模块原样包住老脚本跑通全流程（行为不变、能审计）→ 第二批把高频模块替换成声明式模块实现幂等 → 第三批沉淀成 roles 进 galaxy 内部仓库
- 讲配套工程：代码进 git + 分支评审 + CI 里 `ansible-lint` + `--syntax-check` + `--check` 门禁；灰度与回滚机制（serial + tags + vault 密钥交接）
- 讲 AWX/AAP 的价值与代价：可视化 job 与审计（`ansible-playbook -i ... --user` 变 Web UI 点击）、RBAC 与凭证中心（SSH 密钥/vault 密码托管）、调度与 webhook 集成 CI/CD；代价是部署维护一套平台（k8s/docker）、学习成本、小团队可能过度建设
- 能给出分阶段路线图与量化验收：比如 3 个月迁 80% 高频任务、事故率下降、变更时长缩短，用数据证明价值

**减分项：**
- 建议"几百个脚本一夜全迁"，没有优先级与灰度概念
- 把迁移当成"用 shell 模块包一层"，没有语义重构（幂等、变量、角色化）
- 不知道 ansible-lint 或 CI 门禁，迁移完照样事故
- 对 AWX/AAP 只会说"可视化"，说不清凭证托管、RBAC、审计这些解决真实问题的能力
- 没有迁移节奏/验收标准，或否定平台价值但给不出理由

**解答：**

迁移的根本目标不是"把脚本变成 Ansible"，而是把"隐式知识"变成"显式、可评审、可复现的声明式资产"。优先级排序用"影响面 × 风险 × 复用度"打分：跨机器、高频、一出错就影响线上的先迁——通常"应用发布、配置变更、账号管理"排第一梯队；纯本机一次性操作不迁（ad-hoc 或保留脚本）；低价值脚本直接淘汰。迁移是语义重构，不是语法翻译：shell 的 `if [ -f x ]; then` 对应 `when` 或 `state: absent` 的声明；函数对应 roles 的 tasks 复用；`set -e` 对应 `any_errors_fatal` 和显式 `failed_when`；一堆 echo 注释对应 task 的 `name`。落地节奏分三批：第一批"包壳保行为"——用 `command`/`shell` 模块把老脚本原样跑起来，只解决"可编排、可审计、可批量"三个问题，行为不变、风险最小；第二批"幂等化"——把高频动作（装包、写配置、管服务、建用户）替换成声明式模块，消除"重复执行出问题"的顽疾；第三批"角色化"——沉淀成内部 roles，进自建 galaxy 仓库，形成团队标准库，新业务直接引用。工程配套要跟上：全部代码进 git、PR 评审、CI 里跑 `ansible-lint`（风格与反模式检查）+ `--syntax-check` + `--check --diff` 作为合并门禁；生产执行带 serial 灰度与回滚预案。最后谈平台：当"谁能在哪批机器上跑什么 playbook"开始成为治理问题时，就该上 AWX/AAP——它解决的恰好是 shell 时代完全没有的三件事：审计（每次 job 的执行人/内容/结果可追溯）、RBAC（运维/开发/审计角色权限分离）、凭证中心（SSH 密钥和 vault 密码托管在平台，不散落个人电脑）；CI/CD 集成后发布流程全部留痕。代价是平台本身的部署维护（AWX 在 k8s 上跑、AAP 要 license）与团队学习曲线，5 人以下团队往往 CI + ansible 命令行就够用。验收用数据说话：三个月内 80% 高频运维操作迁移完毕、相关事故率下降、单次发布平均耗时从 X 分钟降到 Y 分钟——用这个向老板证明"现代化"不是折腾，是降险降本。

**延伸考点：** AWX 的 credentials 与 vault 密码在"审计要求凭证不可见"下的实现机制是什么？如果团队坚持"脚本够用"，你怎么用数据论证迁移 ROI（事故成本、新人上手成本、变更耗时）？

---

### Q21. 一次 Playbook 批量执行误删线上文件，事故复盘与四道防线怎么设计？

**问题：** 周五傍晚你执行了一个"清理旧日志"的 playbook，本想删 /var/log/app 下 7 天前的 *.log，结果 find 正则写错把整个目录删了，服务大面积报错。复盘时你发现：这件事技术上"完全合法"地发生了——没有 check、没有备份、一台没漏地全量执行了。请梳理事故根因，并把 check、备份、灰度、权限四道防线逐条设计出来。

**期望加分项：**
- 复盘先定性再归因：把根因从"人写错正则"上升到"系统为什么允许一个错误的正则造成全量灾难"，逐层追问四道防线各自缺在哪
- check 防线：`ansible-playbook --check --diff` 作为生产执行前置门禁，CI 里强制跑；讲透 check 的边界——shell/command 模块在 check 模式下默认跳过（除非显式 check_mode），危险操作应尽量用声明式模块（file 的 state: absent）并配合 changed_when 人工审查
- 备份防线：删、改、覆盖三类操作执行前必须有可验证的备份——copy 带 `backup: yes`、删除前先 archive/fetch 落盘、数据库变更前先 dump；强调"备份了"不等于"能恢复"，恢复演练写进发布流程
- 灰度防线：serial 分批 + `--limit` 先单台边缘验证 + 每批后验活（uri 打健康接口），配 `max_fail_percentage`/`any_errors_fatal` 控制失败传播，杜绝全量一把梭
- 权限防线：危险任务收口——rm/truncate/覆盖类任务单独打 `tags: [destructive]`，执行命令必须显式 `--tags` 带出并二次确认，或默认 `check_mode` 开启；sudoers 按命令白名单，删文件权限不落到普通账号
- 事后沉淀：危险操作清单、CI 门禁、playbook 评审 checklist 三样固化产物，让"个人失误"变成"组织防线问题"

**减分项：**
- 复盘停在"我写错了"，不上升到流程与工具防线缺失
- 以为 --check 万能，不知道 shell 类模块在 check 下默认被跳过
- 备份=备份了，没验证可恢复性，也没有恢复演练
- 没有灰度与验活意识，仍是一次性全量执行
- 无事后措施（防线沉淀、规范更新），下次同样事故照发

**解答：**

复盘的第一原则：别把根因定在"人写错了正则"，而要问"系统为什么允许一个错误的正则造成全量灾难"。逐层审四道防线。第一道 check：任何 playbook 进生产前必须 `ansible-playbook --check --diff` 生成变更清单并人工审阅，CI 把它做成合并门禁；但必须认清 check 的边界——shell/command 模块在 check 模式下默认跳过（除非该 task 显式 `check_mode: no`），所以用 shell 拼 rm 的 playbook，check 根本拦不住。这也是为什么危险操作要优先用声明式模块：`file: path=... state=absent` 自带"目标状态"语义，配合 `changed_when` 让每次变更可预期、可审查。第二道备份：对"删、改、覆盖"三类动作，执行前必须有可验证的备份——copy 加 `backup: yes` 保留旧文件；删除目录前先 `archive` 打包到备份路径并把备份位置写进变更单；数据库变更前先 dump。关键纪律是"备份必须验证可恢复"：只执行备份动作、不做恢复演练，等于没有备份——事故里你会发现备份文件损坏或权限不可读。第三道灰度：`serial: 20%` 分批、`--limit` 先一台边缘机器试跑、每批后 `uri` 模块打健康接口验活，配合 `max_fail_percentage` 与 `any_errors_fatal: true`，任何一台异常立即中止，而不是 200 台同时中招。第四道权限：把 rm/truncate/覆盖类任务统一打 `tags: [destructive]`，执行命令强制写成 `ansible-playbook site.yml --tags destructive --limit <一台>` 的显式形式，并配二次确认；sudoers 按命令白名单收口，让"删全盘"这种操作在权限层就不可执行。事后恢复的预案要在事故前写好：旧文件回放（从备份恢复 + reload 服务）、服务回滚步骤、以及"先恢复再复盘"的次序。最后把教训固化成三样东西：危险操作清单（哪些 task 必须 check+备份+灰度）、CI 门禁（check/diff 必跑）、playbook 评审 checklist。至此，"写错正则"才真正从个人失误变成组织防线问题——这也是事故复盘该有的样子。

**延伸考点：** shell 模块为什么是 check 模式与备份体系的最大盲区（默认跳过 + 无目标状态）？如果被删机器上没有备份且无法恢复，正确的事后动作是什么（保留现场、止损、按证据复盘）？

---

### Q22. 上千台机器配置基线统一项目：Inventory 规划、基线 Playbook 与漂移治理怎么设计？

**问题：** 安全团队要求 1200 台 Linux 机器（Web/DB/缓存/大数据多类角色）统一配置基线：SSH 加固、内核参数、时区、NTP、日志轮转、公共安全基线。你牵头这个项目。请设计 Inventory 怎么规划、基线 Playbook 怎么分层、以及"漂移"如何检测与修复，并给出能向安全团队汇报的度量方式。

**期望加分项：**
- Inventory 规划：按角色建父组（web/db/cache/bigdata）+ 环境/机房子组，基线差异全部进 group_vars 分层承载，playbook 零硬编码；云上机器用动态 inventory 按 tag 自动入组，不手写 1200 台静态清单
- 规模意识：默认全量 gather_facts 在 1200 台上要二三十分钟——基线 playbook 用 `gather_facts: no` + 按需 `gather_subset`、开 facts 缓存、配 pipelining 与合理 forks，把一轮全量压进 10 分钟内
- 基线 Playbook 分层：拆成 base role（时区/NTP/内核参数/SSH 加固/日志轮转/公共用户），全部用声明式模块——sysctl 写 /etc/sysctl.d、lineinfile 管 sshd 参数并配 `validate: /usr/sbin/sshd -t %s` 防改错断连、template 生成 rsyslog/journald 配置、systemd 管服务；模块幂等，重跑只收敛差异
- drift 检测：两类手段——每日 `--check` + CI 收集"本应变更却未变更"的差异清单；或专门 audit playbook 逐条比对基线项（读文件内容/运行 sysctl 后与期望对比），输出每台机器的合规率报告
- 修复策略：检测与修复合一——校验失败即自动修复并 reload 对应服务；但必须先区分漂移来源，业务手工改动过的配置自动覆盖前必须告警走审批，否则基线工具本身会变成事故源
- 度量汇报：合规率（完全符合/部分符合/不符合）+ 漂移率趋势两个数字，每月出报告，让安全团队看到"数字在变好"而不是"跑过一遍"

**减分项：**
- 没提动态 inventory，也没意识到 1200 台的 facts 收集与执行性能问题
- 基线值写死在 playbook 里，没有参数化与分层覆盖
- 只有检测没有修复，或自动修复无告警、无灰度、无审批
- 改 sshd/sysctl 这类高危配置不评估影响、不留逃生通道（如第二个 SSH 会话）
- 没有合规率与漂移率的度量机制，汇报时拿不出数据

**解答：**

1200 台做基线统一，本质是"期望状态建模 + 持续收敛"，分四件事。第一件 Inventory：机器按角色分组（web/db/cache/bigdata 父组 + 机房/环境子组），云上机器用动态 inventory 按 tag 自动入组，杜绝 1200 台静态文件手写；基线差异全部进 group_vars——`[web]` 组覆盖 web 专属 sysctl、`[db]` 组覆盖 ulimit 与内核参数，playbook 里零硬编码，新机器打 tag 即自动纳入基线。规模是隐藏难点：默认每 play 对每台机器全量收集 facts，1200 台一轮要二三十分钟，所以基线 playbook 用 `gather_facts: no` + 按需 `gather_subset`（如 `!hardware,!network`）、开 facts 缓存、配 `pipelining = True` 与 forks 调优，把一轮全量压进 10 分钟以内——跑不动的东西没法天天跑，也就谈不上持续收敛。第二件基线 Playbook：拆成 base role（时区/NTP/内核参数/SSH 加固/日志轮转/公共用户），全部用声明式模块——sysctl 模块写 /etc/sysctl.d 持久化、lineinfile 管 sshd 参数且必须配 `validate: /usr/sbin/sshd -t %s`（防止一行参数改错导致 sshd 起不来、所有机器失联，这是基线项目最高危的坑）、template 生成 rsyslog/journald 配置、systemd 管 NTP 与日志服务；每个模块幂等，重跑只收敛差异，不产生噪音。第三件 drift 检测与修复：两条路径并行——日常跑专门 audit playbook，逐条比对基线项（读配置文件内容、执行 sysctl 后与期望值比对、检查服务 enabled 状态），输出每台机器的 ok/failed 汇总与合规率；再配合 `--check` 定期收集"应改未改"的差异清单。修复策略是"校验失败即自动修复"（file/lineinfile/sysctl 直接收敛 + reload 对应服务），但自动修复前必须区分漂移来源：如果是业务手工改动（比如 DBA 调过某个内核参数），无脑覆盖会引发冲突，正确做法是告警 + 审批后修复。第四件度量汇报：用合规率（完全符合/部分符合/不符合的机器数）与漂移率趋势两个数字，每月出报告——安全团队要的是"数字在变好"和"违规可追踪"，而不是"我们跑过一遍 playbook"。这套体系落地后，"新机器上线即合规"与"存量机器持续收敛"共用同一份基线代码，这就是配置基线项目的完整闭环。

**延伸考点：** 基线项目里"自动修复"与"告警审批"的边界怎么定（按风险等级/影响面）？改 sshd/sysctl 后是否需要重启生效，如何用 Ansible 管理"需重启标记"（如 /var/run/reboot-required）？

---

### Q23. 你们同时有裸机、云主机和 K8s 集群，Ansible 与 Terraform/K8s 的自动化体系怎么设计？

**问题：** 团队同时管理裸机与云主机（业务应用部署在其上）和 K8s 集群（容器化服务），你想构建一套"基础设施自动化"体系。面试官追问：Ansible、Terraform、Kubernetes 各自的职责边界在哪？Terraform 建好的资源怎么交给 Ansible？K8s 集群内的事为什么不该用 Ansible 做？请给出整体设计与流程编排。

**期望加分项：**
- 边界清晰：Terraform 管"基础设施资源生命周期"（云主机/VPC/安全组，有 state 可销毁重建）；Ansible 管"机器内状态"（OS 初始化/装软件/写配置/非容器应用，幂等收敛）；K8s 用声明式 API + Helm/Kustomize 管容器化工作负载，绝不用 Ansible 逐台 ssh 进节点去 kubectl
- 衔接点：Terraform `terraform output` 或 `cloud.terraform` 动态 inventory 插件把资源清单交给 Ansible；user_data 只做 bootstrap（装 python/注入公钥/注册 hostname）；K8s 集群本身（kubeadm init/join、容器运行时、CNI）是 Ansible 的活，集群就绪后一切工作负载管理交给 kubectl/Helm
- 反模式识别：Ansible 调云 API 建资源（有 amazon.aws collection 但等于重复造 Terraform 轮子、丢 state）；Ansible 管理 K8s 工作负载（kubectl 硬塞）——两者都要主动指出并说明代价
- 流程编排：CI/CD 流水线或 AWX/Terraform Cloud 作编排层——代码提交 → CI 校验（terraform fmt/validate、ansible-lint + --check、helm lint/template）→ terraform plan 人工确认 → ansible-playbook（serial 灰度）→ helm upgrade（--atomic）→ 自动化验证；讲清每层工具的校验动作
- 混合场景：裸机 Terraform 管不到，由 Ansible 承担 inventory 与初始化；云上资源 Terraform 管，机器内部 Ansible 管
- 回滚分层：资源层 terraform 回滚 state、配置层 Ansible 幂等重跑旧版本、应用层 helm rollback；跨层回滚顺序——先应用层（最快见效）再配置层最后基础设施层

**减分项：**
- 边界混乱：用 Ansible 建 EC2、用 Ansible 管理 K8s 工作负载，说不出为什么是反模式
- 不知道 Terraform → Ansible 的 inventory 衔接手段（output/动态插件）
- 把 K8s 当"更高级的机器"，设计"Ansible 逐台操作集群"
- 只有单工具视角，没有编排层设计与跨层回滚
- 说不清 user_data 为什么只能做 bootstrap

**解答：**

体系设计的第一原则是"按资源类型选工具，不重复造轮子"。三层职责各归其位：基础设施层由 Terraform 负责——云主机、VPC、安全组、负载均衡这类"资源生命周期"问题，它有 state 文件追踪资源、能 plan 做增量 diff、能 destroy 回收，这是 Ansible 替代不了的；机器内层由 Ansible 负责——OS 初始化、装软件、写配置、起服务、非容器化应用部署，这是 Terraform 的 user_data 替代不了的（user_data 只在首次启动执行、不幂等、无编排顺序）；容器化应用层由 Kubernetes 的声明式 API + Helm/Kustomize 负责——Deployment/Service/Ingress 的期望状态由集群自身收敛，绝不该用 Ansible 逐台 ssh 进节点去 kubectl（那等于把控制器该干的事手工做一遍，还丢了声明式与自愈能力）。衔接点是设计最容易出错的地方，三件事：① Terraform → Ansible 用 `terraform output` 生成 inventory 文件，或用 `cloud.terraform` 动态 inventory 插件直读 tfstate，让 Ansible 自动知道"机器在哪、什么标签"；② user_data 只做三件事——装 python、注入运维 SSH 公钥、注册 hostname，其余全部交给 Ansible；③ K8s 集群的"建"是 Ansible 的活——编排 kubeadm init/join、装 containerd、配 CNI，因为这是"机器内状态"问题，但集群就绪后一切工作负载管理回到 kubectl/Helm，Ansible 只在少数运维场景（批量维护节点）介入。整体编排落在 CI/CD 流水线（或 AWX/Terraform Cloud 这类编排层）：代码提交 → CI 做每层工具的校验门禁（terraform fmt/validate、ansible-lint + --syntax-check + --check --diff、helm lint + template）→ terraform plan 人工确认 → ansible-playbook 按 serial 灰度执行 → helm upgrade 带 --atomic --timeout → 自动化验证（健康检查/流量拨测）。回滚按层独立又讲究顺序：应用层 helm rollback --revision 最快见效先做；配置层 Ansible 幂等重跑旧版本变量；基础设施层 terraform 回滚到上个 state 是最后手段（重建代价最大）。这套体系的价值判断标准很简单：每个"该谁干"的问题都有明确答案——工程师顺着流水线能看清一个资源的完整生命周期，从 Terraform apply 到 Ansible 配置到 Helm 部署再到故障回滚，没有模糊地带。

**延伸考点：** kubeadm join 为什么是非幂等操作，Ansible 编排时如何把它做成幂等（先查 node 是否已注册）？引入 GitOps（ArgoCD）后，Ansible 在容器化应用层的角色如何进一步退场？

---
