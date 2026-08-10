# DevOps · 容器化（面试题库）

本文件面向 DevOps、SRE 与后端工程师，考察候选人在容器化工程落地上的真实能力：镜像构建与瘦身、运行时隔离原理、网络与存储、资源限制与稳定性、安全加固，以及存量应用的迁移改造。不考八股文定义，而是以线上事故、发布流程和性能问题场景切入，重点看候选人能否给出量化依据、讲清方案取舍，并用真实命令和排查过程佐证自己的实践。题目难度从实践基础到架构级渐进。

---

### Q1. 镜像分层和联合文件系统是什么？为什么两个镜像只是 Dockerfile 前几行不同，就能共享大部分存储？

**问题：** 团队几十个服务的镜像加起来占满了构建机的磁盘，有人提议把每个镜像"重新从零构建"来省空间。你如何解释镜像分层机制，并说明为什么正确做法是复用基础层？

**期望加分项：**
- 能说清分层原理：每一条 RUN/COPY 指令生成一个只读层，层之间通过 overlayfs 的 lowerdir 叠加，容器写操作走 copy-on-write（写时复制）到 upperdir
- 能给出量化概念：不同镜像共享基础层（如同一个 base image + JDK 层），拉取时逐层去重，磁盘上只存一份
- 能说出镜像 ID 是内容寻址（layer digest），只有内容变化才会产生新层，COPY 内容不变则 digest 不变、层可复用
- 能结合线上实践：用 `docker image ls` 看共享层、`docker history` 看层构成、迁移镜像仓库时逐层并行拉取的加速收益
- 主动考虑边界：分层多了镜像变"胖"、overlayfs 的层数限制、以及删除中间层不会释放空间的问题

**减分项：**
- 只背"分层"两个字，讲不清 overlayfs 的读写流程和 COW 机制
- 说不清"层复用"为什么能同时省磁盘和加速拉取（增量传输）
- 忽略镜像 ID 与内容寻址的关系，以为"同一 Dockerfile 每次构建都产生新镜像"
- 无实践佐证，答不出 docker history / docker image inspect 这类命令

**解答：**
先给判断：磁盘占用高的根源往往不是"镜像太多"而是"重复构建产生了大量内容相同的层"和未清理的悬空镜像，绝不是重新构建能解决的。分层机制：Dockerfile 的每条指令生成一个只读层，overlayfs 把多个 lowerdir 与一个可写 upperdir 合并成统一视图，容器内文件写入先查 lowerdir、写时复制到 upperdir，删除文件则通过 whiteout 标记遮蔽。层复用依赖内容寻址：层以 digest（内容哈希）标识，基础镜像 + JDK 层只要内容不变，digest 就不变，几十个业务镜像共享同一批底层，磁盘只存一份、拉取也只传一次。所以正确做法是：统一、固定的基础镜像，把变化的业务层放在最后（依赖层置前），并定期 `docker image prune -f` 清理悬空镜像。实践坑：很多人以为删除 Dockerfile 中间的 RUN 会减小镜像，其实只是产生新的中间层，旧层还躺在仓库里；用 `docker buildx build --metadata-file` 或 `docker history --no-trunc` 才能看清层构成；另外 overlayfs 层数过多（默认 128）和"删文件但层还在"导致镜像不减反增，都是高频翻车点。

**延伸考点：** 既然层只读，容器内改了 /etc/passwd 后重启容器为什么又还原了？overlayfs 的 whiteout 文件是怎么标记删除的？

---

### Q2. 多阶段构建具体怎么给镜像瘦身？编译环境和运行环境为什么要分离？

**问题：** 你们的 Java/Go/前端镜像动辄 1GB+，有人在 Dockerfile 里用同一个镜像既装编译工具又跑程序。请用多阶段构建重构这个 Dockerfile，并说明各阶段的作用。

**期望加分项：**
- 能给出标准的 FROM ... AS builder / FROM ... 运行阶段的写法，运行阶段只拷产物（COPY --from=builder）
- 能说清收益的量化口径：编译期依赖（GCC、JDK 全套、npm 依赖、go mod 缓存）与运行期依赖的差距，举例如 alpine 的 Go 镜像从 800MB 降到 30MB 以内
- 能结合语言特性：Go 静态编译可跑 scratch；Java 用 jlink 裁剪 JRE 或用 eclipse-temurin 的 slim 变体；前端构建产物配 nginx 静态镜像
- 能指出缓存策略：构建阶段放最后、利用 BuildKit 的 --mount=type=cache 缓存依赖目录
- 主动考虑边界：剥离了构建工具后调试不便（可留调试专用 stage），distroless/scratch 对 glibc 和 CA 证书的处理

**减分项：**
- 只背"多阶段构建能瘦身"，写不出 COPY --from 的具体用法
- 分不清编译期依赖和运行期依赖，或照抄精简镜像导致程序起不来（缺 libc、缺证书）
- 忽略了构建缓存，导致每次构建都全量重编译
- 无量化概念，说不出瘦身前后的大致体积变化

**解答：**
核心思路一句话：编译环境只负责产出，运行环境只负责执行，两者通过"产物"交接。以 Go 为例：第一阶段 `FROM golang:1.22 AS builder`，`RUN CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /app/server ./cmd/server`，得到静态二进制；第二阶段 `FROM scratch`（或 distroless），`COPY --from=builder /app/server /server`，再加 `COPY --from=... /etc/ssl/certs/ca-certificates.crt` 处理 HTTPS 证书，最终镜像只有几 MB 到 30MB。Java 场景：builder 用完整 JDK 做编译，运行阶段用 `RUN jlink --add-modules java.base,java.sql ... --strip-debug --no-man-pages` 裁剪出 40MB 左右的 JRE，或直接用 temurin 的 jre slim 镜像。前端场景：node 阶段 npm ci && npm run build，第二阶段 `FROM nginx:alpine` 只 COPY dist 和 nginx.conf。实践坑：Go 程序只要用了 net 包解析 DNS 就需要 glibc 或 cgo 关闭后的纯静态；用了 sqlite/证书库的要么留 cgo、要么改用 distroless；`ldd` 检查动态依赖后再选 scratch 才能避免"镜像很小但起不来"的尴尬。多阶段构建同时配合 `--mount=type=cache` 缓存 go mod / npm 缓存目录，可把 CI 构建时间再压一个量级。

**延伸考点：** jlink 裁剪 JRE 时怎么确定需要哪些模块？distroless 镜像里怎么进容器排查问题（debug 镜像方案）？

---

### Q3. Dockerfile 里指令顺序、COPY 和 ADD 该怎么用？为什么依赖要放在前面？

**问题：** 你们的 CI 每次构建都几乎全量重跑，部署节奏被构建时间拖住。有人把 Dockerfile 改得乱七八糟后更慢了。请给出一个"构建缓存友好、层数合理"的 Dockerfile 最佳实践。

**期望加分项：**
- 能说清缓存失效规则：RUN/COPY 的层缓存按"指令内容 + 上下文文件变化"校验，COPY 内容变了其后的所有层全部失效
- 能给出依赖层置前的经典写法：先 COPY package.json / go.mod 再 RUN install，最后才 COPY 业务代码
- 能分清 COPY 与 ADD：COPY 只做拷贝，ADD 隐含解压 tar 和远程 URL 拉取（后者不缓存、有安全风险，默认禁用）
- 能结合 .dockerignore 说明：把 node_modules、.git、日志排除出构建上下文，避免上下文大、缓存频繁失效
- 能给出"合并 RUN 减少层数 vs 保持可读性"的取舍，以及 apt-get 前先 update 的坑（update 和 install 拆开会缓存过期包列表）

**减分项：**
- 只会背"少用 RUN 多用 COPY"之类的口号，讲不清缓存失效的真正触发条件
- 分不清 COPY 和 ADD 的语义差异
- 忽略 .dockerignore 的作用，把整个 repo 塞进构建上下文
- 不会处理 apt/apk 缓存和包列表过期的组合问题

**解答：**
构建快慢的关键是层缓存命中率。机制上：Docker 按"指令文本 + 受影响文件内容"计算缓存 key，一条 COPY 的内容变了，从该指令开始往后的所有层全部重建。所以原则是把"变动少的放前面、变动多的放最后"。Node 应用的标准写法：`FROM node:20-alpine` → `WORKDIR /app` → `COPY package.json package-lock.json ./` → `RUN npm ci`（依赖层，只有锁文件变了才重装）→ `COPY . .` → `RUN npm run build`。Go 同理：先 `COPY go.mod go.sum ./` 再 `RUN go mod download`，业务代码最后拷入。.dockerignore 必须包含 node_modules、.git、dist、*.log，否则每次代码提交都带着几十 MB 无关文件进上下文，缓存 key 跟着失效，且发送上下文本身就很慢。层数方面：不是越少越好，而是"一条 RUN 内多个命令用 && 合并、但别把不相关的事硬揉在一起"，同时清理中间产物（`apt-get install ... && rm -rf /var/lib/apt/lists/*`）。实践坑：`RUN apt-get update` 和 `apt-get install` 拆成两条会缓存过期的包列表导致安装失败，必须合并；`npm ci` 比 `npm install` 严格走 lockfile，才能保证依赖层缓存真正稳定；ADD 的远程 URL 特性会绕过缓存且无法保证内容一致性，生产环境一律用 COPY + curl。

**延伸考点：** BuildKit 的 `--mount=type=cache` 和传统层缓存有什么区别？如何用 `docker buildx build --target` 只构建某个 stage 加速调试？

---

### Q4. 容器和虚拟机到底差在哪？什么场景容器不该硬上？

**问题：** 有人主张"把所有服务都容器化，虚拟机淘汰掉"，另一部分服务（如内核模块、特殊网卡驱动）在容器里跑不起来。你怎么看这个争论？

