# DevOps · Linux 运维实操（面试题库）

本文件面向 DevOps、SRE 与后端工程师，考察候选人在 Linux 系统运维实操上的真实能力：系统排障四件套（CPU/内存/IO/网络）、进程与 systemd、磁盘与文件系统、Shell 脚本与文本处理、日志与定时任务、性能分析、安全加固，以及从故障案例到高可用与自动化平台建设的工程思维。所有题目均以线上真实场景切入，不考八股文，重点看候选人能否给出量化依据、讲清方案取舍，并用具体命令和排障过程佐证自己的实践。题目难度从实践基础到架构级渐进，覆盖日常运维到平台化演进的全链路。

---

### Q1. 系统排障方法论：CPU/内存/IO/网络「四类资源」的排查顺序与工具矩阵

**问题：** 监控告警"某台业务机器响应变慢"，你到现场后第一步做什么？如何按 CPU、内存、IO、网络四类资源系统化排查，而不是凭感觉乱点命令？

**期望加分项：**
- 有明确方法论：先看现象定方向（`uptime`/`top` 看 load），再用 `vmstat 1` 观察 r（运行队列）与 b（阻塞队列）快速区分「CPU 忙」还是「IO 忙」，而不是一上来就刷 top
- 能给出完整工具矩阵：CPU 用 `top`/`pidstat`/`sar -u`，内存用 `free`/`vmstat`/`sar -r`，IO 用 `iostat`/`iotop`/`sar -d`，网络用 `ss`/`iftop`/`nload`，每个工具知道看哪一列
- 遵循「先全局后局部、先系统后进程」：uptime → vmstat → top 找进程 → pidstat/top -H 下钻到线程，逐层收窄
- 会用 `sar -q -u -r -d -n DEV -f /var/log/sa/saXX` 回溯历史数据，对齐故障发生时间点，而不是只盯当前瞬间
- 能识别反直觉场景：load 很高但 CPU 空闲（D 状态进程阻塞在 IO）、单核高 vs 多核均摊、上下文切换（`vmstat` 的 cs 列）飙升
- 有止损意识：先评估是否需要扩容/摘流量止血，再慢慢根因分析

**减分项：**
- 没有方法论，上来就 `top` 乱翻，无法按资源维度收敛排查范围
- 分不清 `vmstat` 各列含义（r/b、us/sy/wa/id、si/so、cs），说不出 CPU 和 IO 瓶颈的区别
- 只会在故障当下看快照，不会用 sar/atop 回溯历史，复现不了就两手一摊
- 答不出"iowait 高但磁盘不忙"这类经典误判场景
- 无真实排障经历支撑，全是背诵的工具列表

**解答：**
标准打法是把"四类资源"当独立子系统逐层排除。第一步先看整体：`uptime` 的 load 若远高于核数，说明系统过载，但过载原因可能是 CPU 忙、IO 阻塞（D 状态进程计入 load）或不可中断锁等待，所以第二屏立刻用 `vmstat 1 5` 看 r/b 列：r 持续大于核数 → CPU 过载；b 非零且有 wa → IO 瓶颈。接着按方向切工具：CPU 方向 `top -o %CPU` 找进程，`pidstat -t -p <pid> 1` 下钻到线程，`perf top` 看热点函数；IO 方向 `iostat -x 1` 看 %util/await，`iotop` 找吃 IO 的进程；内存方向 `free -h` 看 available，`vmstat` 看 si/so 是否在换页；网络方向 `ss -s` 看连接数，`iftop` 看流量，必要时 `tcpdump` 抓包。关键习惯是留证据：故障期间立刻 `sar -o` 采集或事后用 `/var/log/sa/saXX` 回溯，很多疑难杂症（半夜偶发、低峰期反涨）只靠现场快照根本定位不了。实践中的坑：一是单核机器 load 0.8 已经很危险，8 核机器 load 8 才算满载，必须按核数折算；二是 top 第一行 %Cpu 的 wa 高不代表磁盘坏，可能是 swap 换页（si/so 非零）或网络文件系统；三是排查期间不要反复重启服务，否则进程级证据全丢。方法论的价值在于把"玄学排障"变成可复现的四象限排查流程。

**延伸考点：** load average 与 CPU 使用率在什么场景下会背离（如 D 状态 IO 阻塞）？如何用一条 `sar` 命令同时拿到故障时间段 CPU、内存、IO、网络的历史指标？

---

### Q2. 常用命令组合：ps/top/htop/free/df 的常见用法，如何通过一条命令链定位「CPU 高的进程」

**问题：** 值班收到告警：某台 8 核机器 CPU 使用率 100% 持续 5 分钟。请给出一条命令链定位是哪个进程、哪个线程在吃 CPU，并说明每一步在看什么。

**期望加分项：**
- 能立刻写出批量化命令：`ps -eo pid,ppid,pcpu,pmem,comm --sort=-pcpu | head -n 10` 或 `top -b -n 1 -o %CPU | head -n 20`，知道非交互式输出用于脚本和取证
- 理解 %CPU 是单核百分比：8 核机器单进程 100% 表示吃满 1 核，800% 才吃满整机，能据此判断是单点热点还是负载均衡失效
- 能下钻到线程级：`top -H -p <pid>` 或 `pidstat -t -p <pid> 1`，再结合 `perf top`/`jstack`（Java）定位是业务代码还是 GC、锁等待
- 能讲清理或止损动作：确认进程归属后 `renice`/`kill`/摘流量/扩容，且先看 `ps -ef` 确认不是关键业务进程
- 会用 `pidstat -u -p <pid> 1` 观察进程 CPU 的 us/sy 占比，区分用户态计算密集还是系统调用/内核态异常

**减分项：**
- 只会交互式 `top` 按 P 键，写不出可脚本化的非交互命令，现场还慌着手工刷新
- 把 %CPU 当全局百分比，说不出 8 核下 100% 的意义
- 停在进程层不往下钻，答不出"怎么定位到具体线程和热点"
- 定位到进程后没有处置预案，或未经确认直接 kill 影响线上
- 不会用 `pidstat`/`perf` 这类进阶工具佐证判断

**解答：**
第一步用快照式命令建立证据链：`top -b -n 1 -o %CPU | head -n 20` 拿到当前 CPU 最高的进程，`-b` 批处理、`-n 1` 只采一次、`-o %CPU` 排序，适合快速取数；更规范的是 `ps -eo pid,pcpu,comm --sort=-pcpu | head -n 10`，输出稳定、方便 awk 二次处理。看结果时先算账：8 核机器 top 第一行 %Cpu(s) 约 800% 为满载，若单进程 %CPU 在 100% 上下，说明是单线程热点（可能是死循环、GC 频繁或锁自旋）；若多个进程各占几十，则更像流量上涨或资源争抢。第二步下钻线程：`top -H -p <pid>` 找出最热的 TID，若机器有 perf 权限，`perf top -p <pid>` 直接看内核态/用户态热点函数；Java 进程可 `jstack <pid>` 结合线程号换算（0x 转 16 进制）定位业务代码行；PHP/Node 场景则是 `strace -p` 观察系统调用卡在哪。第三步处置：先 `ps -ef | grep <pid>` 确认进程归属与启动参数，业务进程给上游限流或扩容，异常进程按变更流程 kill 并复盘。实践中的坑：top 的 %CPU 是瞬时采样且会跳变，判断稳定热点要多采几次或 `top -d 2`；`ps` 的 pcpu 是累计均值，两者口径不同；排查前先 `date` 记录时间点，与监控系统对齐数据。

**延伸考点：** `pidstat -t` 输出的线程号如何与 Java 线程 dump 对应起来？`top` 里 %CPU 超过 100% 出现在什么时候（多线程进程）？

---

### Q3. 进程管理与信号：进程状态（R/S/D/Z）、僵尸进程成因与清理、kill 信号（TERM/KILL/HUP）的语义

**问题：** 线上出现大量 `ps` 输出里标记 Z 的僵尸进程，同时有进程停在 D 状态。这两类分别是什么？怎么安全清理？`kill -9` 是不是万能的？

**期望加分项：**
- 能准确描述状态机：R 运行/可运行、S 可中断睡眠（等事件）、D 不可中断睡眠（通常等 IO）、T 停止、Z 僵尸（已退出但未被父进程回收）
- 讲清僵尸成因与生命周期：子进程 exit 后内核保留 PCB 等父进程 `wait()` 回收；父进程不 wait 就堆积；父进程退出后由 1 号进程（systemd）收养并回收
- 能给出清理手段：僵尸进程本身是"尸体"杀不掉（kill 无效），正确做法是处理其父进程（kill 父进程、重启父进程所在服务），少数由 systemd 收养的僵尸可通过重启系统清理
- 说清 D 状态的处理：通常与 IO 相关（磁盘、NFS），先 `iostat`/`cat /proc/<pid>/stack` 看阻塞点，盲目 kill 无济于事甚至加剧问题
- 讲清信号语义：TERM(15) 优雅退出可捕获做收尾、KILL(9) 强杀不可捕获、HUP(1) 挂断/重载配置（nginx -s reload、sshd 重读配置）、QUIT(3) 生成 dump；`kill -l` 可列出全部
- 有工程经验：批量 kill 前先 `pgrep -f` 确认匹配范围，避免误杀；Java 应用收到 TERM 后要等优雅停机超时再 KILL

**减分项：**
- 分不清僵尸与 D 状态，或说出"僵尸进程用 kill -9 杀掉"这种硬伤（僵尸已死，只能处理父进程）
- 只背 R/S/D/Z 四个字母，讲不清 D 状态和 IO 的关系
- 把 kill -9 当第一选择，不懂 TERM 优雅退出的价值（丢数据、状态不一致的坑）
- 说不出僵尸回收靠父进程 wait / init 收养的机制
- 无批量操作经验，`pkill`/`killall` 乱用误伤其他进程

**解答：**
先区分两类"异常"：Z 状态（僵尸）意味着子进程已经死透，只是父进程没有调用 wait() 收尸，内核保留其 PCB 供父进程读取退出码；父进程持续不回收就会堆积，积到一定量会耗尽 pid 资源导致无法 fork 新进程。清理手段不是 kill 僵尸（它已是尸体），而是处理父进程：`ps -eo ppid,stat,comm | awk '$2 ~ /Z/ {print $1}'` 找出僵尸的父进程，确认后 kill 父进程或重启其服务，父进程死亡后 systemd 会收养这些孤儿并自动回收。D 状态则完全不同：进程处于不可中断睡眠，通常是等磁盘/网络 IO 完成（NFS 挂死是经典场景），`cat /proc/<pid>/stack` 和 `iostat -x 1` 才能看到阻塞点，此时 kill 信号根本送不进去，只能等 IO 恢复或重启宿主机。信号语义是必考基础：TERM(15) 是"请你退出"，进程可捕获后优雅收尾（刷盘、断连接、写退出日志），是服务重启的标准方式；KILL(9) 是"必须死"，不可捕获、不可忽略，常用于 TERM 超时后的兜底；HUP(1) 最常用在重载配置（nginx -s reload、systemctl reload）。实践要点：给 Java/Go 服务的 kill 流程应"先 TERM，等优雅停机超时（如 30s）再 KILL"；批量操作前用 `pgrep -f '完整命令'` 精确匹配并先 `echo` 预览，杜绝 `killall java` 这种误杀全场的操作。

**延伸考点：** 为什么僵尸进程会占满 pid 空间导致系统无法启动新进程？`kill -HUP` 重载配置的适用条件是什么（哪些服务不支持）？

---

### Q4. systemd 与服务管理：unit 文件结构、systemctl 常用操作、服务自启动与故障重启策略（Restart=）

**问题：** 团队新上一个 Java 服务，要求：开机自启、进程崩溃后 5 秒内自动拉起、但 1 分钟内失败超过 3 次就停止尝试（防止重启风暴）。请写出完整的 systemd unit 并说明每个关键配置的作用。