**期望加分项：**
- 能说清隔离边界：容器靠 namespace + cgroup 做资源隔离，共享宿主机内核；虚拟机有独立内核，靠 hypervisor 隔离
- 能给出性能差异的量化认识：容器几乎无虚拟化开销（除特定 syscall 场景），VM 有内核启动时间和固定资源占用，但两者在现代硬件上差距越来越小
- 能讲清安全边界差异：容器逃逸会直接打到宿主内核，VM 逃逸需突破两层，强隔离要求（多租户、合规）下 VM 更稳
- 能结合实践选型：无状态 Web 服务、CI 构建、微服务适合容器；需要自定义内核/驱动、强多租户隔离、裸机性能的场景保留 VM/物理机
- 主动考虑边界：Docker in Docker、特权容器、混合调度（K8s 上跑 Kata/runV 这类安全容器）等折中方案

**减分项：**
- 把"共享内核"和"独立内核"讲反，或误以为容器是"轻量虚拟机"
- 只谈性能不谈安全边界，忽略多租户场景下逃逸风险
- 一刀切主张"全容器化"或"全 VM"，给不出场景化判断
- 无实践佐证，说不出自己迁移或保留 VM 的真实理由

**解答：**
先给框架：隔离对象不同。容器是进程级隔离——PID/Network/Mount/UTS/IPC/User namespace 隔离进程视图，cgroup 限制资源用量，但全部共享宿主内核，一个容器里 `uname` 看到的是宿主内核版本；虚拟机通过 hypervisor 提供完整虚拟硬件和独立内核，隔离边界是硬件级。由此推出两个关键结论：一是性能上，容器几乎没有虚拟化开销（同内核、同 syscall），启动是毫秒级、密度高，但共享内核意味着一个容器拖垮宿主（如 fork 炸弹、内核 bug）会影响同宿主所有容器；二是安全上，容器逃逸（如通过漏洞拿到宿主内核权限）风险远高于 VM 逃逸，所以公网多租户场景默认给 VM，容器只跑可信工作负载。工程选型：无状态微服务、CI 构建、批处理任务用容器（K8s 调度）；需要加载内核模块、定制内核参数、跑特殊驱动的用 VM；强合规隔离用 VM 或 Kata 这类安全容器（内部是轻量 VM）。实践坑：很多人把"容器启动快"当性能优势，其实对常驻服务意义有限，真正的优势是镜像分发、环境一致和弹性调度；而"容器里改 sysctl、挂载 /proc、开特权模式"这类操作基本等于放弃隔离，等于在 VM 里跑了个普通进程。

**延伸考点：** gVisor / Kata Containers 是如何兼顾容器的体验和 VM 的隔离的？什么信号下你会把跑在容器里的服务迁回 VM？

---

### Q5. namespace 和 cgroup 各自管什么？容器里 `top` 为什么看不到别的进程？

**问题：** 面试者说容器就是"把进程装进沙箱"。请具体解释：一个容器进程启动后，为什么 `ps` 看不到宿主机其他进程？为什么一个容器可以吃掉所有 CPU 而另一个容器只能干瞪眼？

**期望加分项：**
- 能列出主要 namespace 及其职责：PID（进程视图）、Network（网络栈）、Mount（文件系统挂载点）、UTS（主机名）、IPC（消息队列/信号量）、User（UID/GID 映射）
- 能说清 cgroup v2 的层次结构：CPU、memory、io、pids 等控制组，配合 systemd 的 slice/scope 管理
- 能联系容器创建流程：runc 基于 OCI 规范 clone + 设置 namespace + 写 cgroup，Docker 默认给容器建独立 namespace
- 能结合故障排查：容器内 `top` 默认只看容器命名空间（或需要 lxcfs 伪装 /proc 才看到 CPU 份额）；`ps aux` 在 PID namespace 里只显示容器内进程
- 能指出边界与坑：cgroup v1 与 v2 的差异（v2 统一 hierarchy）、/proc 是宿主的导致监控数据失真、user namespace 是否开启影响特权操作

**减分项：**
- 把 namespace 和 cgroup 的功能讲混（以为 cgroup 管隔离、namespace 管资源）
- 说不全 namespace 种类，或不知道 PID namespace 里 ps 只能看到自己命名空间的进程
- 不知道 cgroup 树怎么配、怎么查（/sys/fs/cgroup 或 systemd-cgls）
- 无实践佐证，答不出"容器内 top 的 CPU% 为什么不可信"这类经典坑

**解答：**
一句话分功：namespace 管"看得到什么"（隔离视图），cgroup 管"能用多少"（限制资源）。进程启动时，runc 按 OCI 规范用 clone 创建进程并指定 CLONE_NEWPID/NEWNET/NEWNS 等 flag，进程便进入了独立的命名空间：PID namespace 里该进程是 PID 1，`ps`/`top` 读的 /proc 在容器内只映射到本命名空间，所以看不到宿主机进程——这也是容器内 `top` 的 CPU% 经常失真（它按容器内进程数算，且 /proc/stat 可能还是宿主的）的原因，生产监控应采集宿主侧或 cgroup 文件（如 /sys/fs/cgroup/cpu.stat）而不是容器内 top。cgroup 则把进程挂进控制组树，Docker 默认创建独立 cgroup，`--cpus`/`--memory` 对应 cpu.max 和 memory.max（cgroup v2 写法）；当容器吃满 CPU 时，cgroup 的 CPU 权重（cpu.weight）和配额（cpu.max）决定它能抢多少份额，另一个容器因此被限流。实践坑：老内核 cgroup v1 中 cpu/memory 分属不同 hierarchy，很多工具按 v1 路径硬编码会失效；`--privileged` 会关掉 namespace 大部分保护（如挂载宿主 /proc、访问设备），等同放宽隔离边界；调试时用 `docker run --pid=host` 或 `nsenter -t <pid> -m -p` 进宿主视角定位"容器内看不到的问题"。

**延伸考点：** cgroup v1 到 v2 演进解决了什么问题（统一层级、线程粒度控制）？为什么说容器逃逸拿到的是宿主内核权限？

---

### Q6. 容器网络有哪些模式？两个容器互相访问为什么不能直接 localhost？

**问题：** 你们用 docker-compose 起了 Nginx 和业务服务，Nginx 里配 upstream 时有人写了 `http://localhost:8080` 一直连接失败，换成容器名就通了。请解释容器网络模型，并说明端口映射和容器间通信的差别。

**期望加分项：**
- 能说清 bridge/host/none 三种模式的语义：bridge 是默认 veth pair + docker0 网桥 + iptables NAT，host 直接复用宿主网络栈（无独立 IP），none 只有 loopback
- 能讲透"localhost 不通"的根因：每个容器有独立 network namespace，localhost 只指容器自身；容器间通信用同一 bridge 网络的容器名/服务名做 DNS 解析
- 能给出端口映射原理：`-p 8080:80` 实际是 DNAT 规则（iptables 的 DOCKER 链），访问宿主机 8080 转发到容器 80
- 能结合排查实践：`docker network ls`、`docker network inspect`、`docker exec 容器 ip addr` 看网卡，`iptables -t nat -L -n` 看 DNAT 规则
- 主动考虑边界：不同 bridge 网络默认不通、overlay 网络（Swarm/K8s）跨宿主、host 模式下 -p 无效

**减分项：**
- 说不全三种网络模式，或以为 -p 映射就是容器网络本身
- 把"localhost 不通"解释成网络没起来，讲不清 namespace 隔离才是根因
- 不会用 docker network 相关命令做排查
- 无实践佐证，说不清 compose 里 service 名解析和链接（--link 已废弃）的差异

**解答：**
先讲隔离：每个容器独立 network namespace，有自己的 loopback 和 eth0，所以 localhost 永远只指向容器自己，跨容器访问必须走容器 IP 或服务名。默认 bridge 模式下，Docker 创建 docker0 网桥，每个容器的 veth 一端接容器 eth0、一端接网桥，容器间在同一个 bridge 网络内直接二层互通；`docker-compose` 会为每个项目建独立 bridge 网络并把服务名写进内嵌 DNS，所以 Nginx 里应写 `upstream backend { server app:8080; }`。访问外部则经 iptables NAT——`-p 8080:80` 生成 DNAT 规则（`iptables -t nat -L DOCKER -n` 可见），宿主机 8080 端口的包被改写目标地址转发进容器；对外访问则做 SNAT。host 模式容器直接用宿主网络栈，性能最好（无 NAT、无网桥转发）、但无独立 IP 且 -p 无效，适合对延迟敏感或需要绑定特权端口的场景；none 模式一般用于纯计算任务。实践坑：跨宿主容器默认不通，K8s/Swarm 用 overlay 网络 + VXLAN 解决；容器内 DNS 解析依赖 Docker 内嵌 DNS，容器改了 /etc/resolv.conf 或断网重启后解析可能失效；另外两个不同 compose 项目的网络互相隔离，需要 `docker network connect` 显式打通。

**延伸考点：** 容器内访问宿主服务时 `host.docker.internal` 在 Linux 上为什么默认没有？CNI 在 K8s 里扮演什么角色（相比 Docker 网络的差异）？

---

### Q7. volumes、bind mount、tmpfs 分别适合什么场景？容器数据怎么持久化和备份？

**问题：** 数据库容器（如 MySQL/Redis）跑在 Docker 里，重启后数据丢了，有人建议"把数据目录 COPY 进镜像"。你怎么纠正，并给出正确持久化方案和备份流程？

**期望加分项：**
- 能分清三种挂载：volume（Docker 管理、存在 /var/lib/docker/volumes/、跨容器共享、备份方便）、bind mount（挂宿主任意路径，适合配置文件/开发热更新）、tmpfs（内存盘、适合临时缓存、重启清空）
- 能给出正确做法：数据库数据用 named volume 或 bind mount 挂到容器数据目录（MySQL 挂 /var/lib/mysql），并说明"数据进镜像"为什么错（镜像层只读+COW、重建容器即丢）
- 能给出备份方案：`docker run --rm -v <volume>:/data -v $(pwd):/backup alpine tar czf /backup/mysql-$(date).tar.gz -C /data .`，以及数据库本身的逻辑备份（mysqldump）
- 能指出权限坑：bind mount 的 uid/gid 与容器内进程不一致导致写不进去，MySQL/Redis 常见 Permission denied
- 主动考虑边界：volume 默认类型为 local（单机），跨节点需要 NFS/云盘或 CSI；tmpfs 的数据不落盘、重启即失