**期望加分项：**
- 能写出规范的 unit 三段式：`[Unit]`（Description/After/Requires/StartLimitIntervalSec）、`[Service]`（ExecStart/User/WorkingDirectory/EnvironmentFile/ExecStartPre）、`[Install]`（WantedBy=multi-user.target）
- 能精确配置重启策略：`Restart=on-failure`（仅异常退出才重启，正常 exit 0 不重启）与 `Restart=always` 的区别；`RestartSec=5` 控制间隔；`StartLimitIntervalSec=60` + `StartLimitBurst=3` 防重启风暴
- 会做资源与环境管理：`LimitNOFILE=65535` 文件句柄、`LimitCORE=0` 禁 core dump、`EnvironmentFile=/etc/myapp.env` 隔离配置、`User=myapp` 非 root 运行
- 熟悉操作面：`systemctl daemon-reload`（改 unit 必做）、`enable --now`、`status`/`is-active`/`is-enabled`、`journalctl -u myapp -f` 看日志
- 能处理坑：Type=forking 的守护进程需要配 PIDFile 否则 systemd 认为没起来；Type=simple（默认）要求 ExecStart 前台运行；旧版 centos6 的 chkconfig/service 迁移注意差异
- 有排障经验：`systemctl status` 与 `journalctl -xe` 结合看失败原因，`systemd-analyze blame` 分析启动耗时

**减分项：**
- 只会 `systemctl start/stop/restart`，写不出 unit 文件
- 改完 unit 不 `daemon-reload`，报错后一头雾水
- 分不清 `Restart=on-failure` 与 `always` 的语义差异
- 不配 StartLimitBurst，服务在 crash-loop 时反复重启拖垮机器
- 用 root 直接跑服务、不设 LimitNOFILE，上线后连接数一高就报 too many open files

**解答：**
以 Java 服务为例给出可直接落地的 unit：`[Unit]` 段配 `Description=myapp service`、`After=network.target` 确保网络就绪后再启动；`[Service]` 段配 `User=myapp`、`WorkingDirectory=/opt/myapp`、`EnvironmentFile=/etc/myapp.env`（配置与代码分离）、`ExecStart=/usr/bin/java -Xmx2g -jar app.jar`、`Restart=on-failure`、`RestartSec=5`、`StartLimitIntervalSec=60`、`StartLimitBurst=3`、`LimitNOFILE=65535`；`[Install]` 段配 `WantedBy=multi-user.target` 实现开机自启。关键点解释：Restart 语义上 `on-failure` 只在非零退出码/被信号杀死时重启，`always` 连正常退出也重启（cron 类任务慎用）；RestartSec 防止疯狂重启；StartLimit* 是 systemd 的"重启断路器"，超限后进入 failed 状态不再自动拉起，需人工 `systemctl reset-failed` 才能再启动——这正好防止了"崩溃即重启、重启即崩溃"的抖动风暴。实践坑：Type 默认 simple 要求 ExecStart 以前台方式运行，Java 的 java 命令天然前台没问题，但若进程自己 daemonize 就必须配 `Type=forking` + `PIDFile`；每次修改 unit 后必须 `systemctl daemon-reload` 否则改的是内存里的旧配置；排查服务起不来时，`journalctl -u myapp -n 50 --no-pager` 比翻日志文件更直接，常见失败原因是环境变量缺失（cron/systemd 环境都极简）和 WorkingDirectory 不存在。

**延伸考点：** `Restart=on-failure` 下被 OOM-Killer 杀掉算不算"failure"？`systemctl disable` 与 `mask` 的区别是什么？

---

### Q5. 磁盘与文件系统：df/du/inode 耗尽、文件系统类型（ext4/xfs）、挂载与 fstab、磁盘扩容

**问题：** 服务突然写不进日志，`df -h` 显示 / 分区 100%，但 `du -sh /*` 加起来远小于 df 的数字，空间去哪了？另外 `df -h` 还有剩余但应用报"磁盘空间不足"，又是什么原因？

**期望加分项：**
- 能讲清 df 与 du 差异的两个经典原因：文件被删除但仍被进程占用（`lsof +L1` 定位，删了进程不释放 fd，重启进程才释放）；`du` 默认不跨越挂载点（用 `-x` 参数限定同一文件系统）
- 知道 inode 耗尽：`df -i` 查看，大量小文件（缓存目录、消息队列、容器 overlay）吃光 inode，磁盘有空间也写不进；`find / -xdev -type f | wc -l` 或 `for d in /*; do find $d -xdev -type f | wc -l; done` 定位
- 能讲文件系统差异：ext4 支持 shrink（resize2fs 可缩小）与在线扩容，xfs 只能 grow 不能 shrink；生产大分区多用 xfs；`xfs_growfs`/`resize2fs` 对应各自 fs 的扩容命令
- 熟悉 fstab 与挂载：`/etc/fstab` 字段（device/mountpoint/fstype/options/dump/pass），`mount -a` 校验、`nofail` 防挂载失败卡启动
- 有扩容实操：LVM 流程（pvcreate → vgcreate → lvextend → resize2fs/xfs_growfs）或直接扩展分区（growpart）；先 `df -h` 确认目标再操作，扩容后立即确认
- 有治理意识：日志落盘策略（logrotate）、`/tmp` 定期清理、配额（quota）或监控告警（磁盘使用率阈值 80%）

**减分项：**
- 不知道 `df -i`，答不出 inode 耗尽的场景
- 说不清"文件已删除但空间不释放"的机制，只会让用户删文件
- 不知道 xfs 与 ext4 扩容命令的差异（xfs 不支持缩小）
- 改 fstab 不先 `mount -a` 校验，重启后起不来
- 无排查步骤，一上来就"重启大法"

**解答：**
分两个经典场景答。场景一：df 100% 但 du 对不上。首选 `lsof +L1 | head` 找出"已删除但仍被进程持有 fd 的文件"，这些文件在文件系统上已无目录项、du 看不到，但占用的块没释放，处理方式是重启持有进程（Java 的日志文件、MySQL binlog、删除中的大文件是高频来源）；其次检查 du 是否忽略了别的挂载点：`du -xsh /*` 限制在根文件系统内统计，避免把其他分区算进来。场景二：df 有剩余但写不进去。查 `df -i`，inode 用满时 ext4/xfs 会直接返回 No space left；定位方法是用 `find` 按目录统计小文件数量，典型元凶是定时任务产生的海量临时文件、邮件队列、容器 overlay2 目录。文件系统选型：xfs 单文件上限大、适合大文件和大分区，但不能 shrink；ext4 可以缩小分区，小规模和老系统更通用；生产上加 LVM 的目的就是让"扩容"变成 `lvextend -L +10G /dev/vg/data && resize2fs`（ext4）或 `xfs_growfs /data`（xfs）两步。fstab 的坑：新增挂载项一定要先 `mount -a` 验证无错再写进 fstab，并给关键挂载加 `nofail` 选项，否则磁盘插拔/顺序变化会导致系统启动时卡在挂载失败进不了系统。最后是治理：磁盘使用率 >80% 告警、日志轮转、/tmp 清理都要提前做，而不是等写满再救火。

**延伸考点：** `df -hT` 里挂载点显示为 overlay/overlay2 是什么场景（容器）？ext4 在线扩容后为什么要 resize2fs，顺序反了会怎样？

---

### Q6. 网络排查：ss/netstat 查看连接、tcpdump 抓包分析、nc/telnet 端口连通性、常见网络问题定位

**问题：** 客户端反馈"连不上 10.0.0.5 的 8080 端口"，你在服务端机器上如何逐层排查？请给出从连通性、端口监听、到数据包层面的完整排查链路。

**期望加分项：**
- 有分层排查思路：先基础连通性（`ping`/`ip addr`），再端口连通（`nc -vz 10.0.0.5 8080`、`telnet`、`bash -c 'echo > /dev/tcp/10.0.0.5/8080'`），最后应用层（curl 带 -v 看响应头）
- 能区分"服务没起、端口被防火墙挡、监听地址不对"三类原因：`ss -lntp` 看监听地址是 0.0.0.0 还是 127.0.0.1；`ss -ant` 看连接状态（ESTABLISHED/SYN_SENT/TIME_WAIT）判断是主动连接失败还是被动断开
- 会用 tcpdump 抓包定位：`tcpdump -i eth0 tcp port 8080 -nn -c 100 -w /tmp/cap.pcap`，看三次握手是否完成、是否有 RST（谁拒绝谁发送 RST）、SYN 是否重传
- 能讲清 TCP 状态机在排障中的应用：SYN_SENT 多→对端或中间防火墙丢包（DROP），收到 RST→对端明确拒绝（服务未监听或 iptables REJECT）
- 会区分 firewalld 与 iptables：`firewall-cmd --list-all`/`iptables -L -n` 查规则，DROP 与 REJECT 对客户端表现不同（超时 vs 拒绝），selinux 也是常见元凶
- 有进阶排查：`ss -s` 看连接统计、`ethtool -S eth0`/`ifconfig eth0` 看丢包错包、`mtr`/`traceroute` 定位链路丢包、MTU 问题（大包通小包不通）

**减分项：**
- 只会 `telnet` 一下就说"连不上"，分不出是防火墙、监听地址还是应用问题
- 分不清 netstat/ss 里 SYN_SENT、TIME_WAIT 的含义，不会用状态分布判断问题方向
- 不会抓包，或抓了不会看（三次握手、RST 的方向）
- 忽略 selinux 和监听在 127.0.0.1 这类低级但高频的坑
- 防火墙排查只背命令，说不清 DROP 与 REJECT 的区别

**解答：**
标准做法是从下往上分层验证。第一层链路与网络：`ping 10.0.0.5` 通不通（不通查路由、安全组/防火墙、对端是否关机）；第二层端口连通性：`nc -vz 10.0.0.5 8080` 返回 succeeded 说明 TCP 能连上，剩下就是应用层问题；超时大概率被 DROP，立即拒绝大概率 REJECT 或服务未监听。第三层看服务端：`ss -lntp | grep 8080` 确认有没有进程监听、监听在 0.0.0.0 还是 127.0.0.1（后者只有本机能连，这是最高频的"服务起了但外网连不上"原因）；再看 `ss -ant | grep 8080` 的连接状态分布：SYN_SENT 一堆→请求包发出没响应，大概率中间设备丢包；TIME_WAIT 大量→连接释放正常现象，一般无碍。第四层抓包定性：`tcpdump -i eth0 tcp port 8080 -nn` 盯 SYN 包：只有 SYN 没有 SYN-ACK 回复→包被中间防火墙/安全组丢了；收到 RST→对端主动拒绝，结合 RST 从哪端发出判断是谁在拒（服务端没监听→内核回 RST；防火墙 REJECT→防火墙回 RST）。最后别忘了两个隐藏元凶：selinux（`getenforce` 为 Enforcing 且审计日志被拒）和监听 backlog 满（`ss -lnt` 看 Send-Q 列就是 backlog 长度，满了会有 SYN 重传但服务无感知）。实践中的坑：先确认自己排查用的网络路径和客户端一致（别本机测通了、客户端走 NAT 不一样）；抓包记得 `-nn` 关解析避免抓包工具自身流量干扰，`-w` 落盘给开发一起看。

**延伸考点：** 连接卡在 SYN_RECV 堆积是什么问题（半连接队列满）？tcpdump 抓包看到大量重传（TCP Retransmission）说明链路还是对端的问题？

---

### Q7. Shell 脚本编写规范：set -euxo pipefail、变量引用引号、避免常见陷阱（空格/通配符/IFS）

**问题：** 团队一个部署脚本偶尔出诡异问题：有时"看着执行成功了其实某一步失败了"、遇到含空格的文件名就报错、变量为空时把整个目录删了。请写出一份生产级 Shell 脚本的编写规范。

**期望加分项：**
- 能准确解释 `set -euxo pipefail` 各开关：`-e` 命令失败即退出、`-u` 未定义变量即报错、`-x` 打印执行过程（调试用）、`pipefail` 让管道返回最后一个非零状态（没有它 `cmd | grep x` 只要 grep 成功就掩盖前面失败）；并知道 `-e` 在条件判断中的坑（`if cmd; then` 里的 cmd 失败不会退出）
- 变量引用规范：所有变量加双引号 `"$var"`、数组展开用 `"${arr[@]}"`，防止空格分词和通配符展开
- 能讲常见陷阱：`for f in $files` 遇空格/通配符炸、`[ "$var" = "x" ]` 空变量报错、命令替换 `$(cmd)` 的结果也要引号、IFS 自定义后未还原
- 会用 `[[ ]]` 代替 `[ ]`（支持正则、避免引用地狱）、`trap 'rm -f "$tmpfile"' EXIT` 做清理、函数内 `local` 变量、明确退出码
- 有工程化习惯：脚本开头 `set -Eeuo pipefail`、`IFS=$'\n\t'`、关键步骤写日志带时间戳、`shellcheck` 静态检查、绝对路径、`cd "$(dirname "$0")"` 定位脚本目录
- 安全红线：不 `eval`、`rm -rf` 前校验变量非空、不用无引号的通配符做删除

**减分项：**
- 不知道 `pipefail`，管道把前面命令的失败吞掉（"管道返回最后一条命令的退出码"）
- 变量不加引号，说不清"文件名带空格为什么炸"
- 用 `[ $var ]` 而不是 `[[ $var ]]` 或 `[ "$var" ]`，空变量/含空格变量直接语法报错
- 无 trap 清理，脚本中断后残留临时文件/锁
- 没跑过 shellcheck，写的脚本上线全靠运气

**解答：**
一份可靠脚本的骨架：`#!/usr/bin/env bash` + `set -Eeuo pipefail` + `IFS=$'\n\t'` + 必要处 `trap cleanup EXIT`。逐个解释：`-e` 让脚本在首个失败处停下（避免"失败还继续往下跑"的连锁事故）；`pipefail` 把管道里所有命令纳入失败判定——没有它 `mysqldump ... | gzip > a.sql.gz` 里 mysqldump 失败但 gzip 成功，退出码是 0，备份其实是个空文件，这类 bug 线上坑过无数人；`-u` 让 `$UNDEFINED` 立即报错而不是静默变空串（空串参与 `rm -rf $dir/` 就是删错目录的根源）；`-x` 平时不开、调试时开。引号是第二生命线：`"$var"` 保证含空格的文件名被当整体，`"${arr[@]}"` 保证数组逐元素传递，`for f in *.log` 这种直接通配符展开是安全的，但 `for f in $list`（把文件名列表放变量里）必炸——要么 `for f in "$@"` 传参，要么用数组。条件判断用 `[[ ]]`：支持 `=~` 正则、空变量安全、内部不做分词。清理用 trap：`trap 'rm -f "$LOCK" "$TMP"' EXIT INT TERM` 保证任何退出路径都收尾，锁文件配合 `flock` 防重入。实践建议：写完 `shellcheck script.sh` 扫一遍（能抓出未加引号、误用 eval 等 90% 的坑）；日志统一 `log() { echo "$(date '+%F %T') [$$] $*" | tee -a "$LOGFILE"; }`；删除操作先 `echo` 预览再执行，目录变量用 `: "${DIR:?DIR not set}"` 强制必填。这些都是"看起来小、线上出事就是大事"的细节。

**延伸考点：** `set -e` 下 `cmd || true` 和 `if cmd; then` 有什么语义区别？为什么脚本里 `sudo` 前要谨慎处理 `$HOME`（sudo 默认重置环境变量）？

---

### Q8. 文本处理：awk/sed/grep 组合实战（提取日志字段、批量替换、过滤统计）

**问题：** 运维要给一份 5 万行的 Nginx access.log 做分析：统计访问量 Top10 的 IP、统计 5 分钟内 5xx 错误数、把配置里某个 IP 批量替换成新 IP。请给出具体命令并解释每一步。

**期望加分项：**
- 能立即写出统计组合拳：`awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -n 10`，并解释 `awk '{print $1}'` 提取 IP 字段（默认按空白切分）→ sort 排序 → `uniq -c` 计数 → `sort -rn` 按次数倒序 → head 取 Top
- 会用 grep 做条件过滤：`grep -E 'HTTP/1\.[01]" 5[0-9]{2}' access.log` 或 `grep -c ' 50[0-9] '`，结合 awk 的时间戳字段做时间窗口过滤（`awk '$4 ~ /^\[10\/Aug\/2026:10:3[0-9]/'`）
- 能用 awk 做字段级统计：`awk -F'"' '{print $6}'` 取状态码列、`awk '{code[$9]++} END {for (c in code) print c, code[c]}'` 做状态码分布，知道 -F 指定分隔符
- 会用 sed 做替换：`sed -i.bak 's/1\.1\.1\.1/2.2.2.2/g' nginx.conf`，替换前先 `sed 's/.../.../g' file | grep 校验` 或 sed 不加 -i 预览，知道 `sed -i` 的备份参数 `.bak`
- 了解 awk 变量与运算：`awk '{sum += $10} END {print sum}'` 统计流量、`awk 'NR>=100 && NR<=200'` 取行区间、`awk -v OFS=',' '{print $1, $2}'` 输出 csv
- 能组合 grep/awk/sed 讲清"提取→过滤→统计→输出"的管道思维，并注意大文件性能（少用 cat 开头、单遍 awk 优于多次 grep）

**减分项：**
- 只会 `grep 关键字`，不会管道组合做统计
- awk 语法生疏：分不清 `$0`/`$1`/`$NF`，不会 `-F`，`uniq` 前不知道要先 `sort`
- sed 直接 `-i` 改文件不校验不备份，改错了没得回滚
- 分不清 uniq 的 `-c` 和 sort 的 `-rn` 各自作用，或忘了 uniq 只能统计相邻行
- 无性能意识，日志量一大就 `cat file | grep | grep` 反复全量扫

**解答：**
逐场景给命令。场景一 Top IP：`awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -n 10`——注意 uniq 只能统计相邻的相同行，所以必须先 sort；`uniq -c` 输出"次数 IP"，再 `sort -rn` 按次数降序。场景二 5xx 统计：先看日志格式确认状态码列，默认 combined 格式里 `$9` 是状态码：`awk '$9 ~ /^5/ {cnt++} END {print cnt}' access.log`；要限定时间窗口再加条件：`awk '$4 ~ /^\[10\/Aug\/2026:10:(0[0-9]|1[0-5])/ && $9 ~ /^5/ {cnt++} END {print cnt}' access.log`，注意日志时间是 `[10/Aug/2026:10:00:01 +0800]` 这种格式，`$4` 就是带时间戳的字段。场景三批量替换：先预览 `sed 's/1\.1\.1\.1/2.2.2.2/g' nginx.conf | grep -n '2.2.2.2' | head`，确认无误再 `sed -i.bak 's/1\.1\.1\.1/2.2.2.2/g' nginx.conf`，`-i.bak` 生成备份文件，正则里 `.` 必须转义成 `\.` 否则会匹配任意字符。进阶玩法：状态码分布 `awk '{c[$9]++} END {for (k in c) print k, c[k]}' access.log | sort -n`；统计总响应字节 `awk '{s+=$10} END {printf "%.2f MB\n", s/1024/1024}'`；取每个用户的最近一条 `tac file | awk '!seen[$1]++'`。实践提醒：日志量大时避免 `cat | grep` 的多次全量扫描，一条 awk 搞定过滤+统计；字段位置依赖日志格式（combined/自定义），先 `head -1` 确认列号再写 awk，这是新人翻车率最高的地方。

**延伸考点：** `sort | uniq -c` 与 `awk '{c[$1]++} END {...}'` 两种统计方式在 10GB 级日志上的性能差异？sed 的正则和 grep -E 的正则语法差异（BRE/ERE）？

---

### Q9. 定时任务：crontab 语法、环境变量问题、日志输出、分布式场景下定时任务幂等与锁

**问题：** 运维写的 crontab 任务"在终端手动执行正常，但定时跑就不生效或报找不到命令"；另外多个应用节点同时执行同一个统计任务导致数据重复。分别怎么解决？

**期望加分项：**
- 能答出 cron 环境差异的核心：cron 默认 PATH 只有 `/usr/bin:/bin` 等极简路径，`$HOME`、`$JAVA_HOME` 等自定义环境变量全没有，脚本内必须用绝对路径或开头 source 环境文件
- 熟悉 crontab 语法与位置：分 时 日 月 周，`crontab -e`/`-l`，系统级 `/etc/crontab` 与 `/etc/cron.d/`（后者可指定执行用户），`%` 在 crontab 里要转义 `\%`
- 有日志与告警习惯：`>> /var/log/cron_task.log 2>&1` 重定向输出，否则 cron 默认把输出发邮件（MAILTO）且不留痕；脚本内自己打日志带时间戳
- 能讲幂等与锁：单机用 `flock -n /tmp/task.lock -c 'sh task.sh'` 或 mkdir 原子锁防重入（任务上次没跑完这次又启动）；跨节点用分布式锁（Redis SETNX/Redisson、数据库锁表、ZooKeeper），或按节点划分数据分片
- 有场景意识：cron 最小粒度 1 分钟，秒级/高频任务应改用 systemd timer（支持更细粒度、失败重试、依赖控制）或常驻进程
- 会排查：`grep CRON /var/log/cron` 或 `journalctl -u cron` 看任务是否触发，`date` 与服务器时区对齐（CST/UTC 差 8 小时导致"凌晨 2 点跑"变成"上午 10 点跑"）

**减分项：**
- 只会写 crontab 不会排查，出问题后不知道看 /var/log/cron 和 MAILTO
- 不知道 cron 环境与交互式 shell 不同，脚本里裸用相对路径/自定义环境变量
- 定时任务无日志无告警，跑挂了三天没人知道
- 分布式场景不做幂等/锁，任务重复执行产生脏数据
- 不知道 `%` 转义、时区这类边角但致命的坑

**解答：**
先解决"手动正常定时不生效"：cron 执行环境的 PATH 极简（通常是 /usr/bin:/bin），交互式 shell 里生效的 `$JAVA_HOME`、`/usr/local/bin` 里的命令在 cron 下全部不可见。规范做法：脚本开头 `export PATH=/usr/local/bin:/usr/bin:/bin` 或用绝对路径调用命令，脚本内部需要的变量用 `source /etc/profile`（注意 cron 下非登录 shell 不读 profile）或显式 export。加日志：`0 2 * * * /usr/bin/bash /opt/scripts/backup.sh >> /var/log/backup.log 2>&1`，这样 stdout/stderr 都落盘，配合 MAILTO 留告警通道。再看语法细节：周与月的执行条件是"或"而非"且"（`0 2 * * 1` 和 `0 2 1 * *` 一起写会每月 1 号 AND 每周一都跑）；`%` 在 crontab 里要写成 `\%`（常出现在 `date +\%F` 里）。重入与幂等是生产环境的必修课：单机加锁 `flock -n /tmp/etl.lock -c '/opt/scripts/etl.sh'`，锁被占用直接放弃本次，防止上一个任务因数据量大没跑完、下一个又启动造成资源竞争和重复数据；跨多节点（如 3 台机器同时跑统计）则用 Redis `SET key value NX EX 3600` 抢锁、数据库唯一索引/锁行、或任务本身支持幂等（重跑不产生脏数据、加业务唯一键）。场景权衡：cron 粒度最小 1 分钟且不感知依赖，需要秒级调度、失败自动重试、跨节点编排的应换 systemd timer 或调度平台（如 airflow、xxl-job）。最后排障三板斧：`grep backup /var/log/cron`（cron 是否执行）、`cat /var/log/backup.log`（脚本是否成功）、`date` 对齐服务器时区。

**延伸考点：** systemd timer 相比 crontab 的优势（OnCalendar/OnFailure/精度）？分布式锁里 SETNX 与 RedLock 的取舍？

---

### Q10. 日志管理与 logrotate：日志轮转配置（按大小/时间）、压缩、保留策略、日志切割不生效排查

**问题：** 应用日志每天涨 2GB，几天就把磁盘写满。请用 logrotate 配置一套方案，并说明"配了 logrotate 但日志就是不轮转"的排查思路。