**减分项：**
- 分不清 volume / bind mount / tmpfs 三者的管理方、存储位置和生命周期
- 认为重启容器数据必然保留，不知道容器删除（docker rm）会连容器层一起删
- 备份方案只会说"把目录拷出来"，没有容器化环境下的具体命令
- 忽略权限、挂载覆盖（mount 会盖住镜像里已有目录内容）等边界

**解答：**
先纠正认知：容器可写层是临时的，`docker rm` 一删全没，把数据 COPY 进镜像只会让镜像变脏且重启即丢。正解是挂载：数据库数据挂 volume 或 bind mount。volume 由 Docker 管理（/var/lib/docker/volumes/），`docker volume create mysql-data`，运行加 `-v mysql-data:/var/lib/mysql`，优点是可 `docker volume` 命令管理、权限默认正确；bind mount 直接挂宿主目录 `-v /data/mysql:/var/lib/mysql`，适合已有宿主管维习惯或开发热更新配置。临时数据用 tmpfs（`--tmpfs /tmp`），内存盘读写快、不落盘，适合缓存和测试。备份要分两层：物理备份用 `docker run --rm -v mysql-data:/data -v /backup:/backup alpine tar czf /backup/mysql-data-$(date +%F).tar.gz -C /data .`（注意 -C 进入数据目录再打包，避免路径嵌套）；逻辑备份用 `docker exec <容器> mysqldump --single-transaction -u root -p ... > backup.sql`，生产建议两者结合并做恢复演练。实践坑：bind mount 到容器内非空目录会把原目录内容"盖住"（不是合并），挂 /etc/nginx 前先确认镜像内已有配置；权限方面 MySQL 镜像默认以 mysql 用户跑，宿主目录属主不对会直接拒绝启动，先 `chown 999:999`（mysql uid）或检查 image 的 USER。

**延伸考点：** docker cp 和挂载卷之间是什么关系，什么时候该用 docker cp？K8s 里 PV/PVC 与 Docker volume 的对应关系是什么？

---

### Q8. 你们怎么保证拉下来的镜像"信得过"？镜像仓库和镜像签名是怎么落地的？

**问题：** 有供应商给了你一个自定义镜像，声称"绝对安全"，但你发现它基础层版本很老。你们团队自己的镜像也存在"某个仓库账号被攻破、往仓库推了恶意镜像"的担忧。你会如何设计镜像供应链安全方案？

**期望加分项：**
- 能说清镜像安全链路：来源可信（私有仓库 + 访问控制 + TLS）、内容可信（镜像扫描 + 签名）、运行可信（运行时校验 + 只拉已签名镜像）
- 能给出工具落地：Trivy/Grype 扫描漏洞并纳入 CI（如 critical/high 阻断）、cosign 对镜像签名（keyless 或私钥）、notary 做内容信任（Docker Content Trust，`export DOCKER_CONTENT_TRUST=1`）
- 能结合量化：依赖扫描阈值（如 critical ≥1 即失败）、漏洞修复的 base image 升级周期
- 能说明私有仓库选型：Harbor（自带扫描 + 签名 + 复制）、自建 Registry + 策略，以及镜像仓库的复制（异地容灾）
- 主动考虑边界：镜像签名保护的是"从推送到拉取"的完整性，防不了构建阶段本身被污染（需要构建环境安全 + SBOM）；旧镜像的漏洞不会因为新镜像安全而消失（需要策略防拉旧）

**减分项：**
- 只说"要扫描"说不出具体工具、阈值和 CI 落地方式
- 分不清扫描、签名、仓库访问控制各自防什么
- 认为"私有仓库就安全了"，忽略账号泄露、供应链投毒
- 无实践佐证，答不出 DOCKER_CONTENT_TRUST 或 cosign 的实际配置

**解答：**
按"供应链三段"设计：来源、内容、运行时。来源层：只允许从自建私有仓库拉取（Harbor 或 Registry + 独立认证），关闭对 Docker Hub 的匿名拉取，仓库开启 TLS 和镜像级 ACL；供应商镜像先导入隔离的"待审"项目，扫描通过才放行到正式项目。内容层：CI 里集成 Trivy 扫描（`trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 <image>`，critical/high 即构建失败），同时产 SBOM（syft）；镜像签名用 cosign（`cosign sign`）或 Docker Content Trust，拉取端 `export DOCKER_CONTENT_TRUST=1` 强制只拉带签名的镜像，K8s 场景配 cosign 的验证准入策略（Kyverno/PolicyController）在 admission 层拦截未签名或扫描不合格镜像。运行时层：容器运行账号最小化（后续专题），只运行已签名的 tag，禁用 latest 漂移。实践坑：签名防的是"仓库被攻破后镜像被替换"，但防不了"构建机被攻破把恶意代码编进镜像"——所以构建环境本身要加固、用 digest 而非 tag 部署（`image@sha256:...`）；扫描只认 CVE 库，0 day 和逻辑后门扫不出来，供应商镜像务必审 Dockerfile 和启动命令；另外镜像复制（Harbor replication）做异地容灾时记得同步校验签名与扫描结果。

**延伸考点：** cosign keyless 模式（基于 OIDC）和传统私钥签名各有什么利弊？如何让 K8s 在拉取时校验镜像签名（准入控制器）？

---

### Q9. 容器里 PID 1 为什么特殊？僵尸进程和信号转发是怎么回事，tini 解决了什么？

**问题：** 你们的 Java 服务容器收到 `docker stop` 后迟迟不退，最终被 10 秒强杀，日志丢失、连接没优雅关闭。查下来发现 entrypoint 直接用 shell 启动 Java。请解释根因和修复方案。

**期望加分项：**
- 能说清 PID 1 的两大职责：回收孤儿/僵尸进程（孤儿被 reparent 到 PID 1，需要 wait 回收）和转发信号（PID 1 默认不处理其他进程的 SIGTERM 语义）
- 能讲透"docker stop 发 SIGTERM 给 PID 1，shell 不转发给子进程 Java"导致 Java 收不到信号、超时被 SIGKILL 的完整链路
- 能给出修复：用 exec 启动（`exec java -jar ...`）让 Java 成为 PID 1，或引入 tini（`ENTRYPOINT ["/tini", "--"]` / docker 自带 `--init`）
- 能结合实践：Java 应用要配 shutdown hook（Runtime.addShutdownHook）+ Spring 优雅停机（server.shutdown=graceful），僵尸进程用 `docker exec` 里 ps 的 Z 状态识别
- 主动考虑边界：多进程容器（如 nginx+php-fpm、sidecar）的信号拓扑、supervisord/s6 的进程管理方案

**减分项：**
- 不知道 PID 1 的特殊性，以为所有容器都能自然处理 SIGTERM
- 只会说"加 tini"，讲不清 tini 解决的僵尸回收和信号转发两个具体问题
- 分不清 SIGTERM 和 SIGKILL 的语义及 docker stop 的默认超时（10s）
- 无实践佐证，答不出自己遇到的优雅停机故障

**解答：**
根因是 PID 1 语义。Linux 里 PID 1（init）有特殊职责：一是孤儿进程会被 reparent 到它，它必须对已退出的子进程调用 wait 回收，否则变成僵尸（ps 显示 Z）；二是向 PID 1 之外的进程发送的信号默认不生效（内核有保护），信号转发需要 init 自己实现。Docker 场景：`ENTRYPOINT ["/bin/sh", "-c", "java -jar app.jar"]` 时 PID 1 是 shell，`docker stop` 只向 PID 1 发 SIGTERM，shell 不会把它转发给 Java 子进程，Java 收不到信号自然不执行 shutdown hook；10 秒后 Docker 默认发 SIGKILL 强杀，连接来不及优雅关闭。修复两步：一是进程拓扑，要么 `exec java -jar app.jar` 让 Java 直接成为 PID 1（单进程容器首选，注意信号处理依赖 JVM 自身逻辑），要么用 tini 作为 PID 1（`ENTRYPOINT ["/tini", "--", "java", "-jar", "app.jar"]`，Docker 也内置 `--init`），tini 会回收僵尸并转发信号；二是应用侧配合，Java 写 shutdown hook、Spring Boot 开 `server.shutdown=graceful` 并设置合理超时（小于 K8s terminationGracePeriodSeconds），把当前请求处理完再退出。实践坑：`docker stop -t` 可自定义宽限时间，但比 K8s 的 preStop+graceful 机制粗；多进程容器（如 Nginx+PHP-FPM 共居）信号拓扑复杂，建议 tini + 明确的主进程托管策略，而不是让 shell 当 init。

**延伸考点：** K8s 里 terminationGracePeriodSeconds、preStop 和 SIGTERM 三者的执行顺序是什么？僵尸进程和孤儿进程在容器化场景下各由谁负责回收？

---

### Q10. `--cpus`、`--memory`、swap 怎么配？容器被 OOM 杀了怎么办？limit 和 request 的关系是什么？

**问题：** 线上一个容器内存涨到 2GB 就被 kill 掉，日志只有一行 `Killed`；另一个容器 CPU 被限流到 10% 但没人动过配置。请解释这两个现象的机制，并给出资源限额的正确配置方法。

**期望加分项：**
- 能说清资源限制的底层：`--memory` 走 cgroup memory.max，超限触发 OOM killer 按 oom_score 杀进程；`--cpus` 走 cpu.max 配额，超限被限流（throttle）而不是被杀
- 能讲清"Killed"排查路径：`dmesg` 看 OOM killer 记录、`docker inspect` 看 Memory/MemorySwap、应用侧监控 RSS 与 cgroup 内使用量（container_memory_working_set_bytes）而不是宿主 top
- 能区分 limit 与 request（K8s）：request 是调度依据（保证量），limit 是硬上限（超了就 OOM/限流），生产上两者不能拍脑袋，要基于压测给梯度
- 能指出 swap 的坑：容器默认可 swap（--memory-swap 未设时）导致"看着内存没超却被 OOM"，以及 swap 引发的性能抖动
- 主动考虑边界：--memory 与 --memory-swap 的关系（swap = 总量 - memory）、OOM 时是杀容器内进程还是整个容器、Java 的 -Xmx 与容器内存限制的关系