**期望加分项：**
- 能写出规范配置：在 `/etc/logrotate.d/myapp` 下写 `daily`/`size 100M`（按时间 vs 按大小，日志不规则增长用 size 更稳）、`rotate 14` 保留份数、`compress` 压缩、`missingok`、`notifempty`、`copytruncate` 或 `create` + `postrotate 信号`
- 讲清 copytruncate 与 create 的取舍：应用进程一直持有日志 fd 不感知轮转时用 `copytruncate`（先 copy 再 truncate，可能丢少量写）；应用支持信号重开文件（nginx/rsyslog）用 `create` + `postrotate kill -USR1/HUP` 更干净
- 知道触发机制：logrotate 本身是 cron 任务（/etc/cron.daily/logrotate），按天执行；daily 配置真正生效要 cron 按时跑，size 配置在 cron 执行时检查
- 会排查不生效：`logrotate -d /etc/logrotate.conf`（debug 模式逐条打印决定）、`logrotate -f /etc/logrotate.d/myapp` 强制执行、查看 `/var/lib/logrotate.status` 记录的时间戳（daily 配置下当天已轮转过就不会再转）
- 有工程细节：`dateext` 用日期命名、`maxsize` 组合策略、压缩用 gzip 与进程内压缩的取舍、轮转后 `find` 清理超期备份、日志目录权限（应用属主才能写）
- 能延伸到容器/平台场景：容器日志建议输出到 stdout 由 docker 引擎轮转（json-file 的 max-size/max-file），避免容器内多套 logrotate

**减分项：**
- 只背 `daily + rotate` 几个词，说不清触发机制和 status 文件
- 不知道 copytruncate 与 create 的区别，轮转后应用继续写旧文件导致磁盘照样满
- 配置完不验证（不知道 `logrotate -d` 和 `-f`）
- 应用不支持信号重开日志，配 create 模式反而把日志写丢
- 不做保留策略，rotate 数随意，磁盘没救了才想起

**解答：**
以"应用固定写 /var/log/myapp/app.log"为例给标准配置：`/var/log/myapp/*.log { daily; rotate 14; compress; delaycompress; missingok; notifempty; dateext; copytruncate; }`。选型逻辑：日志增长与流量强相关、时高时低，用 `size 100M`（每次 cron 检查时超过 100M 就转）比 `daily`（固定每天转一次，流量大的日子一天能写 2GB 不转）更能控磁盘；要两者结合可用 `maxsize 100M`（到时间或到大小任一满足即转）。`copytruncate` 适用于应用不感知文件轮转（持有旧 fd 继续写）：先 `cp` 原文件为轮转文件、再把原文件截断为 0，代价是 copy 与 truncate 之间可能丢几行日志；如果应用支持 HUP/USR1 重开日志（nginx、tomcat 的 jcatalina 处理、rsyslog），用 `create 0644 myapp myapp` + `postrotate kill -USR1 \`cat /var/run/myapp.pid\`` 最干净，轮转后应用重新打开新文件。触发与排查：logrotate 由 `/etc/cron.daily/logrotate` 驱动，`/var/lib/logrotate.status` 记录每个文件上次轮转时间——这就是"配了 daily 但今天没转"的常见原因（当天已转过，或 cron 没跑）；排查用 `logrotate -d /etc/logrotate.d/myapp` 看 verbose 决策、`-f` 强制执行并观察文件是否轮转、`ls -l` 和 `cat /var/lib/logrotate.status` 验证结果。实践坑：`compress` 会延迟到下次轮转才压缩（delaycompress 时更明显），大日志 `gzip` 可能耗时导致磁盘瞬时双倍占用；轮转文件落在磁盘不足时 `logrotate` 会跳过并告警；日志目录属主不对（root 建的文件应用写不进）是新坑；容器场景建议直接把日志打 stdout，由 Docker 的 `max-size`/`max-file` 或平台采集侧处理，别再容器里叠 logrotate。

**延伸考点：** `delaycompress` 的作用是什么，为什么配了它压缩会推迟？copytruncate 模式丢日志的量级怎么评估（两命令间隙的写入）？

---

### Q11. 性能分析工具：perf/top/atop/pstree 定位热点、平均负载与 CPU 使用率的区别（load average 含义）

**问题：** 一台 4 核机器 `uptime` 显示 load average 是 5.0，但 `top` 里 CPU 使用率却只有 20% 且 CPU idle 很高。这个负载是哪来的？怎么定位？

**期望加分项：**
- 能讲清 load average 的构成：运行队列中的进程数 + 不可中断睡眠（D 状态）进程数，1/5/15 分钟三个窗口；它不是"CPU 使用率"，D 状态进程（等 IO、锁、NFS）会拉高 load 但不占 CPU
- 能解释本场景：4 核 load 5.0 说明超出饱和（每核 1.25），但 CPU 空闲 → 大量进程阻塞在 IO/锁等待，用 `vmstat 1` 看 b 列（阻塞进程数）和 wa 列验证，或 `ps -eo pid,stat,comm | awk '$2 ~ /D/'` 数 D 状态进程
- 会用工具链深挖：`vmstat` r/b 列定性 → `iostat -x 1` 看磁盘 await/%util → `iotop` 定位吃 IO 的进程 → `cat /proc/<pid>/stack` 看阻塞内核栈（NFS、ext4 journal、块层排队）；`atop` 可回放历史（-r 参数读历史文件）
- 会用 perf 定位 CPU 热点：`perf top` 实时、`perf record -g -p <pid>` + `perf report` 看调用栈火焰图数据，区分用户态业务逻辑 vs 内核态（锁、中断、调度）
- 用 pstree/ps 理顺进程树：`pstree -ap <pid>` 看父子关系，识别是否单个应用 fork 大量线程/进程拖垮系统
- 有量化与趋势意识：load 的 15 分钟均值用于看趋势（是否持续恶化），1 分钟看当下；持续高 load 与偶发 spike 处置策略不同

**减分项：**
- 把 load average 直接等同于 CPU 使用率，看到"load 5 CPU 空闲"就懵
- 不知道 load 包含 D 状态进程，说不清 IO 阻塞会拉高负载
- 不会用 vmstat 的 r/b 列区分"运行队列满"与"阻塞队列满"
- 只停在"看 load 高"不深挖，不会用 perf/iostat/iotop 往下定位
- 无历史数据意识，不看 1/5/15 分钟趋势就下结论

**解答：**
先纠正概念：load average 是"运行队列长度 + 不可中断睡眠进程数"的滑动平均，1/5/15 分钟三个窗口。它反映的是"有多少进程在等待 CPU 或等待 IO"，而不是"CPU 忙不忙"——这正是本场景的反直觉点：4 核机器 load 5.0 意味着平均有 1 个进程在排队等待资源，但 CPU 空闲 80%，说明排队等的大多是 D 状态（等 IO 完成）而不是 R 状态（等 CPU）。定位步骤：第一步 `vmstat 1 5` 看 b 列（阻塞在 IO 的进程数），若 b 持续大于 0 且 wa（CPU 等 IO 占比）高，方向锁定 IO；第二步 `iostat -x 1` 看 await（单次 IO 平均耗时）和 %util，SSD 的 await 应在几毫秒内，若到几十毫秒说明磁盘饱和或 RAID 问题，`%util` 到 100% 且 avgqu-sz 大说明排队严重；第三步 `iotop -o` 找出具体吃 IO 的进程（常见：日志风暴、数据库全表扫描、crontab 里的大数据拷贝）；若 b 列高但磁盘指标正常，则查锁与内核阻塞：`cat /proc/<pid>/wchan` 和 `/proc/<pid>/stack` 看阻塞点（NFS 挂起、ext4 日志、socket 缓冲区满都是典型）。CPU 热点用 perf 系列：`perf top` 实时看热点符号，`perf record -g -p <pid> -- sleep 30` 后 `perf report --stdio` 看带调用栈的热点，能直接区分是业务死循环还是内核锁；`pstree -ap` 帮你看清"一个进程 fork 出几百个子进程"这类失控场景。atop 的价值在事后：`atop -r /var/log/atop/atop_20260810` 回放故障时间段 CPU/IO/内存全量数据，很多"现场已恢复"的故障靠它复盘。实践坑：load 与 CPU 使用率的背离还有另一种——大量线程在用户态锁上自旋（spinlock）时 CPU 会高而 load 未必高，反之 NFS 挂死时 load 飙升 CPU 空闲，两种都要会判。

**延伸考点：** 1 分钟 load 高、15 分钟 load 低（或反过来）分别说明什么趋势？`/proc/loadavg` 第一列与 `uptime` 的关系？

---

### Q12. 内存管理：free 各字段含义（buff/cache）、内存回收、OOM 机制与 dmesg 查看、swap 配置

**问题：** 监控显示内存 used 95%、free 只剩几百 MB，报警"内存不足"，但业务没有明显异常。同事说"Linux 内存不能这么看"。你怎么解释，并给出正确的判断与处置方法？

**期望加分项：**
- 能逐字段解释 `free -h`：total/used/free/shared/buff/cache/available；重点：buff/cache 是内核利用空闲内存做的缓存（块缓冲+页缓存），可被回收，**判断内存是否不足看 available 而非 free**
- 能讲内存回收机制：page cache 先被回收（写回脏页后释放）、内核 kswapd 在压力下回收、`/proc/sys/vm/swappiness` 控制换页倾向（默认 60，倾向优先回收 cache 还是换 swap）
- 知道 OOM 机制与排查：当内存真不足，内核 OOM-Killer 按 oom_score（含 oom_score_adj）选进程杀；`dmesg -T | grep -i -E 'oom|killed process'` 或 `journalctl -k` 查被杀进程与内存快照，`/var/log/messages` 也有记录
- 能讲 swap：`swapon -s` 看现状、`mkswap`+`swapon` 创建 swapfile/swap 分区、swap 不是越多越好（swap 耗尽伴随系统卡死）；容器场景注意 cgroup 内存限制下 swap 行为
- 有量化与工程经验：判断阈值看 available 占比（如低于 20% 预警）、区分"进程 RSS 总和超物理内存"（真不足）与"cache 占用大"（假象）、用 `smem`/`ps -o rss` 统计进程真实内存
- 有止损预案：OOM 高频时调整 oom_score_adj 保护关键进程、限制单进程（ulimit -v / cgroup）、或直接扩容/优化内存占用

**减分项：**
- 只看 used/free 两个数字就报"内存不足"，不知道 buff/cache 可回收、不知道看 available
- 不知道 OOM-Killer 的存在与机制，或只会"重启机器"处置
- 查不到 OOM 日志，不知道 dmesg/journalctl 的命令
- 盲目加大 swap 或用 swappiness=0 一刀切，讲不清取舍
- 说不清 page cache 与进程匿名内存的区别

**解答：**
先纠正看数口径：`free -h` 里 used 是"已被进程和内核用掉的"，但其中的 buff/cache 是内核把空闲内存拿去做缓存（文件读写加速），**内存紧张时内核会自动回收 cache，不会直接 OOM**，所以真正要盯的是 available（对无 swap/无压力进程可用的估算值），"free 只剩几百 MB"在 Linux 上几乎永远是常态而非异常。判断内存是否真不足：看 available 占比（如低于总内存 20% 且持续下降）、看 `vmstat` 的 si/so（持续非零说明在换页，这才是真压力）、或对比进程 RSS 总和 `ps aux --sort=-rss | head` 是否接近物理内存。若真 OOM，内核会启动 OOM-Killer 按 oom_score 杀进程（得分基于进程内存占用、运行时长等，oom_score_adj 可人工加减权重），此时 `dmesg -T | grep -i -E 'oom-kill|out of memory'` 能看到被杀 PID、进程名和当时的全局内存快照——这是事后复盘的第一手证据，重点回答三个问题：谁占的内存（看快照里各进程 RSS）、为什么触发（谁在短时间内存暴涨）、有没有被误杀（关键业务要提前 oom_score_adj=-1000 保护）。swap 的配置逻辑：`mkswap` 创建 swapfile 后 `swapon` 启用并写进 fstab 持久化；swap 的意义是"缓冲"而非"性能"，进程冷数据换出后能腾出内存给活跃数据，但 swap 耗尽时系统会整体卡死（换页风暴），所以 swap 大小一般按物理内存比例（如 1-2 倍或按业务压测），SSD 时代可以小一些；`vm.swappiness` 调低（如 10）表示"尽量少用 swap、优先回收 cache"，适合内存较充裕的 Java 服务，调高则相反。实践坑：Java 的 -Xmx 只约束堆，JVM 的堆外（Metaspace/线程栈/direct buffer）常把物理内存吃爆导致 OOM-Killer 杀的是别人；容器里 cgroup 内存限制触发 OOM 与宿主机 OOM 是两套逻辑，排查要看容器侧事件。

**延伸考点：** OOM-Killer 的 oom_score 是怎么算出来的（rss、swap、oom_score_adj 的权重）？drop_caches（echo 3 > /proc/sys/vm/drop_caches）在生产上为什么慎用？

---

### Q13. 文件权限与 ACL：chmod/chown/setuid/setgid/粘滞位、ACL 场景、sudo 配置

**问题：** 两个业务团队要共享一个项目目录：A 团队的文件 B 团队要能读、某些子目录要双方都能写，但既不想全员 777，也不想每次都找管理员。你怎么用 Linux 权限体系解决？

**期望加分项：**
- 能讲清基础三件套：`chmod`（rwx 与数字/符号法）、`chown`/`chgrp`，以及 umask 对新建文件默认权限的影响
- 会讲 setgid 与粘滞位：目录设 setgid（`chmod g+s`）后新建文件继承目录属组，最适合团队共享目录；`/tmp` 的粘滞位（1777）表示"只有属主和 root 能删自己的文件"
- 能用 ACL 解决"多用户/多组不同权限"：`setfacl -m u:alice:rwx,g:devteam:rw dir`、`getfacl dir` 查看、`-R` 递归、default ACL（`setfacl -m d:u:bob:rwx`）让新建文件自动继承
- 知道 setuid 与风险：`chmod u+s` 让程序以属主身份运行（如 /usr/bin/passwd），生产环境要定期 `find / -perm -4000 -type f 2>/dev/null` 审计 setuid 文件
- 能配 sudo 最小权限：`visudo` 编辑，`user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx` 白名单授权，用 wheel 组管理 sudo 用户，禁 root 直登
- 有安全与工程意识：不用 777 解决协作（ACL 或组权限是正解）、权限变更先 `ls -ld` 确认现状、ACL 与 setfacl/getfacl 属于 acl 包、NFS/容器挂载对 ACL 和 setuid 的支持差异

**减分项：**
- 只会 777 一把梭，说不清最小权限
- 不知道 setgid 目录和粘滞位的作用
- 完全不知道 ACL（setfacl/getfacl），多用户权限只能靠 chmod
- setuid 只背"提权"概念，说不出审计与风险
- sudo 配成 `ALL=(ALL) ALL` 或 NOPASSWD 全部命令，等于没做权限管控

**解答：**
解题分三层：基础权限、组协作、ACL 兜底。第一层基础：目录 `/data/share` 属主设 root、属组设 `devteam`（两团队共同组），`chmod 2770 /data/share`——2 是 setgid 位，含义是"在此目录下新建的文件/子目录自动继承属组 devteam"，这样 A 团队创建的文件 B 团队也能按组权限访问，这是团队共享目录的标准解法，不需要 777。第二层粘滞位：如果共享目录让所有人可写（如 `/tmp`），要加粘滞位 `chmod +t`（显示为 1777），否则任何人都能删别人的文件。第三层 ACL：当"组"粒度不够（比如要单独给审计账号 alice 只读、给外包 bob 只写），用 `setfacl -m u:alice:r-- /data/share` 做细粒度授权，`getfacl` 验证；想让规则对新文件也生效，配 default ACL：`setfacl -m d:u:bob:rwx /data/share`，新创建的文件自动带上 bob 的权限——注意 ACL 生效时 `ls -l` 权限位末尾会显示 `+`。setuid/setgid 的安全面：setuid 程序以属主身份运行，是 Linux 提权的经典入口（suid 的 shell 脚本、可写路径的 setuid 程序都是漏洞），运维要定期 `find / -perm -4000 -type f` 审计并清理多余 suid；给脚本或服务配 sudo 时用 `visudo` 写白名单：`ops ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/sbin/nginx -s reload`，只给明确命令，杜绝 `ALL=(ALL) ALL`。实践坑：ACL 依赖文件系统支持（ext4/xfs 默认开启，NFS 挂载要看 export 选项）；`cp -a` 会保留 ACL 和 suid，`cp` 默认不带——批量拷贝权限策略时容易"丢了 ACL 还以为没问题"；给应用开目录权限时先确认进程实际用户（很多人 root 建目录、应用用 www 用户跑，写不进去）。

**延伸考点：** `ls -l` 权限位里 `s`/`S`、`t`/`T` 大小写分别代表什么（有无对应执行位）？为什么 setuid 位在 shell 脚本上被现代内核禁用？

---

### Q14. 系统启动与内核参数：开机启动流程（BIOS→bootloader→内核→systemd）、sysctl 常用参数（网络/文件句柄）

**问题：** 一台生产机器重启后服务迟迟起不来、网络也异常，你要从开机启动流程角度定位卡在哪一步；同时新服务需要调优并发连接数和文件句柄上限，改哪些内核参数、怎么持久化？

**期望加分项：**
- 能完整描述启动链路：BIOS/UEFI 自检 → bootloader（GRUB2 读 /boot/grub2/grub.cfg，支持选内核）→ 内核加载并解压 initramfs（加载磁盘/根文件系统驱动）→ 挂载根分区 → 交给 systemd（PID 1）按依赖并行启动各 target/unit
- 会用工具排查启动问题：`journalctl -b` 看本次启动日志、`systemd-analyze blame` 看各 unit 启动耗时排名、`systemd-analyze critical-chain` 看关键链路的依赖等待、`systemctl list-units --failed` 列出失败 unit
- 掌握常用 sysctl 调优及含义：`net.core.somaxconn`（监听队列上限）、`net.ipv4.tcp_tw_reuse`+`tcp_fin_timeout`（TIME_WAIT 复用）、`net.ipv4.ip_local_port_range`（本地端口范围）、`fs.file-max`（系统级文件句柄）、`net.ipv4.ip_forward`（转发）、`vm.swappiness`
- 知道持久化：临时改 `sysctl -w key=value`，永久写入 `/etc/sysctl.conf` 或 `/etc/sysctl.d/99-tuning.conf`（推荐后者，`sysctl -p` 生效，`sysctl --system` 按序加载）；改 ulimit 用 /etc/security/limits.conf 或 systemd unit 的 LimitNOFILE
- 有风险意识：改网络参数前知道默认值和影响面，改完压测验证（如 somaxconn 调大要配合应用 backlog 参数），不当调优（如盲目开 tcp_tw_recycle）会引入连接异常
- 能解释 file-max 与 ulimit 的关系：内核全局上限（fs.file-max）与进程级软/硬限制（ulimit -n）共同决定实际句柄数，`/proc/sys/fs/file-nr` 查看当前已用

**减分项：**
- 说不全启动流程，或只知道"重启"，不会用 journalctl -b / systemd-analyze 定位
- 只背 sysctl 参数名，说不出含义、默认值和适用场景
- 改完不持久化，重启即失效；或不知道 /etc/sysctl.d/ 的存在
- 混淆 fs.file-max 与 ulimit，说不清两个层级
- 盲目照抄网上"优化参数"，不验证不压测

**解答：**
启动链路按顺序讲：BIOS/UEFI 完成自检后按启动顺序找到引导盘，GRUB2 加载 /boot 下的内核镜像（vmlinuz）和 initramfs（initrd，包含磁盘控制器、根文件系统等必要驱动）；内核解压后先跑 initramfs 里的 init 脚本加载驱动、挂载真实根分区，然后切换根并启动 systemd（PID 1）；systemd 按 target 依赖关系并行拉起各 unit（network.target、multi-user.target 下的服务）。排查启动问题三板斧：`journalctl -b -p err` 看本次启动错误、`systemd-analyze blame` 找出最慢的 unit（往往是网络等待、数据库服务）、`systemctl list-units --failed` 看谁起失败，这三步能覆盖 90% 的"重启起不来"场景。内核参数调优按场景：高并发服务改 `net.core.somaxconn=4096`（默认 128，应用侧 listen 的 backlog 要同步调大）、`net.ipv4.ip_local_port_range="1024 65535"`（默认 32768 起，NAT/高并发出连接时端口不够）、TIME_WAIT 多时可 `tcp_tw_reuse=1`（注意与 NAT 场景的冲突，tcp_tw_recycle 因 NAT 丢包问题已被内核废弃，别再用）；文件句柄：`fs.file-max=1048576` 是内核全局上限，进程级用 `ulimit -n`/`/etc/security/limits.conf`（systemd 服务则配 unit 的 `LimitNOFILE=`），`/proc/sys/fs/file-nr` 三个数字分别表示"已分配/未用/上限"，句柄耗尽报 too many open files 时先看 file-nr 判断是全局还是进程级。持久化规范：调优参数写 `/etc/sysctl.d/99-custom.conf`（比改 /etc/sysctl.conf 更清晰、可整体回收），写完 `sysctl --system` 或 `sysctl -p /etc/sysctl.d/99-custom.conf` 生效。实践坑：改 somaxconn 后要同步改 nginx worker_connections、Java 的 backlog 参数，否则上限没顶到还是排队失败；VM 迁移/容器场景参数会被宿主机覆盖，要在平台层统一基线。

**延伸考点：** `systemd-analyze critical-chain` 输出的关键链路怎么解读（Waiting for 谁是瓶颈）？修改 fs.file-max 后为什么还要调 ulimit，两者谁限制谁？

---

### Q15. 安全加固：SSH 配置（禁 root/密钥登录/限 IP）、防火墙（firewalld/iptables）、最小权限原则

**问题：** 一台公网服务器被暴力破解 SSH（日志里大量 Failed password），你负责加固。请给出从 SSH 配置、防火墙到账号权限的完整加固方案，并说明如何避免把自己锁在门外。

**期望加分项：**
- SSH 加固完整：`PermitRootLogin no`（禁 root 直登）、`PasswordAuthentication no`（禁密码登录，仅密钥）、`PubkeyAuthentication yes`、`MaxAuthTries 3`、`LoginGraceTime 30`、`AllowUsers ops@10.0.0.0/24`（限来源）、`Protocol 2`；修改 `/etc/ssh/sshd_config` 后 `sshd -t` 校验语法再 `systemctl reload sshd`
- 防火墙策略：firewalld 按 zone 管理（`firewall-cmd --permanent --add-service=ssh --add-rich-rule='rule family=ipv4 source address=10.0.0.0/24 port port=22 protocol=tcp accept'`）、默认 zone 的 target 设为 drop、只放行必要端口；iptables 与 firewalld 的对应关系与共存注意
- 最小权限：普通用户 + sudo 白名单（visudo 只授必要命令）、禁 root 空密码、用户密码策略（chage -l 查看/修改过期策略）、定期清理离职账号、审计 `last`/`/var/log/secure`
- 有防锁死预案：修改 sshd 配置前先开第二个会话保持在线、`sshd -t` 验证、用 cron 兜底任务自动恢复（如配置错误 5 分钟后自动还原）
- 进阶防护：fail2ban 自动封禁爆破 IP（jail 配置、findtime/maxretry/bantime）、SSH 端口改高位（安全通过混淆，仅做辅助）、双因素认证（PAM + google-authenticator）
- 有验收意识：改完 `nmap -p 22` 或从外部实际验证访问，检查 `/var/log/secure` 爆破尝试是否归零

**减分项：**
- 只改端口不关密码登录，爆破只是换个端口继续
- 直接 `PasswordAuthentication no` 但密钥没配好，把包括自己在内的所有人都锁在门外
- 防火墙规则不清不楚，或配完没 `--reload` 不生效
- 不做最小权限，普通用户配 `ALL=(ALL) ALL` 等于没加固
- 没有兜底与回滚意识，改配置全靠赌