**减分项：**
- 分不清 OOM 击杀和 CPU 限流两种现象的本质区别
- 只给数字不会推导：不基于压测、不区分 request/limit 语义
- 排查 OOM 不看 dmesg / cgroup 文件，只会在宿主上 top
- 忽略 JVM 等运行时与容器限制的配合（-Xmx 设成物理机内存）

**解答：**
两个现象对应两种机制：内存超限→OOM killer。`--memory 2g` 即 cgroup memory.max=2G，容器内进程总内存（含 page cache）超过后内核按 oom_score 选进程杀掉，Java 常被杀的是整个 JVM 进程，宿主日志 `dmesg | tail` 里有 "Out of memory: Killed process ..." 记录，容器里可能只有 "Killed"。CPU 超限→限流。`--cpus 1` 即 cpu.max=100000 配额，超配时内核把进程放进 cgroup throttled 状态，进程没被杀只是变慢（`/sys/fs/cgroup/cpu.stat` 的 nr_throttled 增加）。所以配置要分两块：内存类先看应用真实占用（Java 用 `-Xmx` + `-XX:MaxRAMPercentage=70` 让 JVM 感知容器限制，别设成物理机内存；Go 程序注意 cgroup 内存感知），基于压测峰值 ×1.5 定 limit；CPU 类按服务类型给（CPU 密集给足配额、I/O 等待型可超卖）。K8s 里 request 决定调度（node 按 request 汇总分配），limit 是硬约束，经典坑是"request 设 1C、limit 设 8C"，节点超卖严重时 CPU 全被 throttle；内存 request 设小、limit 设大则可能"调度上去就 OOM"。实践坑：容器内读 /proc/meminfo 是宿主内存，监控必须用 cgroup 指标；swap 默认不设 --memory-swap 时容器可借用宿主 swap，容易掩盖内存泄漏，生产通常 --memory-swap 与 --memory 相等即禁用 swap。

**延伸考点：** oom_score_adj 是什么，怎么让"想保的进程"优先存活？K8s 的 Guaranteed/Burstable/BestEffort QoS 等级与 request/limit 的对应关系？

---

### Q11. 健康检查和优雅退出怎么配合？HEALTHCHECK 探的是什么、preStop 清的是什么？

**问题：** 你们服务滚动发布时经常"旧实例还在收流量、代码已经没了"，或者"新实例 ready 了但实际没起来"。请设计一套从容器健康检查到优雅停机的完整发布保护方案。

**期望加分项：**
- 能说清 HEALTHCHECK 的语义：Docker 的 `HEALTHCHECK --interval=10s --timeout=3s --start-period=40s --retries=3 CMD curl -f http://localhost:8080/actuator/health || exit 1`，start-period 用于容忍启动期
- 能指出"探活不等于就绪"：liveness 探针（活没活）和 readiness 探针（能不能接流量）要分开，启动慢的服务用 startupProbe 兜底（K8s）
- 能讲优雅退出链路：SIGTERM → 应用停止接受新连接 → 处理完存量请求 → 退出，K8s 的 preStop 钩子做主动摘流（调注册中心下线、排空 LB）
- 能结合实践：健康检查端点本身要探真实依赖（DB/Redis 挂了返回 503），但不能把下游抖动放大成自身不健康（级联摘除）
- 主动考虑边界：检查用 wget/curl 需要镜像里装命令（distroless 里没有），可改用 TCP 探测或扩展镜像；发布策略（滚动窗口、maxSurge/maxUnavailable）与健康检查联动

**减分项：**
- 只说"加个 healthcheck"，讲不清 liveness/readiness 的职责差异
- 优雅停机只想到信号，漏掉"摘流量"这一前置动作
- 探活端点只回 200，不探测真实依赖，或把健康检查当摆设
- 无实践佐证，讲不出自己发布流程中"旧进程还没断流量"的具体事故

**解答：**
完整的发布保护 = 启动就绪 + 存活探活 + 优雅摘流 + 优雅退出。第一步定义健康检查：K8s 里 readinessProbe 决定 Pod 是否进 Service Endpoint（就绪才收流量），livenessProbe 决定要不要重启（死了才重启），启动慢的服务加 startupProbe 防止误杀；Docker 原生场景用 `HEALTHCHECK` 指令，Dockerfile 里 `HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 CMD wget -q -O - http://localhost:8080/health || exit 1`，或 compose 的 healthcheck 配置，`docker ps` 会显示 starting/healthy/unhealthy 状态。第二步探活内容：health 端点应检查关键依赖（DB 连接、消息队列）但做超时兜底——依赖抖动若直接返回 503，会导致整个实例被摘掉、流量雪崩（级联摘除问题）。第三步优雅退出：K8s 先执行 preStop（调注册中心下线、sleep 让 LB 排空连接），再发 SIGTERM，应用收到后停止 accept 新请求、等存量请求完成（Spring 的 graceful shutdown + timeout 小于 terminationGracePeriodSeconds），超时才 SIGKILL。实践坑：健康检查用的 wget/curl 在 distroless/scratch 镜像里不存在，要么换 TCP/HTTP 探测要么用 debug 镜像；滚动发布要配好 maxUnavailable=0、maxSurge=1 保证新旧实例交替时无空窗，否则"摘旧建新"瞬间容量打折。

**延伸考点：** 探活端点依赖 DB 挂了返回 503，为什么会引发级联故障？readiness 探针和负载均衡摘除（如 Nginx upstream 的 passive health check）有什么区别？

---

### Q12. 容器安全加固怎么做？为什么不能默认 root 跑？capability、seccomp、AppArmor 各挡什么？

**问题：** 安全审计报告指出你们生产容器"全部以 root 运行、未限制内核能力、未开 seccomp"。请给出从 Dockerfile 到运行时的一整套加固方案，并说清每个手段在防什么。

**期望加分项：**
- 能说清"非 root 运行"的具体做法：Dockerfile 里 `RUN useradd -r app && USER app`（或直接继承镜像自带非 root 用户），运行时可 `--user 10001:10001`
- 能讲清 capability 收缩：默认容器有约 14 个 capability，`--cap-drop ALL --cap-add NET_BIND_SERVICE`（只需绑定低端口时），解释与 root 的关系（capability 是 root 权限的拆分）
- 能说清 seccomp 与 AppArmor 的分工：seccomp 拦 syscall（默认 profile 封禁 mount、reboot、ptrace 等危险调用），AppArmor 做文件/网络 MAC 访问控制
- 能结合量化与工具：`docker run --security-opt seccomp=default.json`、`capsh --print`、`docker inspect` 看 CapAdd/CapDrop、kube-sec 或 kubescape 扫描结果
- 主动考虑边界：非 root 跑带来的副作用（端口 <1024 要提权、写日志目录权限、Java 的 JVM 需用户可写目录）、audit2allow/aa-genprof 调 AppArmor 策略

**减分项：**
- 只会说"别用 root"，讲不出为什么 root 在容器里更危险（内核漏洞利用面）
- 分不清 capability、seccomp、AppArmor 三层防护的职责
- 把所有安全手段堆上去，但不考虑业务兼容性（如 NTP、ICMP 需要的 cap）
- 无实践佐证，答不出"加了限制后服务起不来"的排查经历

**解答：**
目标：把容器内的权限从"宿主 root 的弱化版"降到"普通用户 + 最小能力"。第一层身份：Dockerfile 加 `RUN useradd -r -u 10001 app && USER app`，运行再加 `--user`；原因不是"root 用户本身能做什么"，而是容器共享宿主内核，一旦有漏洞（如脏牛、容器逃逸类），root 身份让利用链直接拿到宿主权限，非 root 至少多一道门槛。第二层 capability：root 权限其实是一组可拆分的 capability，Docker 默认给容器 14 个左右；用 `--cap-drop ALL --cap-add NET_BIND_SERVICE` 只留必须的，比如 nginx 要绑 80 端口才需要 NET_BIND_SERVICE，普通 Web 应用可全删；注意很多应用需要 NET_RAW（ping/ICMP 相关）、SYS_TIME（NTP），删过头服务会异常，加回来要说明理由。第三层系统调用：seccomp 默认 profile 已封禁 mount、ptrace、reboot 等敏感 syscall，`--security-opt seccomp=unconfined` 是危险配置；AppArmor/SELinux 做 MAC 文件访问控制（K8s 里配合 podSecurityContext）。实践坑：非 root 后最常见的问题是"目录写不进去"——应用数据目录、/tmp、Java 的 class 目录都要显式 chown 给应用用户；改完安全配置务必跑一遍全量测试，因为有些框架启动时调用了被禁的 syscall（如早期 JVM 版本需要某些 syscall）。

**延伸考点：** seccomp 拦截的是哪个层级的操作（syscall 级），AppArmor 和 SELinux 在实现思路上有什么区别？K8s 的 PodSecurity / PSS 策略里 restricted 级别具体约束了什么？

---

### Q13. alpine、slim、distroless 怎么选？镜像"瘦"了之后踩过哪些坑？

**问题：** 你们团队推镜像瘦身，有人把所有镜像基础层换成 alpine，结果有服务启动报找不到 libcrypto.so.1.1，还有服务 DNS 解析异常。请给出镜像瘦身的完整决策树。