**解答：**
按"认证 → 网络层 → 账号层 → 兜底"四步走。认证层是核心：生产标准配置是 `PermitRootLogin no`（运维通过普通用户 + sudo 提权，root 无密钥可爆破）、`PasswordAuthentication no`（只认密钥，杜绝字典爆破）、`PubkeyAuthentication yes`、`MaxAuthTries 3`、`LoginGraceTime 30s`；如果担心误操作，分阶段执行：第一周先 `PubkeyAuthentication yes` + `PasswordAuthentication yes`（保证所有人都配好密钥）再切到 no。网络层：firewalld 里把默认 zone 的 target 设成 drop，只放行 22 和业务端口，来源限制用 rich rule 或 `firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.0.0.0/24 port port=22 protocol=tcp accept'`；老系统是 iptables，`iptables -A INPUT -s 10.0.0.0/24 -p tcp --dport 22 -j ACCEPT` + 默认 `-j DROP` 收尾（注意 firewalld 和 iptables 是两套机制，CentOS7+ 用 firewalld 就别直接写 iptables 规则，会互相覆盖）。账号层：用 `wheel` 组管理 sudo 用户，`visudo` 只授必要命令：`ops ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/sbin/nginx -s reload`，不留 `ALL=(ALL) ALL`；`passwd -l` 锁离职账号、定期 `last -20` 和 `grep 'Failed password' /var/log/secure` 审计。防锁死是必修课：改 sshd 配置前先另开一个终端保持登录、改完先 `sshd -t` 校验再 `systemctl reload sshd`，更保险的做法是留一条 cron 兜底（如 3 分钟后若 sshd 没恢复就自动回滚配置）。进阶：fail2ban 对公网暴力尝试自动封禁（`bantime=3600`、`maxretry=5`），或上 SSH 双因素。最后做验收：`nmap -p 22`、`ssh -i key user@host` 验证密钥登录、确认密码登录已拒绝、看日志爆破尝试清零。

**延伸考点：** 为什么禁止 root 直登仍然建议给 root 配置密钥（作为系统应急兜底）？firewalld 的 zone 概念与 iptables 链的关系？

---

### Q16. 批量操作与自动化：SSH 免密、ansible ad-hoc 与 playbook 基础、批量排查的脚本化

**问题：** 有 50 台机器需要"批量查看磁盘使用率、批量重启 nginx、批量收集系统信息"，人工一台台连既慢又容易漏。给出你的自动化方案，从免密登录到批量执行工具的选型。

**期望加分项：**
- SSH 免密搭建：`ssh-keygen -t ed25519` 生成密钥、`ssh-copy-id user@host` 分发公钥、`~/.ssh/config` 管理主机别名与参数（`Host web-*` 批量匹配）、`ssh-agent` 管理带口令的私钥；知道 ed25519 优于 rsa 2048
- 批量脚本基本功：for 循环 + ssh 串行执行、`ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no` 防卡死、收集每台机器的退出码与输出、`pssh`/`pdsh`/`parallel-ssh` 并行执行（-i 参数看输出、-t 超时）
- ansible 基础：inventory（INI/YAML 主机清单 + 分组）、ad-hoc（`ansible web -m shell -a 'df -h'`、`-m command` vs `-m shell` 区别、`-b` 提权）、playbook 结构（hosts/tasks/name/模块参数）、幂等性概念
- 常用模块：`command`/`shell`/`script`（跑本机脚本）/`copy`/`service`/`yum`，知道"模块化 + 幂等"是 ansible 相比裸 ssh 的核心价值
- 工程规范：批量操作前先小范围试点（`--limit` 或分批）、操作记录留日志、失败主机收集汇总（`-f` 并行数、`--forks` 调整）、免密失效的兜底（堡垒机/跳板机配置）
- 选型意识：几十台 for+ssh/pssh 足够，规模上百或需要状态收敛、配置漂移治理时上 ansible；对比 saltstack（agent/pull 模式）、puppet/chef 的适用场景

**减分项：**
- 一台台手动 ssh 去敲命令，或用密码登录脚本（密码明文写在脚本里）
- 批量脚本不设超时，一台机器卡住整个任务卡死
- 不收集失败结果，跑完不知道哪些机器成功了
- 分不清 ad-hoc 与 playbook 的适用场景，或不知道幂等是什么
- 不做试点和分批，50 台一次性全量操作出事故无法回滚

**解答：**
先解决免密：运维机器上 `ssh-keygen -t ed25519` 生成密钥对，公钥用 `ssh-copy-id` 或 ansible 的 `authorized_key` 模块分发到各机器，日常管理用 `~/.ssh/config` 定义 `Host web01`/`Host 10.0.0.*` 的主机别名、用户、端口，命令就简化为 `ssh web01 df -h`。批量执行分三个层级：小批量（<20 台）用 for 循环：`for h in $(cat hosts.txt); do ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no $h "df -h /" || echo "$h FAIL"; done`——关键点：ConnectTimeout 防单台卡死、每台输出带主机名前缀并收集失败列表；中批量用 pssh 并行：`pssh -h hosts.txt -t 10 -i 'df -h / && uptime'`，`-t` 超时、`-i` 把输出连同主机名打出来；规模化且要管状态用 ansible。ansible 的价值不在"能跑命令"，而在幂等与声明式：ad-hoc 应急快（`ansible web -m shell -a 'free -h'`、`ansible web -m service -a 'name=nginx state=restarted' -b`），playbook 做标准操作（`- hosts: web; tasks: - name: 安装 nginx; yum: name=nginx state=present`），同一 playbook 跑十遍结果一致，这正是"手工脚本跑 N 遍可能搞出 N 种状态"的对立面。工程铁律：任何批量变更先 `--limit web01` 试点一台，确认无误再全量；全量时 `--forks 20` 控制并发避免风暴；操作前 `cp` 备份配置、操作后抽查校验；免密登录本身就是一把钥匙，配合跳板机/堡垒机管理和 sudo 白名单一起用。实践坑：ansible 的 shell 模块与 command 模块区别是 shell 支持管道重定向（也意味着更危险）；批量改配置前先 `ansible web -m shell -a 'cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak'`；`StrictHostKeyChecking=no` 只应在可信内网用，公网环境是中间人攻击的入口。

**延伸考点：** ansible 的 `gather_facts` 是什么、什么场景该关掉它（性能）？幂等性对 `shell` 模块意味着什么（为什么不幂等、怎么补救）？

---

### Q17. 磁盘 IO 性能分析：iostat/iowait/await 指标、IO 瓶颈判断、大文件与随机 IO 的影响

**问题：** 数据库和文件服务部署在同一台机器上，最近 `top` 里 wa（iowait）从 5% 涨到 40%，业务响应变慢。请判断 IO 是否真的是瓶颈、瓶颈出在哪个环节，并给出进一步分析手段。

**期望加分项：**
- 能正确理解 iowait：它是 CPU 等待 IO 完成的占比，只能说明"有进程在等 IO"，不能直接等于"磁盘坏了"；要结合 iostat 看磁盘侧真实负载
- 会用 `iostat -x 1 3` 读关键列：`r/s w/s`（IOPS）、`rkB/s wkB/s`（吞吐）、`await`（单次 IO 平均耗时，含排队）、`svctm`（纯服务时间）、`%util`（设备忙占比）、`avgqu-sz`（队列深度）；SSD 的 await 通常 <5ms，机械盘 10-20ms 算正常，几百 ms 就是排队严重
- 能区分两类 IO 负载：顺序大 IO 看吞吐（rkB/s，适合日志、备份、数据仓库），随机小 IO 看 IOPS（适合数据库事务），并解释这对"要不要加磁盘"的决策影响
- 会判断饱和：%util 持续接近 100% 且 await 远大于 svctm → 排队说明饱和；avgqu-sz 大；机械盘 100% util 真实饱和，SSD/NVMe 多队列设备 %util 常虚高（参考 await 与 IOPS 上限判断）
- 能定位 IO 来源：`iotop -o` 看进程、`pidstat -d 1` 看每进程 IO、`blktrace`/`iostat -x` 结合应用日志；区分是应用自身读写、还是 swap 换页、page cache 回写（脏页比例 `/proc/meminfo` 的 Dirty）
- 有治理思路：缓存/读写分离、日志与数据分盘、调 `vm.dirty_ratio`、SSD 与机械盘的选型（IOPS 型业务上 SSD）、压测用 fio 建立基线（`fio --rw=randread --bs=4k --numjobs=16 ...`）

**减分项：**
- 只看 iowait 就下"磁盘瓶颈"结论，不看 iostat 的 await/%util
- 分不清 IOPS 与吞吐、顺序读与随机读的差异
- 不知道 %util 在 SSD/多队列设备上的局限（100% 不代表饱和）
- 不会用 iotop/pidstat -d 定位具体进程
- 无基线概念，不知道"多高算高"，答不出 fio 压测

**解答：**
先纠正判断链条：iowait 高只是现象（CPU 有进程在等 IO），瓶颈是否在磁盘要看 iostat。执行 `iostat -x 1 3` 后重点读四列：`await`（IO 平均耗时=排队+服务）、`%util`（设备忙占比）、`avgqu-sz`（平均队列深度）、`r/s w/s`（IOPS）。判断逻辑：如果 %util 接近 100% 且 await 远大于 svctm，说明 IO 在排队，磁盘确实是瓶颈；如果 %util 不高但 iowait 高，则可能是慢速设备、设备降级（RAID 重构、坏道重映射）或大量小 IO 分散。然后分清 IO 特征决定对策：数据库是典型的随机小 IO（4K-16K 随机读写，吃 IOPS），文件服务/日志是顺序大 IO（吃吞吐）——`iostat` 里 rkB/s 高且 r/s 低是顺序（一根流），r/s 高且每次量小是随机（大量散列）。若判瓶颈成立，下一步用 `iotop -o`（只显示有 IO 的进程）和 `pidstat -d 1` 定位是谁在写：常见元凶有日志风暴（应用 debug 日志爆量）、数据库 checkpoint 落盘、备份任务撞业务高峰、swap 换页（`vmstat` si/so 非零时先解决内存）。别忘了内核层因素：page cache 回写（`/proc/meminfo` 的 Dirty 字段大时调 `vm.dirty_ratio`/`vm.dirty_background_ratio`）和文件系统日志（ext4 的 journal 写放大）。治理落地：日志与数据分盘、热数据上 SSD（IOPS 型业务从机械盘换 SSD 是数量级提升）、读多场景加缓存或副本读、错峰跑批；改前用 fio 压测建立设备基线（`fio --name=test --rw=randrw --bs=16k --numjobs=8 --size=4G --runtime=60` 看 iops/latency 报告），改后对比 iostat 验证。实践坑：云盘/虚拟化环境 %util 会被宿主机调度稀释，多块盘要 `-p` 逐盘看；SSD 的 %util 100% 常见但 IOPS 未必到上限，要以 await 突增为准。

**延伸考点：** iostat 里 await 与 svctm 差距大说明什么（排队 vs 服务）？数据库磁盘用 RAID 时随机 IO 受条带大小（stripe size）影响，怎么理解？

---

### Q18. 常见故障案例：磁盘满（日志/临时文件）、文件句柄耗尽、端口冲突、DNS 解析异常的处理流程

**问题：** 值班一晚上连续收到四类告警：磁盘 100%、应用报 too many open files、服务启动报端口被占用、应用偶发"域名解析失败"。请分别给出每个故障的快速止血和根因排查流程。

**期望加分项：**
- 磁盘满流程完整：`df -h` 定位分区 → `du -xsh /* 2>/dev/null | sort -rh | head` 逐级下钻找大目录 → 区分日志文件（logrotate/清理）/临时文件（/tmp 清理策略）/已删除未释放（`lsof +L1`）→ 止血（删过期文件/扩容）→ 根治（日志轮转、配额、告警阈值 80%）
- 文件句柄耗尽：`ulimit -n` 查进程限制、`cat /proc/<pid>/limits` 看实际、`ls /proc/<pid>/fd | wc -l` 或 `lsof -p <pid> | wc -l` 统计 fd 数、`/proc/sys/fs/file-nr` 看系统全局；定位是连接泄漏（`ss -antp | grep <pid> | wc -l` 看连接数）还是文件打开未关（代码 bug），止血=重启服务 + 调大 limit，根治=修代码 + 监控 fd 趋势
- 端口冲突：`ss -lntp | grep <port>` 或 `lsof -i :8080` 查占用进程、确认是新服务撞旧服务还是僵尸进程占着、处置（停旧服务/换端口/查 SO_REUSEADDR 配置）、批量检查 `for p in 8080 8081 8082; do ss -lntp | grep ":$p "; done`
- DNS 解析异常：`dig @8.8.8.8 domain`/`nslookup` 直查权威、`getent hosts domain`（走 nsswitch 全链路，比 nslookup 更接近应用实际路径）、查 `/etc/resolv.conf`、`/etc/hosts`、本地缓存（nscd/systemd-resolved）、`/etc/nsswitch.conf` 的解析顺序；区分是 DNS 服务器挂了、网络到 DNS 不通、还是缓存污染
- 工程素养：先止血再根因（业务优先）、每类故障有固定命令链、处理完记录时间线与根因、把高发故障沉淀成 runbook/巡检脚本

**减分项：**
- 只会"重启大法"，重启完不查根因，第二天同一故障复发
- 磁盘满了不知道用 du 逐级定位、不知道 lsof +L1
- 文件句柄只调 ulimit 不查泄漏，治标不治本
- DNS 问题只用 nslookup 测一下就说"DNS 好的"（nslookup 走静态配置，和应用的解析路径可能不同）
- 处理完不做复盘、不沉淀文档

**解答：**
四类故障各给一套固定动作。磁盘满：先 `df -h` 确认哪个分区，`du -xsh /* 2>/dev/null | sort -rh | head -n 5` 逐级往下（-x 限同一文件系统），找到大目录后区分性质：应用日志直接配 logrotate + 删除旧轮转文件；/tmp 的临时文件按 mtime 清理（`find /tmp -type f -mtime +7 -delete`，先预览）；最隐蔽的是"文件已删空间不还"——`lsof +L1 | head` 找仍持有已删除文件 fd 的进程，重启该进程即释放。止血同时把 80% 告警、日志轮转、清理策略一起落，避免复发。文件句柄：先量化——`cat /proc/<pid>/limits` 看软硬限制、`ls /proc/<pid>/fd | wc -l` 数当前 fd、`ss -antp | grep <pid> | wc -l` 数连接数：如果连接数≈fd 数且都接近 limit，说明是连接泄漏（长连接没释放）；如果 fd 数远大于连接数，则是文件打开未关（代码 bug）。止血是重启 + 临时调 `ulimit -n`（systemd 服务改 unit 的 LimitNOFILE），根治是修代码 + 给 fd 数加监控告警。端口冲突：`ss -lntp | grep :8080` 看谁占用，先确认是不是同一个服务的旧实例（常发生在滚动发布时新实例先于旧实例被杀）、还是别的服务硬撞端口，处置是停旧或换端口，并约定端口分配登记表。DNS 解析异常：先 `getent hosts api.example.com` 验证应用实际解析路径（nslookup/dig 走系统静态配置，应用可能走 nsswitch 的缓存或 hosts 优先），再看 `/etc/resolv.conf` 的 nameserver 是否可达（`nc -vzu <dns> 53`）、查 nscd/systemd-resolved 缓存是否污染（`systemd-resolve --flush-caches` 或重启 nscd）、检查 `/etc/hosts` 是否有过期条目。每类处理完补一条：写进 runbook、加监控与自动巡检，让"值班被同一块石头绊倒两次"的概率归零。

**延伸考点：** `nslookup` 与 `getent hosts` 的结果不一致时说明什么（解析链路差异）？服务 fd 数缓慢爬升但从未到上限，要不要处理（连接泄漏的早期信号）？

---

### Q19. 服务高可用基础：keepalived/VRRP 虚拟 IP、负载均衡（nginx）基础、健康检查

**问题：** 一个核心服务目前单机部署，要做高可用。请设计一套"VIP 漂移 + 负载均衡 + 健康检查"的方案，并说明故障切换的机制与时间量级、以及最怕出现的脑裂问题。

**期望加分项：**
- keepalived/VRRP 机制讲透：两台（或多台）节点通过 VRRP 组播周期性交换优先级，主节点（MASTER）持有虚拟 IP（VIP）对外服务，主节点故障/失联超过阈值后 BACKUP 节点接管 VIP；`priority` 与 `preempt`/`nopreempt` 控制抢占策略
- 能写核心配置：`vrrp_instance` 里的 `state`/`interface`/`virtual_router_id`（同一组必须一致）/`priority`/`virtual_ipaddress`/`advert_int`（通告间隔，默认 1s，决定切换时间量级约 2-3s）/`vrrp_script` + `track_script`（业务健康检查，如 `curl -f http://127.0.0.1:8080/health`，脚本失败则降权）
- 健康检查分层：进程级（检查端口）/业务级（检查 /health 接口返回值）/数据库级（检查主从状态），越往业务层越真实但开销越大；nginx 的 upstream 健康检查：`max_fails`+`fail_timeout` 把故障节点摘除、`proxy_next_upstream` 故障转移
- nginx 负载均衡基础：`upstream backend { server 10.0.0.1:8080 weight=3; server 10.0.0.2:8080; }` + `proxy_pass http://backend`，策略（轮询/least_conn/ip_hash/url_hash）、`keepalive` 连接复用
- 能讲脑裂与防范：主备之间心跳中断但双方都活着，各自抢到 VIP（双 master）导致流量分叉、数据双写；检测手段（心跳线冗余、仲裁机制、VIP 绑定 ARP 通告、第三方仲裁如 DB/API 探测）、keepalived 的 `notify` 脚本在状态切换时做业务联动（释放/抢占资源）
- 有架构与量化意识：切换时间=advert_int 的 3 倍左右（秒级）、应用层要容忍重连（连接池、重试）、数据库高可用（主从 + 半同步）比应用 HA 更难、同机房 vs 跨机房（跨机房要仲裁防止双活双写）

**减分项：**
- 只背 keepalived 名词，说不清 VRRP 通告机制和 VIP 漂移过程
- 健康检查只做"端口通不通"，服务假死（进程在、业务 500）检测不到
- 完全不知道脑裂是什么、怎么防范
- 切换时间量级没有概念（以为"秒级切"就万事大吉，不处理客户端重连）
- 把应用高可用和数据库高可用混为一谈

**解答：**
分层设计：入口层是 VIP（keepalived），转发层是 nginx 负载均衡，后端是服务多副本。keepalived 的原理：主备节点通过 VRRP 协议周期性（advert_int，默认 1s）组播通告优先级，BACKUP 连续 3 个通告周期收不到 MASTER 的包就认定主节点故障，接管 VIP 并发送免费 ARP 通知交换机刷新转发表——所以切换时间量级是 2-3 秒（3 倍通告周期），这是设计时就要接受的现实：客户端要有重连/重试机制。核心配置骨架：`vrrp_instance VI_1 { state MASTER; interface eth0; virtual_router_id 51; priority 100; advert_int 1; virtual_ipaddress { 10.0.0.100/24 dev eth0 } vrrp_script chk_health { script "/etc/keepalived/check.sh"; interval 2; fall 2; rise 2 } track_script { chk_health } }`，备机 priority 99、state BACKUP；check.sh 里做业务健康检查（`curl -sf http://127.0.0.1:8080/health || exit 1`），脚本失败时 keepalived 降权让 VIP 漂到备机——这解决了"进程活着但业务已死"的假死问题，是健康检查的分层价值：端口探测只能发现进程挂了，业务探活才能发现应用卡死。nginx 层：`upstream backend { server 10.0.0.1:8080 max_fails=3 fail_timeout=10s; server 10.0.0.2:8080; }`，后端连续失败 3 次被摘除 10 秒，配合 `proxy_next_upstream http_502 http_503` 做请求级故障转移；业务无状态时轮询即可，有会话粘性需求用 ip_hash。最怕的脑裂：主备心跳中断（网卡/交换机故障）导致备机也升主，两个节点同时持有 VIP，客户端流量被 ARP 分流到两台、数据库双写——防范手段：心跳用独立链路（双心跳线）、部署在带隔离机制的网络（云厂商的 keepalived 通常受限，用 SLB）、关键场景加第三方仲裁（如数据库表/对象存储锁，谁抢到锁谁才升主），keepalived 的 notify 脚本在状态切换时执行资源接管/释放脚本。数据库高可用另说（主从复制 + 半同步 + 自动切换），比应用层 HA 复杂度高一个量级，架构设计时应用要尽量无状态，把状态下沉到数据库。

**延伸考点：** VRRP 的 virtual_router_id 不一致会发生什么（分组错乱）？nginx 的 upstream 里 `backup` 参数与主备 keepalived 的语义差异？

---

### Q20. 从脚本到自动化平台：脚本规范（参数化/日志/退出码）、配置管理工具选型、运维平台演进路线

**问题：** 团队从"一个人手工运维"发展到"十几人服务几十套环境"，日常操作还是靠个人脚本和口头传口令，经常出现"上次谁改了什么没人知道"。请规划一条从脚本到自动化平台的演进路线，并给出每个阶段的关键交付物。

**期望加分项：**
- 脚本规范化：参数化（`getopts`/`$1` 带默认值与校验）、统一日志（时间戳+级别+落盘）、明确退出码（0 成功/非 0 失败、失败即停）、幂等设计（重复执行结果一致）、版本控制（git 管理脚本与配置）、错误处理与 trap 清理
- 配置管理选型逻辑：先 ansible（无 agent、SSH 直连、声明式 + 幂等，最适合从零起步）解决"配置漂移"，后按需引入 Terraform 管云上基础设施（IaC）、CMDB/资产登记管元数据；对比 salt/puppet 的 agent 模式适用大规模；避免一上来就自研平台
- 演进路线清晰：阶段一脚本库（git + 规范 + runbook）→ 阶段二配置管理（ansible playbook 标准化部署与巡检、定期"对账"修正漂移）→ 阶段三 CI/CD（代码提交触发构建测试、流水线发布、灰度与回滚、变更审批留痕）→ 阶段四平台化（操作 Web 化/API 化、权限与审批流、发布记录与审计、自助服务）→ 阶段五可观测（监控告警、日志中心、链路追踪、SLO/值班体系）
- 有工程治理：变更管理（审批+灰度+回滚预案）、配置与代码分离（环境差异走变量/密钥管理 Vault）、发布留痕（谁在什么时间对哪台机器做了什么）、定期演练（恢复流程跑一遍）
- 能讲平台化的坑：平台是工具不是银弹（流程僵化/过度设计）、先有标准和数据再上平台、平台建设以"消除重复操作"为验收标准、API 化让脚本可被平台调用而不是被取代

**减分项：**
- 一上来就要自研运维平台，无视现有脚本资产的迁移成本
- 脚本无参数化、无退出码、不幂等，是"一次性脚本"没法复用
- 不先解决配置漂移和一致性，直接上平台（平台管的是"脏环境"）
- 所有操作不做留痕和审批，出了事查不到责任人
- 无选型依据，盲目对比工具或跟风新技术

**解答：**
演进路线分五步，每步有明确交付物和验收标准。第一步脚本工程化：把散落的个人脚本收进 git 仓库，定规范——`getopts` 解析参数并校验必填项（`: "${ENV:?ENV is required}"`）、统一日志函数（时间戳 + 级别 + 输出到日志文件）、显式退出码（成功 0、失败非 0 并输出错误原因）、操作幂等（先查状态再执行，如"已启动则跳过"）、删除/破坏性操作先确认。这一步的验收是"任何人跑任何脚本都能预期结果"。第二步配置管理：引入 ansible，把环境初始化、软件安装、配置文件模板、巡检（磁盘/负载/证书过期）写成 playbook，inventory 分组对应环境（dev/staging/prod），关键诉求是幂等对账——定期 `ansible-playbook` 全量执行一遍，自动把被手改漂移的配置收敛回标准。云上资源用 Terraform 管 IaC，与 ansible 分工（Terraform 管基础设施，ansible 管配置与应用）。第三步 CI/CD 化：脚本/playbook 纳入流水线，提交即触发检查（shellcheck + ansible-lint）、测试环境验证、生产发布走审批 + 灰度（先一台再全量）+ 可回滚（发布前备份/镜像 tag），所有发布动作自动留痕。第四步平台化：把高频操作（发布、扩容、巡检、回滚）暴露为 Web 页面/API，接权限系统（谁能对哪套环境执行什么操作）、审批流和审计日志，前端是"门户 + 自助服务"，后端调用的还是第三步沉淀的 playbook——平台不是替代脚本，而是把脚本变成受控的服务。第五步可观测与 SLO：监控告警、日志中心、链路追踪打通，故障响应从"人肉排查"变"告警→runbook→自动止血"。全程贯穿的原则：先标准后工具、先数据后平台、变更留痕、定期演练。最大的坑是第四步没到就自研平台，管着一堆"没标准的环境"，平台只会固化混乱。