**期望加分项：**
- 能说清三类基础镜像的定位：alpine（musl + busybox，最小但 ABI 与 glibc 不同）、slim/debian（保留 glibc、体积适中）、distroless（无 shell/无包管理器，只含运行时与依赖，最安全但不可调试）
- 能解释 alpine 踩坑根因：musl 与 glibc 的 ABI 差异，预编译二进制（如 node-sass、oracle client、部分 SDK）要求 glibc，alpine 需要装 gcompat/编译重编
- 能给出决策树：纯 Go 静态编译 → scratch/distroless；Java → temurin JRE slim（别用 alpine 版 JRE，JVM 在 musl 下性能与兼容性都有坑）；Node/Python 带 native 模块 → debian slim；追求极致体积且无 native 依赖 → alpine
- 能结合量化：同一 Java 服务基础镜像从 400MB（完整 JDK）→ 150MB（jre slim）→ 40MB（jlink 裁剪），Go 从 300MB → 10MB
- 主动考虑边界：压缩层（squash/--squash）的利与弊（层复用损失）、体积 vs 构建时间 vs 调试便利的三角取舍

**减分项：**
- 无脑推 alpine，不知道 musl/glibc ABI 差异及其引发的连锁故障
- 把"镜像小"当唯一目标，忽略调试便利性（没有 shell 怎么排查）和构建时间
- 分不清镜像体积、拉取时间、磁盘占用三者的关系（压缩传输 vs 解压后大小）
- 无实践佐证，讲不出 alpine 换回来或 distroless 踩坑的真实案例

**解答：**
先纠正方向：瘦身的终点不是"体积最小"，而是"体积、兼容性、可调试性"的平衡，且优先保证运行时稳定性。决策树按语言/依赖走：Go 且 CGO_ENABLED=0 → distroless 或 scratch（只加 CA 证书和时区文件）；Java → 用 eclipse-temurin 的 jre slim（glibc）而非 alpine 版 JVM，因为 OpenJDK 官方对 musl 支持一般、部分特性（如某些 native 库、AsyncProfiler）在 musl 下不可用，真要压到 40MB 用 jlink 裁剪；Node → 有 node-gyp/native 模块用 node:slim（debian），纯 JS 项目可试 alpine；Python → 依赖里带 .so 预编译轮子的一律 debian slim，否则 uwsgi 这类都可能编译失败。alpine 踩坑的根因一句话：busybox 提供的工具不完整 + musl 与 glibc ABI 不同，`ldd` 找不到 libcrypto.so.1.1、DNS 解析依赖 nsswitch 行为差异都是典型症状。distroless 的代价是无 shell：进不去容器，排查要配 debug 镜像（distroless 有 -debug 变体带 busybox），或临时挂载工具。层压缩（`docker buildx build --squash`）能显著减小拉取体积，但会破坏层复用和增量拉取，CI 里慎用。实践坑：改基础镜像后 CI 全绿但线上偶发故障很常见，基础镜像变更要跑完整回归而不是只跑 smoke test。

**延伸考点：** 为什么说 alpine 的 DNS 解析行为（/etc/resolv.conf、nsswitch）与 glibc 镜像不同？UPX 压缩 Go 二进制为什么会引发不可预期的问题？

---

### Q14. 容器日志怎么打、怎么收？stdout/stderr 和日志驱动是怎么配合的？

**问题：** 你们容器里应用日志写进了 /var/log/app.log，排查问题时发现 docker logs 什么都看不到，日志还经常把容器磁盘写爆。请解释容器日志的正确姿势和采集架构。