**延伸考点：** ansible 管理配置漂移的原理（幂等 + 声明式）与"脚本检测-修正"模式的差异？运维平台建设中 CMDB（资产配置库）的定位和常见失败原因？

---

### Q21. 线上服务器 CPU/磁盘/内存同时异常，从现象到根因的现场排查演练（工具链、时间线、结论）

**问题：** 某台线上业务机 22:00 起监控同时告警：CPU 使用率 100%、磁盘 IO 等待飙升、可用内存骤降，业务接口超时。你作为值班运维，如何在现场按时间线系统排查，最终定位根因并复盘？

**期望加分项：**
- 有时间线意识：先 `date` 并保存现场快照（top/vmstat/iostat/free/sar 落盘），再逐层排查，避免"重启大法"毁掉证据
- 工具链组合：`vmstat 1` 看 r/b/si/so 区分 CPU 忙/IO 忙/内存换页，`iostat -x 1` 看 await/%util，`free -h` 看 available 而非 free，`ss -s` 看连接数
- 能建立因果链：先有 CPU 风暴还是先有磁盘瓶颈，用 `pidstat`/`iotop`/`perf` 定位到具体进程与线程
- 会回溯历史：`sar -q -u -r -d -f /var/log/sa/saXX` 对齐 22:00 前后各资源曲线，确认异常先后顺序
- 有止损与复盘：先止血（限流/摘流量/扩容），根因定位后给出防治措施（错峰、资源限制、监控补齐）并沉淀 runbook

**减分项：**
- 一上来就重启，或只盯单一指标（只看 CPU 不看 IO/内存）
- 分不清"CPU 高导致内存被换页"还是"磁盘慢导致 CPU 等待"的因果方向
- 不留现场证据，事后无法复盘
- 不会用 sar 回溯，只能靠当前瞬间判断
- 止血与根因不分，处理完就完事不沉淀

**解答：**

处理"多资源同时异常"的现场，第一原则是"先存证据再动手"：执行 `date` 记录时间点，立即把 `top -b -n 3`、`vmstat 1 5`、`iostat -x 1 5`、`free -h` 的输出落盘，再判断是否需要止血（接口超时先摘流量或限流）。第二步用 vmstat 的 r/b/si/so 四列定性：r 高且 us/sy 高，方向是 CPU；b 非零且 wa 高，方向是 IO；si/so 持续非零说明内存不足在换页——但三者常常互为因果：磁盘 IO 慢拖慢读写，进程堆积导致内存膨胀，内存不足触发 swap 换页又加剧 IO，形成"IO 慢→内存换页→更慢"的恶性循环，此时单看一个指标必然误判。第三步用 `pidstat -t` 和 `iotop -o` 下钻到具体进程：本场景最常见根因是 22:00 定时任务启动了全量日志扫描/数据备份，吃满 CPU 与磁盘 IO，同时应用日志风暴把 page cache 挤出、进程开始 swap。第四步用 `sar -q -u -r -d -f /var/log/sa/sa22` 回溯确认时间线：哪个指标先抬头——"先日志量暴涨→磁盘 IO 先升→CPU 后升"和"先 CPU 风暴→IO 排队被动升高"是两种完全不同的根因，历史曲线才能给出答案。实践中的坑：一是只看当前快照（故障往往 20 分钟前已开始，现场是连锁反应的结果）；二是把 IO 等待误判为 CPU 瓶颈去盲目加 CPU；三是没有给定时任务做资源限制（nice/ionice）和错峰导致周期性拖垮系统。结论落到防治：定时任务错峰 + `ionice -c3`、日志轮转与告警、available 内存监控，把这次演练沉淀成可复用的 runbook。

**延伸考点：** 如何用一条 sar 命令同时拿到故障时段 CPU/内存/IO/网络历史曲线并判断因果先后？定时任务触发 IO 风暴时，ionice 与 nice 分别控制什么？

---

### Q22. 需要批量替换 100 台服务器上的某个应用版本，设计一套安全可回滚的脚本化方案

**问题：** 你需要在一小时内把 100 台线上服务器的业务应用从 v1.2 升级到 v1.3（安装包已备好），要求尽量不停服、失败能快速回滚。请设计一套完整的脚本化批量发布方案。

**期望加分项：**
- 分层设计：准备（校验包完整性、灰度预演）→ 分批发布（先边缘后核心，每批验证再下一批）→ 回滚预案（旧版本保留、回滚脚本预写）
- 脚本要素：参数化（版本号/批次/机器列表）、`set -Eeuo pipefail`、日志带时间戳落盘、退出码校验、幂等（已是最新版本则跳过）
- 批量执行与并发控制：ansible/pssh 或 for+ssh，`--forks`/并行度控制，`ConnectTimeout` 防卡死，失败主机收集汇总不中断
- 安全回滚：发布前备份（旧包/配置）、回滚脚本与发布脚本成对、支持"一键回滚到上一版本"
- 验证机制：健康检查（/healthz、端口、进程、版本号）、发布中摘流量/发布后挂回、灰度比例与观察时间
- 兜底与沟通：先 1 台试点、值班盯监控、明确发布窗口与回滚决策人

**减分项：**
- 100 台一次性全量发布，出问题全线挂
- 没有回滚预案，失败了只能手工恢复
- 脚本不幂等、不校验退出码，部分机器半新半旧
- 无健康检查，发布完靠用户投诉发现
- 不保存旧版本，回滚时找不到原包

**解答：**

核心思路是"分批、灰度、可回滚"，而不是"100 台一把梭"。第一步准备：下载并校验安装包（`md5sum -c` 比对发布清单），把机器按角色分组（web-01~10 边缘、api-01~50 核心），写好成对的 `deploy.sh` 与 `rollback.sh`：脚本开头 `set -Eeuo pipefail`，参数化接收版本号与批次，日志统一 `log() { echo "$(date '+%F %T') [$HOSTNAME] $*" >> /var/log/deploy.log; }`，关键动作（停服务→备份旧包到 /opt/app/backup/v1.2→解压新包→启动→健康检查）每步校验退出码，失败即停并记录。第二步灰度：先用 `--limit` 只发 1 台，跑完健康检查（进程存活 + `curl -sf http://127.0.0.1:8080/healthz` + 版本号核对 `app --version`）确认无误，再按"边缘 10 台→核心 30 台→剩余 60 台"分批，每批间隔 5-10 分钟观察监控（错误率、延迟、CPU）。第三步批量执行：用 ansible `-m script` 把脚本分发到各机本地执行（失败重跑天然幂等），`--forks 20` 控并发避免风暴，`ansible web -m script -a 'deploy.sh v1.3'` 后收集每台退出码，失败主机单独重试或人工介入，绝不"一台失败继续全量"。第四步回滚预案：旧包已备份在每台机器本地，回滚脚本与发布脚本对称（停服务→恢复备份目录→启动→健康检查），观察期内错误率超阈值（如 1%）立即全量回滚，回滚决策人在发布窗口内待命。实践中的坑：健康检查只看端口会漏掉"进程活着但业务挂"（必须探业务 /healthz）；并发太高会拖垮配置中心与数据库连接数；版本号校验要放进健康检查里，防止"以为发布了实际没生效"——上线前用一台机器完整演练一遍发布与回滚，把"预计 1 小时"变成"实际 20 分钟"。

**延伸考点：** 发布脚本如何保证幂等（重复执行不产生脏状态）？若 30 台发布成功后触发回滚，这 30 台与未发布的 70 台如何保证最终一致？

---

### Q23. 一次"诡异"的定时任务失效/服务自动重启问题，完整定位过程与根因

**问题：** 最近一周，一台生产服务器的服务每天凌晨 3 点左右"自动重启"，但 crontab 里没有配置任何重启任务；同时另一个定时备份任务"时灵时不灵"。请给出完整的定位过程与可能的根因。

**期望加分项：**
- 证据优先：`journalctl -u <service> --since "today 02:30"` 看服务停止/启动的确切时间与原因，`journalctl -b -1 -e` 翻上一个 boot 会话
- 排查面广：systemd 的 Restart 策略 + StartLimitBurst 被触发、OOM-Killer（`dmesg -T | grep -i -E 'oom|killed process'`）、内存不足触发 swap、cron 与 systemd timer 的时区/环境差异、磁盘满导致服务写日志异常
- 时间线对账：把"服务重启时间"与"cron 任务执行时间""监控告警时间"对齐找相关性
- 会查 crontab 全貌：`crontab -l`、`/etc/crontab`、`/etc/cron.d/`、`/var/spool/cron`、`grep CRON /var/log/cron` 看实际执行记录
- 能识别"自动重启"假象：OOM 杀进程后 systemd 按 Restart= 策略自动拉起，表现为"每天定点重启"
- 有复盘意识：根因确认后给防治（错峰、oom_score_adj 保护、cron 环境显式化、内存监控）

**减分项：**
- 只看现象不查日志，或只看应用日志不看系统日志
- 不知道 OOM-Killer 会把进程"杀掉又被 systemd 拉起"造成"自动重启"的假象
- 不查 journalctl -b / dmesg，全靠猜
- crontab 只看 `crontab -l`，漏掉系统级 cron.d 与 timer
- 不建立时间线，找不到因果

**解答：**

"服务自动重启"和"定时任务时灵时不灵"大概率指向同一根因：凌晨 3 点前后的系统内存压力引发的连锁反应。定位从"找证据"开始而非猜。第一步 `journalctl -u myapp --since "today 03:00" --until "today 03:10"` 看服务停止/启动的精确时刻与 journal 记录的停止原因（systemd 会记录正常停止还是异常退出），再 `journalctl -b -1 -e` 翻上一 boot 会话尾部。第二步查系统级证据：`dmesg -T | grep -i -E 'oom|killed process'`——若凌晨 3 点有 OOM-Killer 记录，"真凶"就浮出水面：内存不足时内核按 oom_score 选进程杀掉，而 systemd 配了 `Restart=on-failure/always`，进程被杀后几秒内又被自动拉起，表象就是"服务每天定点自动重启"；"时灵时不灵的备份任务"则是同一时刻另一个 cron 任务（如全量备份）与业务服务叠加把内存顶爆，或 cron 脚本因 PATH/绝对路径问题静默失败。第三步对账时间线：`grep CRON /var/log/cron` 确认凌晨 3 点有哪些 cron 真实执行，`sar -r -f /var/log/sa/saXX` 看内存曲线是否在同一时刻触顶，三者对齐后因果清晰：凌晨 3 点备份任务 + 业务高峰叠加 → 内存耗尽 → OOM 杀进程 → systemd 自动拉起（"自动重启"）→ 备份任务因内存不足/被杀而失败（"时灵时不灵"）。第四步验证与防治：`free -h` 看当前内存，给关键服务设 `oom_score_adj=-500` 保护、备份任务错峰到 4 点并 `ionice` 降优先级、cron 脚本内显式 export PATH 与绝对路径、加 available 内存监控告警。实践中的坑：只看应用日志永远看不到 OOM（它不写业务日志）；"自动重启"往往不是有人配了重启，而是 Restart 策略与 OOM 的组合；时区差异（cron 用系统时区，CST/UTC 差 8 小时）会让"凌晨 3 点"实际在另一时刻执行。复盘结论要落到机制：内存水位告警 + 关键进程保护 + 定时任务错峰资源隔离，让"诡异"变成"可预期"。

**延伸考点：** systemd 的 Restart 策略与 OOM-Killer 如何共同造成"服务自动重启"的假象，如何区分"主动重启"与"被拉起"？cron 任务的时区问题（CST/UTC）如何排查与规避？

---