**期望加分项：**
- 能说清第一性原理：12-factor 里日志走 stdout/stderr，Docker 的 json-file driver 把 stdout/stderr 重定向到宿主文件（/var/lib/docker/containers/<id>/*-json.log），docker logs 才能看到
- 能给出日志驱动对比：json-file（默认、单机查看方便、需要轮转）、journald（与 systemd 集成）、fluentd/syslog（转发到集中平台）、local/gelf 等，以及容器级 `--log-driver --log-opt max-size=10m max-file=3` 的轮转配置
- 能讲清采集架构：Filebeat/Fluent Bit 读宿主 json.log 文件 → Kafka → ELK/Loki，或应用侧直接推（结构化为 JSON，加 trace_id）
- 能指出日志丢失/阻塞的坑：驱动是阻塞式写，日志量暴涨可能拖慢容器（可加异步/丢日志策略）；多个容器日志抢占磁盘
- 主动考虑边界：K8s 里容器日志落宿主 /var/log/pods/，由节点级 DaemonSet 采集；结构化日志（JSON lines）与日志平台字段映射

**减分项：**
- 不知道"容器日志必须走 stdout/stderr"，看到 docker logs 为空还去找应用日志文件
- 分不清日志驱动的作用位置（是 Docker daemon 侧的重定向，不是应用侧的事）
- 只会说"加日志采集器"，说不清 json-file 的存储位置和轮转机制
- 无实践佐证，答不出"日志把容器磁盘写满"事故的完整链路

**解答：**
先立规矩：容器内日志一律写 stdout/stderr，别写文件。原理是 Docker daemon 启动容器时把 stdout/stderr 管道接到日志驱动——json-file 驱动逐行转成 JSON 追加到宿主 `/var/lib/docker/containers/<id>/<id>-json.log`，`docker logs` 就是从这读；写 /var/log/app.log 相当于把日志藏在可写层里，docker logs 看不见、容器一删日志没了、磁盘还容易写爆。驱动选型：单机调试用默认 json-file；规模化采集用 fluentd/gelf 或节点级采集器读 json.log（Filebeat/Fluent Bit 的 docker 输入模块天然支持）；与 systemd 生态集成用 journald。轮转必须配：`--log-opt max-size=10m --log-opt max-file=3`，否则 json.log 无上限增长（我们曾见 40GB 单文件把数据盘打满，docker logs 也拉不动）。采集链路建议：Filebeat/Fluent Bit 按容器名打标签 → 过滤/结构化（应用日志先打成 JSON lines，如 {"ts":...,"level":...,"trace_id":...}，比逐行正则解析稳得多）→ 发 Kafka → ELK/Loki。实践坑：json-file 是阻塞写，某个容器日志暴涨时可能拖住容器主进程（I/O 阻塞），高吞吐场景考虑给 logger 加非阻塞模式或干脆让应用自建缓冲区；另外 `docker logs` 全量拉取大日志文件很慢，排查时优先 `docker logs --since`、`--tail`，并配合日志平台全文检索。

**延伸考点：** K8s 下容器日志的存储路径和轮转是谁负责的（kubelet 的 containerLogMaxSize）？如何保证日志采集不丢不重（offset 记录与 at-least-once 语义）？

---

### Q15. Docker Compose 怎么搭一套像样的本地开发环境？服务依赖和热更新怎么处理？

**问题：** 新同学入职第一天，跑项目本地环境要手动装 MySQL、Redis、Nginx 再改一堆配置，折腾一天。请你用 docker-compose 搭一套"一条命令起全部依赖"的开发环境，并说清编排、依赖顺序和开发体验优化。

**期望加分项：**
- 能给出 compose 文件骨架：services 定义各服务（image、ports、volumes、environment、networks、depends_on），`docker compose up -d` 一键拉起
- 能处理依赖顺序：depends_on 只保证启动顺序不保证就绪，配合 healthcheck + condition: service_healthy，或让应用侧做重试
- 能说清开发体验优化：bind mount 挂代码目录实现热更新（如 node --watch / 后端 dev 模式）、.env 文件管理环境变量、profiles 区分本地/测试依赖
- 能讲配置差异：端口冲突处理（不同项目用不同宿主端口）、volumes 持久化本地数据库（不每次重建）
- 主动考虑边界：compose 与 K8s 的关系（本地编排 vs 生产调度，K8s 不直接读 compose，可用 kompose/DevContainer 或 k8s 原生清单）、Mac/Windows 的 bind mount 性能问题（文件监听失效）

**减分项：**
- 只会 `docker compose up`，写不出依赖健康检查、环境变量、卷挂载
- 以为 depends_on 就保证"服务可用"，不知道它只管启动顺序
- 本地环境与生产环境割裂（compose 里跑的东西和生产不一样，镜像不一致）
- 无实践佐证，答不出热更新/端口冲突/数据持久化这些日常痛点

**解答：**
先给标准骨架：`services: mysql: image: mysql:8.4, environment 设 root 密码和库名, volumes 挂 mysql-data 与初始化 SQL, ports 暴露 3306, healthcheck: test: ["CMD", "mysqladmin", "ping"]`；`redis: image: redis:7-alpine, command: redis-server --requirepass xxx, ports 6379`；业务服务 `build: .`, `ports: "8080:8080"`, `environment` 里 DB 地址写服务名 `mysql:3306`（compose 内建 DNS），`volumes: - .:/app` 挂代码，`depends_on: mysql: condition: service_healthy`。三个关键细节：一是就绪问题——depends_on 只是容器启动先后，MySQL 容器起了但还没就绪，应用连库必然失败，所以要么 healthcheck + service_healthy，要么应用连接重试（更通用）；二是热更新——本地把代码 bind mount 进容器并启动 dev 模式（`npm run dev` 或 Java devtools），改代码即生效；三是环境一致性——用 `.env` 管端口/密码/版本，compose 里固定镜像 tag 与生产同源，避免"本地能用线上炸"。实践坑：Mac/Windows 的 bind mount 走文件共享层，监听大目录（node_modules）时 watch 事件常丢，解决是排除 node_modules 或把依赖装进镜像；多个项目都用 3306 会端口冲突，宿主端口按项目前缀分配；compose 默认网络对项目隔离，跨项目要 `external: true` 网络。

**延伸考点：** compose 的 `depends_on` 有哪些条件形式（service_started/service_healthy/service_completed_successfully）？`docker compose watch` 和 bind mount 热更新的区别？

---

### Q16. 构建上下文和缓存失效是怎么回事？为什么改一行依赖重装十分钟？
**问题：** 一个前端项目每次 CI 构建都要 npm install 十分钟，改了 package.json 之后更慢；本地构建却很快。Docker 构建缓存到底怎么命中/失效的？怎么设计 Dockerfile 让依赖缓存最大化？

**期望加分项：**
- 能说清缓存机制：Docker 按指令逐层缓存，只有"该层指令+上下文输入"变化才失效，且失效后的层全部重建
- 能讲 COPY 的顺序策略：先把 package.json/lock 单独 COPY 再装依赖，最后 COPY 源码，让代码改动不触发依赖重装
- 知道上下文大小对构建的影响：.dockerignore 排除 node_modules/.git 等，减少发送到 daemon 的上下文
- 知道 BuildKit 的改进：并行层、远程缓存（registry/github action cache）、`--mount=type=cache` 挂 npm/maven 缓存目录
- 能谈 CI 与本地构建差异：CI 无缓存冷启动 vs 本地增量缓存，需要把缓存落到 CI 的 layer cache 或远端
**减分项：**
- 以为 Dockerfile 指令执行顺序不影响缓存
- 不知道 lock 文件单独 COPY 的价值，package.json 和源码一起 COPY
- 上下文塞满 node_modules 还抱怨构建慢
- 不知道 BuildKit / cache mount 这类优化手段
**解答：**
Docker 构建按指令逐层缓存：每条指令产出一个只读层，缓存命中条件是"该条指令本身没变 + 依赖的上一层缓存命中 + COPY 的上下文文件内容没变"。所以一旦某层失效，它后面的层全部重建——这是理解缓存的核心。经典优化是把"变化频率从低到高"排列指令：先 COPY 包管理锁文件（package-lock.json / pnpm-lock.yaml / pom.xml）→ 运行安装命令 → 再 COPY 源码。这样改一行业务代码，只有最后 COPY 源码那层重建，依赖层直接命中缓存。第二个重点是控制构建上下文：docker build 会把整个上下文目录打包发给 daemon，node_modules 动辄几百 MB，必须用 .dockerignore 排除 node_modules、.git、dist 等，否则光传输上下文就比构建还慢。第三是 BuildKit（Docker 20.10+ 默认）：支持并行构建、`--mount=type=cache,target=/root/.npm` 把 npm/pnpm 缓存目录挂进构建（容器删除后缓存仍在宿主机或远程），以及远程缓存（推送到 registry 或 GitHub Actions cache）让 CI 也能复用上一次构建结果——CI 上没缓存时"冷启动装依赖十分钟"很正常，解法是 layer cache 或 cache mount。实践坑：本地快 CI 慢，通常是本地有 daemon 缓存而 CI 每次新机器；COPY 顺序写反了排查很费劲——先看构建日志里哪一层开始 `CACHED` 消失。

**延伸考点：** BuildKit 的 `--mount=type=cache` 与 layer cache 的区别和适用场景？多阶段构建中 COPY --from 跨阶段的缓存如何命中？

---

### Q17. 容器内时区、locale 和依赖一致性怎么处理？为什么容器里日期差 8 小时？
**问题：** 应用容器化后日志里的时间戳比本地少了 8 小时，中文乱码、某些库依赖 locale 报错。容器内的时区、locale、系统依赖这些"看不见的配置"怎么保证和测试/生产一致？

**期望加分项：**
- 能说清根因：官方镜像默认 UTC 时区（如 alpine、debian 默认 UTC），需要显式设置时区
- 能给出时区方案：Dockerfile 里 `ENV TZ=Asia/Shanghai` + 安装 tzdata，或运行时挂载 /etc/localtime；并说明 alpine（musl）与 debian（glibc）的差异
- 能讲 locale 问题：glibc 需要安装 locale 并设置 LANG/LC_ALL，alpine 默认无完整 locale，中文/Unicode 处理受限
- 能谈构建与运行一致性：构建阶段和运行阶段基础镜像版本、依赖来源要固定（锁文件/固定 tag），避免"本地装好了生产没有"
- 知道常见连带坑：Cron 定时任务时区、日志采集解析时区、JDK 应用读系统时区 vs JVM 时区
**减分项：**
- 不知道容器默认 UTC，遇到时间差 8 小时才懵
- 只在运行时 set 时区、不改镜像，换节点/重建又丢
- 不区分 alpine 和 debian 在 locale/glibc 上的差异
- 依赖一致性上依赖"恰好能跑"，不用锁文件固定版本
**解答：**
容器默认继承基础镜像的时区，官方镜像基本都是 UTC，所以应用看到的时间比北京时间少 8 小时。正确做法是在 Dockerfile 里显式声明，而不是运行时手动改：Debian 系 `RUN apt-get install -y tzdata && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone`，再 `ENV TZ=Asia/Shanghai`；Alpine 系 `RUN apk add --no-cache tzdata && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`。注意两点：一是时区最好固化进镜像（构建期设置），运行时挂载 /etc/localtime 依赖宿主机且易漂移，不适合多节点；二是不要只设 TZ 环境变量——很多语言运行时不读 TZ 而是读 /etc/localtime，两者都要设置。locale 问题集中在 glibc 系：Java/Python 的国际化、某些库（ICU）依赖 locale，需要 `RUN locale-gen zh_CN.UTF-8` 或 `ENV LANG=C.UTF-8`；alpine 基于 musl，对 glibc 库兼容性差（原生扩展可能编译不过），有 glibc 依赖就选 debian 系镜像。依赖一致性是另一层：构建阶段和运行阶段都要用锁文件（package-lock、go.sum）和固定 tag，基础镜像 tag 用具体版本而非 latest，否则"本地能跑、生产崩"。实践中还常遇到：JDK 应用时区要 `-Duser.timezone=Asia/Shanghai` 与系统时区一致，定时任务（cron 在容器内默认 UTC）和日志采集（采集器按宿主时区解析）两处都要对齐，否则日志时间线全乱。

**延伸考点：** alpine 与 debian 镜像在 glibc 兼容上的根本差异，什么场景必须换 debian-slim？容器内时间同步（libc 时间缓存、clock 精度）在分布式系统里有什么坑？

---

### Q18. 容器化部署后连接池句柄泄漏、文件句柄耗尽、DNS 缓存过期，这些坑怎么排查？
**问题：** 服务容器化上 K8s 后，运行几天出现 `Too many open files`、数据库连接数暴涨、偶尔请求报域名解析失败。这些"容器特有的坑"分别怎么定位和根治？

**期望加分项：**
- 能分门别类列出容器化常见坑：文件句柄（ulimit/host 共享）、连接池（进程不退出导致句柄累积）、DNS 缓存（glibc 不缓存 vs 应用内缓存）、端口冲突（host 网络）
- 文件句柄：容器与宿主机共享内核，ulimit 受宿主限制，检查 `cat /proc/1/limits`，确认进程是否反复创建 fd（连接池、日志文件、临时文件）没释放
- 连接池：应用连 MySQL/Redis 的连接要复用（连接池），检查 ESTABLISHED 连接数与服务实例数×池上限是否吻合，句柄泄漏往往伴随连接数只增不减
- DNS：glibc 不缓存 DNS 结果，但 JVM/Go/Node 的 DNS 缓存策略不同，K8s 里 Pod IP 变动导致缓存到失效地址；排查 `getent hosts` / nslookup 与应用实际解析
- 能讲排查命令链：`lsof -p`、`ss -tn`、`cat /proc/PID/fd`、`ss -lntp` 定位是哪种资源耗尽
- 主动给预防手段：连接池上限+超时、fd 上限监控、DNS 缓存 TTL 配置、定期压测暴露
**减分项：**
- 只报"连接数太高"，拿不出定位命令
- 不知道容器与宿主机共享内核意味着 ulimit/句柄受宿主限制
- 不区分连接池连接与请求连接，误判为"应用连接泄漏"
- 不知道不同运行时（JVM/Go/Node）DNS 缓存行为差异
**解答：**
这类问题特征是"容器能跑，跑几天出问题"，因为都是慢性的资源累积。逐类拆：1) 文件句柄耗尽（Too many open files）——容器与宿主机共享内核，容器内 ulimit 通常继承宿主或 K8s 配置，排查先 `cat /proc/1/limits` 看上限，再看 `ls /proc/<pid>/fd | wc -l` 与 `lsof -p <pid>` 统计是哪个资源（连接/日志/文件）占用；根因常是连接池没设上限、每次请求新建连接不关闭、或日志文件句柄没轮转。2) 数据库连接数暴涨——先 `ss -tn | grep :3306 | wc -l` 看 ESTABLISHED，再对比"实例数 × 每实例池上限 × 复用系数"算理论值；连接只增不减通常是连接池泄漏（异常分支没归还连接）或健康检查/慢查询把池占满；容器环境下还要注意每个副本都会建池，副本数×池大小才是总量。3) DNS 解析失败/过期——glibc 本身不缓存 DNS，但 JVM 默认缓存（networkaddress.cache.ttl 默认永久缓存！）、Go 用纯 Go resolver 按 TTL 缓存、Node 走系统解析；K8s 里 Pod IP 每次重建都会变，应用如果缓存了旧 IP 就请求失败；排查用 `getent hosts service.ns.svc` 看集群内解析、`nslookup` 看外部域名，再对照应用缓存配置。预防三件套：连接池设上限+空闲超时+异常归还（finally/context）、fd 与连接数纳入监控告警（配合 Q：监控模块）、DNS 缓存 TTL 显式配置（JVM 设 `-Dsun.net.inetaddr.ttl=30`）。这些都是容器化后"看不见宿主机的便利"带来的典型债，排查思路永远是"先确认是哪种资源、再看谁在累积、最后加防护"。

**延伸考点：** K8s 里 Pod 重建后 IP 变化，除了 DNS 缓存还有哪些服务发现方式？Go 的纯 Go resolver 与 cgo resolver 在容器里行为差异？

---

### Q19. 多架构镜像怎么构建和发布？为什么在 ARM Mac 上构建的镜像到 x86 服务器跑不起来？
**问题：** 团队有人用 Apple Silicon 开发机 build 了镜像推到仓库，线上 x86 服务器拉下来直接 `exec format error`。多架构镜像（multi-arch）是怎么工作的？CI 里怎么一次性产出 amd64 + arm64 镜像？

**期望加分项：**
- 能说清根因：镜像里是编译好的二进制/机器码，不同 CPU 架构不兼容，exec format error 就是架构不匹配
- 知道多架构镜像机制：manifest list / OCI index，一个 tag 指向多个平台的 manifest，客户端按自身架构拉对应层
- 会用 buildx：`docker buildx build --platform linux/amd64,linux/arm64 --push`，配合 QEMU 模拟交叉构建，或用各平台原生构建
- 知道构建约束：基础镜像要选多架构的（alpine/debian 官方支持）、依赖（原生扩展）要按平台分别编译、go build 交叉编译 CGO_ENABLED=0 或按平台矩阵
- 能谈运行时坑：pull 时按平台选层，但要确认运行环境（K8s 节点架构）、本地开发机架构与部署架构不一致时的验证
**减分项：**
- 不知道 exec format error 的含义，绕圈子查环境变量
- 以为一个镜像能跑所有架构
- 不会用 buildx，只会 docker build（单平台）
- 交叉构建时原生依赖（node-gyp、cgo）直接编译失败不知道怎么处理
**解答：**
`exec format error` 的含义是"二进制格式与 CPU 不兼容"——镜像里是编译产物，ARM 机器编译的二进制在 x86 上直接报这个错。多架构镜像的正确姿势是 manifest list（OCI index）：一个 tag（如 `nginx:1.27`）在仓库里是一个清单，下面挂着 amd64/arm64 各自的镜像（每份含对应架构的层），`docker pull` 时客户端根据自身 CPU 架构自动选择对应那份层，所以同一个 tag 在 Mac（ARM）和服务器（x86）上拉到的"内容"不同但都能跑。构建侧用 BuildKit 的 buildx：`docker buildx build --platform linux/amd64,linux/arm64 -t reg/app:v1 --push .`，buildx 用 QEMU 做模拟执行（跨架构构建），或配置多台原生构建机（buildkit 远程 driver）避免模拟构建慢且偶发不准。关键在依赖：基础镜像选官方多架构的（alpine/debian/distroless 都支持），但原生扩展必须按平台分别编译——node-gyp 编译的 .node、cgo 的 Go 程序、Python 的 C 扩展，都要在对应架构下构建（交叉编译要关 cgo：`CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build`）。实践坑：CI 里构建机是 x86，要产出 arm64 就得靠 QEMU 模拟（慢）或远端 arm 构建机；Java/JVM 应用是纯字节码跨架构没问题，Go/Rust/C 这类编译型语言必须在 Dockerfile 里做成多阶段+按平台参数化（`ARG TARGETARCH`），不要用 `go build` 默认架构。验证手段：`docker buildx imagetools inspect reg/app:v1` 看平台列表，`docker run --platform linux/amd64 ...` 在本地强制模拟验证。生产上还要确认 K8s 节点全是同一架构，否则调度到异构节点会出现部分 Pod 起不来。

**延伸考点：** QEMU 模拟构建与原生构建的差异，什么情况下必须用原生构建机？manifest list 的 digest 和 tag 如何做到"发布后回滚到上一个平台组合"？

---

### Q20. 存量老应用容器化改造，怎么定改造边界和迁移节奏？
**问题：** 一个跑在裸机/虚拟机上的老应用，有定时任务、读写本地磁盘日志、依赖某个系统库，直接塞进容器就崩。容器化改造怎么拆解？哪些应用"不适合容器化"要说明原因？

**期望加分项：**
- 能先做应用画像：无状态优先、有状态后置；先评估依赖（系统库/本地磁盘/固定 IP/多实例）
- 给出改造步骤：配置外置（环境变量/ConfigMap）→ 日志输出到 stdout 或挂卷 → 依赖装进镜像 → 健康检查补全 → 探针/优雅退出 → 分批灰度
- 能识别不适合容器化的场景：强绑定固定 IP 的长连接服务、需要极低延迟共享内存/高性能计算、依赖特定宿主设备/内核模块、状态全在本地磁盘且无外部存储
- 能讲迁移节奏：影子模式/并行运行（容器与 VM 同时跑对流量）→ 灰度放量 → 全量切换 → 回滚预案
- 主动考虑数据：本地磁盘日志/文件上传要迁到外部存储或挂卷，定时任务与 K8s CronJob 的映射
- 有量化收益认知：容器化的收益（发布速度、弹性、环境一致）vs 改造成本（存储改造、网络模型、运维体系变化）的权衡
**减分项：**
- 一股脑全容器化，不考虑有状态和数据问题
- 不知道本地磁盘、固定 IP、多实例状态是改造硬骨头
- 没有灰度迁移和回滚预案，一把梭
- 答不出"什么场景不该容器化"，只会说"容器化是趋势"
**解答：**
容器化改造的本质是"把应用从宿主环境假设中解耦"，所以第一步做应用画像：它依赖宿主的什么？——本地磁盘文件（日志/上传/临时文件）、固定 IP、系统库（/usr/lib、内核模块）、多实例共享状态。凡是被这些绑架的，先改造再容器化。标准改造路线：1) 配置外置——硬编码配置改环境变量/ConfigMap，镜像内不留环境差异；2) 日志改造——进程日志写 stdout/stderr 交给采集器，或挂卷持久化；3) 依赖入镜像——系统库、JDK/Node 版本锁进镜像（Dockerfile 安装），不再依赖宿主机；4) 可管理性补全——健康检查接口、优雅退出（SIGTERM 处理）、资源声明（request/limit）；5) 数据迁出——本地磁盘上传/日志迁到对象存储或持久卷（PV），定时任务迁到 CronJob。迁移节奏用"影子+灰度"：先在容器里跑一个副本与 VM 并行（对比指标、不接流量），再灰度 5%→50%→100%，全程可回滚。明确"不适合容器化"的场景要说清楚：强依赖固定 IP 且长连接注册（如老旧 RPC 服务）、需要访问宿主内核模块/设备（DPDK、特定网卡）、超低延迟共享内存通信、状态全在本地且无法迁移的数据库/缓存——这些容器化收益低、成本高，不值得硬改。最后要谈收益量化：容器化换来的是发布速度（分钟级）、弹性（HPA）、环境一致性（镜像即环境），但如果应用 1 年发 3 次版、无弹性需求，改造投入产出不成正比——判断标准是"发布痛点是否真实存在"，而不是"别人都在容器化"。

**延伸考点：** 有状态应用在 K8s 上可以用 StatefulSet，什么条件下值得把数据库/缓存也迁进去？容器化后定时任务与 K8s CronJob 的时钟、幂等、失败重试差异怎么处理？

---

### Q21. 生产容器频繁 OOM/被重启，从现象到根因的完整排查流程（资源限制、JVM 参数、内存泄漏）

**问题：** 生产环境一个 Java 服务容器每隔几小时就被重启，监控显示内存持续上涨、然后 OOMKilled，重启后循环复现。但开发说"本地跑没问题"，运维说"内存限制太小"。从现象到根因，你会怎么完整排查？资源限制、JVM 参数、内存泄漏三个层面怎么区分和定位？

**期望加分项：**
- 有完整排查链路：先确认"谁杀的"（容器超 limit 被 cgroup 杀 vs 节点内存耗尽牵连）→ 拉证据（docker inspect / cgroup 事件 / dmesg / 监控曲线）→ 分层定位（cgroup 限制、JVM 堆配置、堆外内存、代码泄漏）
- 能区分两类 OOM：容器超 limit（`docker inspect` 的 State.OOMKilled、`/sys/fs/cgroup/memory.events` 的 oom_kill 计数）vs 节点内存耗尽（dmesg 显示被杀的可能是别的进程/容器，要治理混部而不是只调这个容器）
- 懂 JVM 与容器的关系：JDK8u191 前不感知 cgroup，按宿主机内存算默认堆 → 容器 limit 4G、JVM 堆按宿主机 32G 配 → 必然 OOM；正确用 `-XX:MaxRAMPercentage=75` 或显式 -Xmx，且给堆外（Metaspace、线程栈、DirectBuffer）留预算
- 会查内存泄漏：堆内（jmap dump + MAT 找大对象与引用链）、堆外（Native Memory Tracking、DirectBuffer 未 release、线程数）、代码层（无限缓存、连接池/流未关闭）
- 有量化证据：内存曲线形态（稳步上涨=泄漏 vs 锯齿=正常回收）、GC 日志（Full GC 频率与耗时）
- 知道 OOMKilled 与 restart 策略拉起重启形成循环，处理要"根因修复"而非"调大 limit 掩盖"

**减分项：**
- 分不清"容器超限被杀"还是"节点内存耗尽"，方向直接带偏
- 只让"调大 limit"，不查根因，把 OOM 当常态运维
- 不知道 JVM 在容器里按宿主机内存算堆的经典坑
- 只查堆内不查堆外（DirectBuffer、Metaspace、线程），排查不完整
- 没有曲线与 GC 日志做量化，全程靠猜

**解答：**

完整排查分四层，先分层再定位。第一层确认"谁杀的"：`docker inspect <id>` 看 State.OOMKilled 与 RestartCount，`cat /sys/fs/cgroup/memory.events` 看 oom_kill 计数是否在增长（容器超 limit 被 cgroup 杀）；同时节点 `dmesg | grep -i "Out of memory"`——如果被杀的是别的容器或系统进程，说明是节点内存耗尽，要治理混部与 request/limit 分配，而不是只调这一个容器。第二层确认"增长形态"：拉内存监控曲线（container_memory_working_set），直线不回落的上涨 = 泄漏；锯齿状（涨-回收-降）但整体上行 = 慢泄漏；周期性被杀 = 与流量/定时任务相关。再配合 GC 日志：Full GC 频繁且耗时长，优先怀疑堆配置问题而不是代码。第三层查 JVM 配置——这是 Java 容器 OOM 最高频的根因：JDK8u191 之前不感知 cgroup，JVM 按宿主机内存算默认堆（宿主机 32G → 默认堆约 8G），容器 limit 只有 4G，启动即崩溃或缓慢 OOM；修复是 `-XX:MaxRAMPercentage=75`（留 25% 给堆外）或显式 -Xmx，同时为 Metaspace、线程栈、DirectBuffer 预留额度。第四层查代码泄漏：堆内用 jmap dump + MAT 找 dominator（无限增长的缓存 Map、未关闭的连接/流）；堆外用 Native Memory Tracking（`-XX:NativeMemoryTracking=summary` + `jcmd VM.native_memory`）定位 DirectBuffer/线程/CodeCache 的增长，排查 netty 未 release、线程池无限创建、ThreadLocal 膨胀。实践坑：一看到 OOMKilled 就调大 limit 是治标不治本——堆配置错误调大 limit 只是掩盖，代码泄漏调多大都没用；容器内对 8G+ 堆做 dump 会卡死应用，要在低峰期或走独立工具容器抓取。排查产出应该是一句话根因（如"JDK 未感知 cgroup，默认堆超 limit"）+ 修复 + 观察 48 小时曲线回归验证。

**延伸考点：** 容器被 OOMKilled 后 restart 策略拉起进程，怎么避免"拉起后立刻再被 OOM"（启动限流、健康检查配合）？堆外内存定位中 NMT 与 pmap 各自能给出什么证据？

---

### Q22. 镜像仓库疑似被污染/供应链攻击事件，处置步骤与长期预防

**问题：** 安全团队发现私有镜像仓库里某个镜像的 layer 与官方版本不一致，镜像里还有可疑可执行文件，怀疑仓库被入侵或构建过程被投毒。作为平台负责人，你会怎么处置？后续怎么长期预防供应链污染？

**期望加分项：**
- 处置有顺序：先隔离（仓库只读、封禁受影响镜像、暂停相关发布流水线）→ 取证（保留现场、导出镜像与审计日志、分析异常 layer）→ 影响面评估（反查拉取/部署记录，确定哪些节点和环境中招）→ 清除重建（轮换全部凭证、重建镜像并验签）→ 复盘改进
- 能讲清供应链污染的入口：不可信基础镜像源、构建依赖被投毒（npm/pip/go mod）、CI 环境被控（runner、密钥、缓存污染）、仓库自身（弱口令、API 暴露、同名 tag 覆盖）
- 长期预防成体系：基础镜像只从白名单源拉取并锁 digest（`FROM ...@sha256:...`）、镜像签名与拉取验签（Cosign/Notation + 准入策略）、扫描进 CI 门禁（Trivy/Clair：漏洞/恶意包/敏感信息）、仓库加固（RBAC、机器人账号短生命周期、不可变 tag、审计日志、网络隔离）
- 有纵深与可追溯：生成并存储 SBOM（syft）随镜像发布，出事后能快速回答"这个镜像里有什么"；依赖 lockfile + 哈希校验；CI 构建环境最小权限与隔离
- 处置后验证：确认入口已堵（重放攻击测试）、组织月度供应链演练验证发现与阻断能力

**减分项：**
- 一上来就删仓库重装，丢了取证现场，根因永远查不清
- 只处置不预防，或只预防不处置，缺完整闭环
- 不知道镜像签名验签、锁 digest 这些关键机制
- 只盯仓库本身，忽略 CI 环境与依赖源也是投毒入口
- 不做影响面评估，不知道哪些环境已经拉过被污染镜像

**解答：**

供应链事件的处置原则是"先止血、再取证、后重建、末预防"。第一步止血与隔离：立即把仓库切为只读（或摘出访问），在发布侧封禁受影响镜像的 tag（不可变 tag + 拒绝拉取清单），暂停相关服务的发布流水线——同时保留现场：不要急着删镜像删日志，先 `docker image save` 备份受影响镜像、导出仓库审计日志与访问记录。第二步取证与影响面评估：用 digest 与官方/上一版本比对（`docker history --no-trunc` 看层构成，解包异常 layer 分析文件），反查拉取记录与部署记录，确定受影响的环境与节点范围——这一步决定后续清理的边界。第三步清除与重建：清理可疑镜像、轮换仓库全部凭证（密码、token、机器人账号）、从官方可信源重新构建并验签，确认 CI/构建环境未被持续控制后再恢复发布。第四步长期预防成体系：① 可信源头——基础镜像只从官方/白名单 registry 拉取并在 Dockerfile 锁 digest（`FROM registry/base@sha256:...`），上游换包时构建即失败，从机制上杜绝"不自知";② 签名验签——Cosign/Notation 给镜像签名，拉取端策略验签（K8s 可用 Sigstore policy-controller 做准入控制），签名私钥放 KMS/HSM；③ 扫描与准入——CI 里加 Trivy/Clair（漏洞 + 恶意包 + 敏感信息扫描）作为门禁，仓库开不可变 tag 防同名覆盖；④ 依赖与构建环境——lockfile 锁版本并校验哈希（`npm ci`/`go mod verify`），CI runner 最小权限与隔离，共享缓存定期清理；⑤ 生成 SBOM（syft）随镜像存储，出事后能回答"镜像里都有什么、该查哪里"。最后是常态化验证：月度供应链演练（模拟一次投毒，验证能否及时发现与阻断）、季度审计日志评审。最贵的教训通常是：没有验签和 digest 锁定，事后连"哪些镜像被污染"都无法确定——安全投入的价值是把"未知"变成"已知"。

**延伸考点：** 仓库只读期间正在灰度发布的服务怎么处理（继续放量还是暂停，切换通道怎么兜底）？基础镜像锁定 digest 后，上游安全更新怎么跟进（Renovate/Dependabot 自动升级 + 变更验证）？

---

### Q23. 遗留单体应用容器化上线的完整改造方案：依赖、数据、配置、探针、灰度替换

**问题：** 一个跑了 8 年的单体 Java 应用（Tomcat 部署、连 MySQL 与 Redis、写本地磁盘文件、配置散落在 properties 和运维脚本里、宿主机有 crontab 定时任务），要完整容器化并替换现有 VM。从依赖、数据、配置、探针到灰度替换，你会给出怎样的完整改造方案？

**期望加分项：**
- 能列出完整改造清单并逐项给方案：依赖（JDK/系统库锁进镜像、多阶段构建）、数据（本地磁盘迁外部存储/挂卷、MySQL/Redis 留在 VM 外部化）、配置（properties 外置环境变量、密钥走 Secret）、探针（健康检查接口补全、readiness/liveness 分离、preStop 优雅退出）、灰度替换（影子部署 → 灰度 → 全量，可回滚）
- 配置改造具体：硬编码 properties 改环境变量注入，环境差异用配置文件模板渲染，密钥进 Secret/配置中心不进镜像
- 灰度替换有节奏：先影子部署（容器与 VM 并行跑、不接用户流量、对比日志与指标）→ 灰度 10%→50%→100% → 全量后 VM 保留数周兜底回滚
- 能识别单体改造独有难点：session 进程内状态（灰度切流掉登录）、定时任务不能双跑、连接池规模随副本数变化要重算、日志/文件存储位置变化
- 探针设计具体：/healthz 带依赖状态检查，readiness 管流量准入、liveness 只探存活性，preStop + terminationGracePeriodSeconds 优雅停机，JVM 容器内资源感知
- 有验收与回滚：新旧错误率/延迟对比 VM 基线（10% 内合格）、一键切回 VM 预案、写库路径不变保证数据一致性

**减分项：**
- 只讲"写个 Dockerfile"，不讲数据、配置、探针、灰度这些真正的改造难点
- 定时任务不处理，容器与 VM 双跑导致任务重复执行
- session/进程内状态不处理，灰度切流后用户掉登录
- 没有影子验证或回滚预案直接一把梭
- 配置不外置，把环境差异焊死在镜像里

**解答：**

单体容器化按"依赖→数据→配置→探针→灰度替换"五步推进，每步独立验证。第一步依赖：把 JDK 版本、系统库（glibc、字体、加密库）全部锁进镜像，多阶段构建（builder 编译打包 → 运行时镜像只放 JDK + 产物）；JDK 必须 8u191+ 并加 `-XX:MaxRAMPercentage` 让 JVM 感知容器内存，否则按宿主机算堆直接 OOMKilled。第二步数据：凡是"写本地磁盘"的全部外置——日志改 stdout 交给采集器、上传文件迁对象存储或挂持久卷、临时文件用 emptyDir 并明确"重启即失"；MySQL/Redis 保持外部托管（VM/云 RDS）不进容器，降低有状态改造风险。第三步配置：扫描所有 properties，区分"环境差异配置"（外置环境变量，部署时注入）与"密钥"（走 Secret/配置中心），删掉运维脚本里的硬编码；镜像内只留"任何环境都一样"的默认值。第四步探针：补真正的健康检查接口（反映 DB/Redis 依赖是否可用，而不是只回 200）；readiness 管流量准入（依赖故障时摘流量但不重启）、liveness 只探进程存活性（别把依赖故障配成 liveness 失败导致频繁重启）、preStop 钩子做优雅排空（下线服务发现 + 停止接新请求 + 等存量请求处理完）、terminationGracePeriodSeconds 给足排空时间。第五步灰度替换：先影子部署——容器与 VM 并行跑同一来源的流量副本（读日志、跑联调）但不接用户流量，验证日志、指标、依赖调用一致；再灰度切流（Ingress/网关权重 10%→50%→100%），每档对比错误率与延迟；全量后 VM 保留 2-4 周兜底，可一键切回。单体特有坑：① session 在进程内——灰度切流后用户掉登录，改造期用 sticky session 或先把 session 外置到 Redis；② 定时任务双跑——crontab 迁到容器侧时保证同一时刻只有一份执行（分布式锁，或先停 VM 任务再启容器任务）；③ 连接池规模——容器副本数多于原 VM，DB/Redis 连接池上限按"最大副本数 × 每副本连接数"重算，否则连接池打满数据库。验收：错误率/延迟与 VM 基线对齐、发布耗时与恢复时间量化记录。

**延伸考点：** 影子部署阶段"不接流量"怎么验证真实行为（日志对比、回放流量、链路追踪采样）？灰度切流期间用户 session 怎么平滑迁移（Redis 外置与双读双写）？

---
