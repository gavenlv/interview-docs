# DevOps · Nginx（面试题库）

本文件面向 DevOps、SRE 与后端工程师，考察候选人在 Nginx 上的真实落地能力：master/worker 进程模型、反向代理与负载均衡、代理缓存、SSL/TLS、限流防刷、location 匹配、日志、性能调优、高可用与排障，以及 Nginx 在云原生（Ingress）中的角色。所有题目均以线上真实场景切入，不考八股文，重点看候选人能否给出量化依据、讲清方案取舍，并用 nginx.conf 配置片段、命令与排障过程佐证实践。题目难度从实践基础到架构级渐进，覆盖日常配置到大规模网关治理的全链路。

---

### Q1. Nginx 架构：master/worker 进程模型、事件驱动（epoll）、worker 数怎么定

**问题：** 你们线上 Nginx 是单机还是多机？一个 Nginx 进程会启动哪些进程、各自干什么？8 核 16G 的机器上 worker_processes 应该配多少，依据是什么？

**期望加分项：**
- 能讲清进程模型：1 个 master（只负责解析配置、管理 worker、信号处理）+ N 个 worker（真正处理连接），还能提到 cache manager / cache loader 两个辅助进程
- 理解事件驱动本质：worker 内用 epoll 单线程事件循环处理成千上万连接（C10K），每个 worker 可支撑数万并发连接，而非"一个连接一个线程"
- worker 数有量化依据：默认 auto 按 CPU 核数配（Nginx 官方建议），8 核配 8；同时知道 trade-off——worker 太多会因 CPU 争抢、锁竞争反而变慢，IO 密集场景（如代理大量文件）可适当调多
- 对比过 Apache：Apache 是进程/线程模型（prefork/worker/event），内存占用高、高并发下性能差，Nginx 用事件驱动天然适合做入口网关
- 能讲 worker 亲和性（worker_cpu_affinity）与 worker_priority 等进阶调优项，知道默认 worker 共享 listen socket 通过互斥锁 accept（accept_mutex）

**减分项：**
- 说成"Nginx 就是单进程/双进程"，讲不清 master 与 worker 分工
- 只会背"epoll"，讲不出它和 select/poll 的差异（O(1) vs O(n)、水平/边沿触发）
- worker 数没有依据，只会"默认 4 个"或"越大越好"
- 讲不清 reload 时 worker 的平滑替换过程（新配置 → 新 worker 接管 → 旧 worker 优雅退出）
- 无任何调优实测数据佐证

**解答：**
先讲进程模型：启动后 `ps -ef | grep nginx` 能看到 1 个 master 进程（root 权限，负责读配置、fork worker、处理 reload/升级信号）和若干 worker（通常以非 root 运行，处理全部请求）；若开了 proxy_cache 还会有 cache manager（淘汰过期缓存）和 cache loader（加载缓存元数据）。worker 之间完全对等，通过共享的 listen socket 争抢 accept（默认开启 accept_mutex 避免惊群）。事件驱动是核心：每个 worker 是一个事件循环，用 epoll 同时监听数万 fd，连接到来时注册可读/可写事件，IO 就绪再回调处理，因此"一个 worker、一个线程"就能撑住高并发——这正是对比 Apache（进程/线程模型，C10K 时内存与切换开销爆炸）的关键。worker 数：官方推荐 `worker_processes auto`（按核数），8 核配 8，因为事件驱动下 worker 数 ≈ CPU 核数即可占满 CPU；配多了反而因上下文切换和互斥锁争抢掉性能。实践中的坑：一是大量磁盘 IO 场景（如代理大文件下载）worker 会阻塞在 IO，可适当多于核数（如 ×1.5）；二是 `worker_rlimit_nofile` 要与 worker_connections 匹配，否则连接数上不去；三是调优要看 `worker_cpu_affinity` 做 CPU 绑核（尤其多路 CPU 机器），并压测验证：`ab -c 10000` 对比不同 worker 数的 QPS 和延迟，而不是拍脑袋。

**延伸考点：** reload 时旧 worker 上的长连接（WebSocket）会被怎样处理？accept_mutex 关闭后在高并发下的惊群问题如何表现？

---

### Q2. 反向代理基础：proxy_pass、转发头、upstream 选择、超时设置

**问题：** 现在要把请求 `/api/` 反向代理到后端 Java 服务 `10.0.1.10:8080`，并确保后端拿到的 Host、真实 IP、协议都是对的。请给出配置并说明 proxy_pass 带不带 URI 的区别。

**期望加分项：**
- 能写出完整配置：`location /api/ { proxy_pass http://api_backend; proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme; }`，并解释每个头的作用
- 讲清 proxy_pass 带 URI 的经典坑：`proxy_pass http://backend;`（不带 URI）保留原 URI，`proxy_pass http://backend/;`（带 /）会用 / 替换匹配到的 location 前缀——例如 `location /api/` 配 `proxy_pass http://backend/;` 时请求 `/api/user` 会变成 `/user`
- 能讲 upstream 与 proxy_pass 结合：上游写在 http 块的 upstream 里（带负载均衡语义），proxy_pass 直接指 IP 则无均衡
- 超时三件套有量化：`proxy_connect_timeout`（默认 60s，内网可压到 2-5s）、`proxy_send_timeout`、`proxy_read_timeout`（默认 60s，慢接口需调大或按 location 单独覆盖），并知道 504 与 read_timeout 的关系
- 知道 proxy_redirect 处理后端 302 回跳错误、proxy_set_header 里 Connection 对长连接的影响
- 有线上佐证：如因缺少 X-Forwarded-Proto 导致后端生成 http 链接、用户被重定向回 http 的案例

**减分项：**
- 只背配置不带 URI 与带 URI 的区别解释不清
- 漏配 Host/X-Forwarded-For，答不出后端日志里 IP 全是 Nginx 地址的原因
- 超时只会默认值，讲不清 504 与 proxy_read_timeout 的关联
- 不懂 `$proxy_add_x_forwarded_for` 与直接写 `$http_x_forwarded_for` 的差异（前者会追加本机 IP）
- 没有真实代理链路调试经验

**解答：**
先给配置：http 块定义 upstream `upstream api_backend { server 10.0.1.10:8080; }`，server 块里 `location /api/ { proxy_pass http://api_backend; proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme; }`。核心考点是 proxy_pass 的 URI 语义：不带 URI（`http://api_backend;`）时 Nginx 把完整原请求 URI 转发，`location /api/` 匹配到的 `/api/user` 原样发给后端；带 URI（`http://api_backend/;`）则用新 URI 替换匹配前缀，`/api/user` 变成 `/user`——这是最常见的转发路径丢失事故源。转发头方面：Host 默认会传原始 Host，但显式设置更稳妥；`$proxy_add_x_forwarded_for` 会把 `$remote_addr` 追加到已有的 XFF 末尾（若客户端伪造了 XFF 会保留伪造值，这就是为什么真实 IP 场景要用 proxy_protocol 或只信任第一跳）；`X-Forwarded-Proto $scheme` 缺失会让后端以为全是 http，生成错误重定向。超时三件套务必按场景覆盖：默认 `proxy_connect_timeout 60s` 对内网后端太长，建议 3-5s 快速失败；`proxy_read_timeout` 决定后端不返回时等多久，默认 60s，大查询/流式接口需按 location 覆盖调大，否则 504。实践坑：后端返回 302 且 Location 是内网地址时，Nginx 默认会改写（proxy_redirect 默认按 proxy_pass 对应关系改写），改错会造成二次转发，可显式 `proxy_redirect off` 观察。

**延伸考点：** 后端如何区分"直连 Nginx"与"客户端伪造 XFF"？proxy_pass 到域名（带 resolver）与到 IP 有什么不同？

---

### Q3. 负载均衡：upstream 配置、权重、ip_hash/least_conn/url_hash、健康检查（主动/被动）

**问题：** 三个后端节点性能不一致（一台 8 核、两台 4 核），且其中一台经常因慢 SQL 变慢但不挂。你要怎么配置 upstream，让流量分配合理、又能自动摘掉故障节点？

**期望加分项：**
- 能写完整 upstream：`upstream app { server 10.0.1.1:8080 weight=4 max_fails=3 fail_timeout=30s; server 10.0.1.2:8080 weight=2; server 10.0.1.3:8080 weight=2 backup; }`，解释 weight 依据（按核数/容量 4:2:2）
- 讲清各调度算法适用场景：默认 round-robin 加权轮询；`ip_hash` 会话保持（同一 IP 到同一后端，代价是热点 IP 打满单节点、后端增减会打乱映射）；`least_conn` 适合长连接/慢请求场景（按活跃连接数分配）；`url_hash`/`hash $request_uri` 适合缓存型后端（同 URL 永远打到同一节点）；`least_time`（商业版）按响应时间分配
- 健康检查有层次：免费版只有被动检查（max_fails 计数 + fail_timeout 熔断，基于转发失败），开源版 1.9+ 的 `ngx_http_upstream_module` 没有主动健康检查（需 nginx-plus 或第三方模块/脚本探测），可结合外部探活或 L4 方案
- 能讲故障转移语义：max_fails=3 指 fail_timeout（30s）内失败 3 次标记不可用，期间不再转发；配合 backup 备用节点实现故障接管
- 讲得出 ip_hash 与一致性 hash 的差别：节点增减时 ip_hash 重映射率高，生产用 `hash $remote_addr consistent`（一致性哈希）更平滑
- 有线上案例：如会话丢失（ip_hash 下节点扩容导致登录态漂移）、慢节点拖垮整体（未配 fail_timeout）

**减分项：**
- 只会默认轮询，讲不出算法选择与业务场景的对应关系
- 把 max_fails/fail_timeout 当背的，说不出熔断判定与恢复机制
- 分不清主动/被动健康检查，误以为免费版 Nginx 有主动探活
- 忽略会话保持需求（接口无状态与有状态混用）
- 没有节点容量差异下的权重设计意识

**解答：**
权重优先按容量：8 核节点 `weight=4`、4 核节点 `weight=2`，用 `weight` 表达服务能力差异，这是最直观也最可控的做法；若后端是长连接或慢查询型，`least_conn` 比轮询更稳（轮询会把新请求均分给正在慢处理的节点，least_conn 会绕开活跃连接多的节点）。会话保持方面：业务无状态优先不依赖 ip_hash（它会带来 IP 热点和扩缩容漂移）；确有状态时优先把 session 外置（Redis），必须粘性则 `ip_hash` 或 `hash $remote_addr consistent`——一致性哈希在增删节点时只影响 1/n 的映射，重映射率远低于 ip_hash，这是生产中容易忽略的坑。健康检查要分清层次：免费版只有被动检查——`max_fails=3 fail_timeout=30s` 表示 30s 窗口内转发失败 3 次即把节点标记为不可用，fail_timeout 到期后自动重新探测（放一个请求试水，成功则恢复）；主动检查（定时发 HTTP 探活、可配请求路径/预期状态码）是 Nginx Plus 的 `health_check`，开源场景可用 Lua 脚本、外部定时 curl 探测上游或借助 k8s ingress 的探针实现。故障转移配置建议：`backup` 备节点只在主节点全挂时启用，配合 `max_fails=2 fail_timeout=10s` 让摘除更灵敏；同时把 `proxy_next_upstream http_502 http_504` 打开（默认仅 error 与 502/503/504 的 error 情形），可让单次转发失败立刻换节点重试，降低失败率。实践坑：proxy_next_upstream 对 POST 请求重试可能造成重复提交，需按幂等性谨慎开启。

**延伸考点：** 后端节点扩缩容时 ip_hash 与一致性 hash 的请求重映射比例分别是多少？如何用 Nginx 变量把"当前命中的上游节点 IP"记录到 access log 用于问题追查？

---

### Q4. 静态资源服务：root vs alias、expires 缓存、gzip 压缩、etag、sendfile

**问题：** 前端打包产物放在 `/data/www/blog`，构建产物文件名带 hash。请配置 Nginx 直接伺服静态资源，让图片走强缓存、HTML 走协商缓存，并开启 gzip，同时讲清 root 与 alias 的区别。

**期望加分项：**
- 能写配置：`server { root /data/www/blog; location / { try_files $uri $uri/ /index.html; } location ~* \.(js|css|png|jpg|svg|woff2)$ { expires 30d; add_header Cache-Control "public, immutable"; } }`，并解释 root/alias 差异
- root vs alias 讲得透：root 是"把 URI 拼到 root 后面"（`root /data/www/blog;` 请求 `/a.png` → `/data/www/blog/a.png`），alias 是"URI 路径与文件系统路径对应关系替换"（`location /static/ { alias /data/www/img/; }` 请求 `/static/x.png` → `/data/www/img/x.png`），alias 常用于 location 前缀与目录名不一致时；alias 必须以 / 结尾否则拼接错误
- 缓存策略有依据：带 hash 文件名 → 长缓存 + immutable（30d/1y）；`index.html` 走协商缓存：`expires -1` 或 `Cache-Control: no-cache` + ETag/Last-Modified，配合 `try_files ... /index.html` 做 SPA 路由回退
- gzip 配置：`gzip on; gzip_types text/plain text/css application/javascript application/json image/svg+xml; gzip_min_length 1k; gzip_comp_level 5; gzip_vary on;`（图片/视频本身已压缩无需再 gzip），知道 gzip 与 Brotli（ngx_brotli）的取舍
- 性能相关：`sendfile on;`（零拷贝，内核态直接送文件，避免用户态拷贝）、`tcp_nopush`/`tcp_nodelay`、`open_file_cache` 缓存文件句柄，并知道 ETag/Last-Modified 是协商缓存凭据，304 由 Nginx 自动返回
- 有线上数据：如 gzip 后 HTML/JS 体积下降 70%、CDN 命中率、304 比例

**减分项：**
- root 与 alias 混用出路径错误，且讲不清底层拼接规则
- 静态资源一律 expires 30d，导致 index.html 也被缓存、发版后用户拿到旧页面
- 不懂 sendfile 零拷贝，只会背配置
- gzip 对 jpg/png 也开，纯属浪费 CPU；或漏掉 json/svg 类型
- 不讲缓存与版本号的联动（不带 hash 的文件名不适合长缓存）

**解答：**
先分清 root 与 alias：root 做"前缀拼接"，`root /data/www/blog;` 时请求 `/js/app.js` 映射到 `/data/www/blog/js/app.js`，因此 root 通常配在 server 层或与 location 前缀同名的目录场景；alias 做"路径替换"，`location /static/ { alias /data/www/files/; }` 时 `/static/a.png` 映射到 `/data/www/files/a.png`，当 location 前缀和实际目录名不一致时必须用 alias，且 alias 后面必须以 / 结尾，否则拼接会少一层目录——这是最经典的一个坑。缓存策略要与构建产物联动：`index.html` 这类入口文件用 `expires -1` + ETag（Nginx 默认生成 ETag，协商缓存命中返回 304，流量极小）；带 hash 的 js/css/img 用 `location ~* \.(?:js|css|png|jpg|jpeg|gif|svg|woff2?)$ { expires 365d; add_header Cache-Control "public, immutable"; }`——immutable 告诉浏览器无需 revalidate，但前提是文件名 hash 变化时必须生成新 URL，否则发版后用户永远拿到旧资源。压缩：`gzip on; gzip_types text/plain text/css application/javascript application/json image/svg+xml application/xml; gzip_min_length 1k; gzip_comp_level 5; gzip_vary on;`（min_length 防止压缩小文件浪费 CPU；jpg/png/webp 自带压缩别开；Brotli 压缩率再高 15-20% 但需第三方模块）。IO 层：`sendfile on` 让内核直接从磁盘到 socket（零拷贝），大文件下载 QPS 显著提升；`open_file_cache max=10000 inactive=60s;` 缓存 fd 与 stat 结果，静态场景命中率高；SPA 路由回退 `try_files $uri $uri/ /index.html;` 保证刷新页面不 404。实践坑：大并发下 304 请求本身也会占用连接，静态资源应尽量交给 CDN 分担；ETag 在 sendfile 场景默认开，若后端生成文件需一致性校验时注意。

**延伸考点：** 为什么 CDN 场景建议关闭 ETag（多源回源时 ETag 不一致导致 CDN 频繁回源）？try_files 的查找顺序与 $uri/（目录）匹配在什么场景会出问题？

---

### Q5. 代理缓存：proxy_cache 配置、缓存键、缓存失效、缓存命中率、purging

**问题：** 后端是一个慢接口（平均 800ms，结果 30 分钟内基本不变），峰值时把后端打垮。你在 Nginx 上加一层缓存，要求：命中直接回、后端更新后能尽快看到新数据。请给出配置并讲清缓存键与失效策略。

**期望加分项：**
- 能写完整缓存配置：http 块 `proxy_cache_path /data/nginx_cache levels=1:2 keys_zone=cache_api:100m max_size=10g inactive=60m use_temp_path=off;`，server 块 `location /api/ { proxy_cache cache_api; proxy_cache_key "$scheme$request_method$host$request_uri"; proxy_cache_valid 200 30m; proxy_cache_valid 404 1m; add_header X-Cache-Status $upstream_cache_status; }`
- 讲清 cache_key 构成：默认是完整 URI，需区分 query 时自动带上；`$request_method` 防 POST 误缓存；必要时把 header（如版本/语言）拼进 key；过大 key 用 `proxy_cache_key ...; proxy_cache_valid` 时用 md5 摘要
- 失效策略有层次：`proxy_cache_valid 200 30m` 时间失效；`proxy_cache_purge`（商业版/模块）按 URL 主动清除；`proxy_cache_bypass`（跳过读缓存）与 `proxy_no_cache`（跳过写缓存）配合按参数/header 穿透；inactive 与 expired 的语义区别（inactive 是"无访问即淘汰"）
- 命中率意识：`$upstream_cache_status` 打点（HIT/MISS/EXPIRED/BYPASS/STALE），统计命中率并调优；多级缓存（浏览器→CDN→Nginx→后端）
- 并发回源控制：`proxy_cache_lock on` 防止缓存击穿（同一个 key 并发请求全部打后端，lock 保证仅一个回源，其余等待）
- 有线上案例：如忘记区分 query 导致不同用户共享缓存、后端更新后缓存 30 分钟不更新被投诉、缓存目录写满磁盘

**减分项：**
- 只配了 proxy_cache_valid 就以为完了，讲不清缓存键与命中率
- 分不清 inactive 与过期时间，或答不出 expired 数据在回源失败时是否可用（proxy_cache_use_stale）
- 忽略缓存击穿/雪崩：无 proxy_cache_lock、无 fallback 策略
- 说不清 $upstream_cache_status 各取值含义
- 无线上命中率数据与调优过程佐证

**解答：**
缓存四要素：存储（proxy_cache_path）、键（proxy_cache_key）、有效期（proxy_cache_valid）、状态可观测（add_header X-Cache-Status）。配置核心如下：`proxy_cache_path /data/cache levels=1:2 keys_zone=api_cache:100m max_size=10g inactive=60m use_temp_path=off;`（levels=1:2 控制目录分层防单目录文件过多；keys_zone 内存里的键索引；max_size 超限由 cache manager 按 LRU 淘汰；use_temp_path=off 避免跨盘 rename）；location 里 `proxy_cache api_cache; proxy_cache_key "$scheme$host$request_uri"; proxy_cache_valid 200 30m; proxy_cache_valid 404 1m; proxy_cache_lock on; add_header X-Cache-Status $upstream_cache_status;`。键设计是灵魂：默认 key 含 URI（query 自动纳入），业务要区分参数/Header 时显式扩展，如按用户维度 `proxy_cache_key "$uri$is_args$args|$http_accept_language"`；注意 POST 默认不进缓存，若开了 `proxy_cache_methods POST` 必须把 method 加入 key 防串数据。失效是实践难点：时间失效（30m）满足"后端更新后尽快可见"，但接口发布更新时 30 分钟内用户仍拿到旧数据——所以要对"可感知更新"的接口配合 `proxy_cache_bypass $http_cache_control` 或带版本参数（如 `/api/v2/` 路径天然换 key），或部署脚本调用 purge 接口主动清 key。防护层面：`proxy_cache_lock on` 解决击穿（并发同 key 只放一个回源）；回源失败时 `proxy_cache_use_stale error timeout` 可让过期缓存兜底（牺牲一致性换可用性，适合读多写少）。上线后必须盯 `$upstream_cache_status`：命中率长期低于 70% 说明键粒度过细或有效期太短；MISS 集中在热点 key 说明 lock 未开导致穿透。实践坑：缓存目录放系统盘会把根分区写满，务必挂独立磁盘并设 max_size；keys_zone 与文件数不匹配（文件数远超内存键槽）会导致大量 MISS。

**延伸考点：** proxy_cache_valid 与 Cache-Control 响应头（s-maxage）谁优先？缓存击穿与缓存雪崩在 Nginx 层分别怎么防？

---

### Q6. SSL/TLS：证书配置、TLS 版本与套件、HSTS、性能优化（session cache/OCSP stapling）

**问题：** 给站点上 HTTPS，要求：兼容 Chrome/Safari 等主流浏览器、禁掉老弱协议、证书更新不能停机、页面首次加载的 TLS 握手尽量快。你会怎么配置？

**期望加分项：**
- 能写完整 server 配置：`listen 443 ssl; ssl_certificate /etc/nginx/ssl/fullchain.pem; ssl_certificate_key /etc/nginx/ssl/privkey.pem; ssl_protocols TLSv1.2 TLSv1.3; ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...'; ssl_prefer_server_ciphers off; ssl_session_cache shared:SSL:10m; ssl_session_timeout 10m; ssl_stapling on; ssl_stapling_verify on; add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;`
- TLS 版本与套件取舍有依据：TLS1.2/1.3（1.0/1.1 已废弃，2020 年后主流浏览器不支持）；优先 ECDHE 前向保密 + AES-GCM/CHACHA20；`ssl_prefer_server_ciphers off`（TLS1.3 由客户端选择套件）
- HSTS 讲得清：`Strict-Transport-Security` 强制浏览器走 HTTPS 并忽略证书错误，max-age 设定强制时长；"preload" 提交到浏览器预载列表；坑：首次访问是 http 时 HSTS 不生效，需要 301 跳转配合，且开了 HSTS 后不能轻易回退 http
- 性能优化有量化：`ssl_session_cache shared:SSL:10m`（约 10m 可存 8 万 session，同一客户端复用 session 免全握手，首屏 TLS 时间从 2-RTT 降到 1-RTT）；`ssl_stapling on` 由 Nginx 代取 OCSP 响应并附带给客户端，省去客户端单独查 OCSP 的额外 RTT；TLS1.3 本身 0-RTT/1-RTT
- 证书管理：Let's Encrypt + certbot --nginx 自动续期，`systemctl reload nginx` 生效；合拼证书链 fullchain.pem = 站点证书 + 中间证书，顺序错了浏览器会报链不完整
- 会用 `openssl s_client -connect host:443` 与 ssllabs 检测套件/版本/链，能看懂评级

**减分项：**
- 只贴配置讲不出"为什么"，如不知道 ECDHE 的意义（前向保密）
- 直接禁用 TLS1.2 只留 1.3，导致老设备（如旧 Android）全挂
- 开 HSTS 但没配 301，或把 HSTS max-age 设得巨大后想回退 http 下不来
- 不知道 session cache/OCSP stapling 这两个握手优化点
- 证书续期流程没落地过（手动换证书忘 reload、链拼接错误）

**解答：**
配置骨架如上，重点讲三块：协议与套件、HSTS、握手优化。协议层：`ssl_protocols TLSv1.2 TLSv1.3;` 是当前主流基线——TLS1.0/1.1 在 RFC 层面已废弃且 PCI-DSS 不认可，但别只留 1.3，仍有不少老客户端（旧 Android 7、部分 IoT）只支持 1.2；套件优先 `ECDHE-ECDSA-AES128-GCM-SHA256` 这类 ECDHE（前向保密：私钥泄露也无法解历史流量）+ AES-GCM（AEAD 硬件加速快），`ssl_prefer_server_ciphers off` 在 TLS1.3 下让客户端选套件更合理。HSTS：`add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;`（always 保证 4xx/5xx 也下发），强制浏览器只能走 HTTPS；部署顺序必须"先 301 跳转 + HSTS，再观察一段时间"——一旦 preload 生效，HTTPS 证书出问题用户将完全无法访问（浏览器拒绝绕过），这是最狠的一个坑。握手优化是性能关键：默认每次新连接都全握手（TLS1.2 要 2 个 RTT），`ssl_session_cache shared:SSL:10m;` 后同一客户端复用 session 只 1 个 RTT；`ssl_stapling on; ssl_stapling_verify on;` 让 Nginx 定期从 CA 拉 OCSP 响应并随握手发给客户端，客户端免去查询 OCSP 的 RTT 且更隐私。证书管理实践：Let's Encrypt 证书 90 天有效期，用 certbot 定时任务自动续期 + `nginx -t && nginx -s reload` 平滑生效（无需停机）；certbot 生成 fullchain.pem（站点证书 + 中间证书）与 privkey.pem，顺序反了或漏中间证书会报 "unable to get local issuer certificate"。验证手段：`echo | openssl s_client -connect www.example.com:443 2>/dev/null | grep -E "Protocol|Cipher|Verify"` 看版本与套件，ssllabs 打分自查。

**延伸考点：** 证书到期前 30 天/7 天如何告警（脚本检查 `openssl x509 -enddate` 或监控 fullchain 的 mtime）？HTTP/2 与 HTTPS 的绑定关系，为什么 h2 只能跑在 TLS 之上（h2c 除外）？

---

### Q7. 限流与防刷：limit_req/limit_conn 原理（漏桶）、burst/nodelay、IP 限流、封禁

**问题：** 线上接口被脚本高频刷（单 IP 每秒几十个请求），偶尔还有正常用户瞬间的突发流量。请用 Nginx 限流，要求：压制恶意高频、但不误伤正常突刺，并说明限流算法原理。

**期望加分项：**
- 能写配置：http 块 `limit_req_zone $binary_remote_addr zone=req_zone:10m rate=10r/s;`，location 里 `limit_req zone=req_zone burst=20 nodelay;`，并解释 rate/burst/nodelay 三者的语义
- 讲清漏桶算法：limit_req 是漏桶（恒定速率出水），rate=10r/s 即令牌按 100ms/个补充；burst=20 是桶容量，允许瞬时积压最多 20 个；不加 nodelay 时积压请求会被限速排队（均匀吐出），加 nodelay 则突发 20 个立刻放行（只对超出的排队，即"直接拒绝多余"）——nodelay 的精确语义是"突发请求不延迟、直接处理，超限部分立即返回 503"
- 讲清 limit_conn：按连接数限制（`limit_conn_zone $binary_remote_addr zone=conn_zone:10m; limit_conn conn_zone 10;`），常用于限制每 IP 并发连接（防爬虫/防 CC）
- 业务区分：限流 key 可以换成 `$server_name$request_uri`（接口级限流）或 `$http_user_agent` 等；`limit_req_status 429`（或 503）可自定义；`limit_req_log_level warn` 观察
- 有量化实践：rate 从压测得出（正常用户 P99 请求率 ×1.5）、nodelay 误伤分析（正常用户突发在 burst 内不受伤）、配合 `geo`/`map` 对白名单 IP 放行（`limit_req zone=req_zone burst=20 nodelay; if 白名单跳过` 用 geo + 空 zone 技巧）
- 防刷升级：配合 `$http_user_agent` 封爬虫、fail2ban 动态封禁、WAF 层防 CC；`limit_req_zone` 内存 1m ≈ 1.6 万 IP 的量级估算

**减分项：**
- 分不清 limit_req（速率）与 limit_conn（连接数），或把令牌桶当漏桶讲
- 只会 rate 不会 burst/nodelay，或者 nodelay 语义讲错（"nodelay 就是不限流"）
- 限流配置后所有正常用户被误杀，无白名单/降级预案
- 不知道超限时返回什么状态码、怎么自定义、怎么打日志观测
- 没有从压测数据反推 rate 的过程

**解答：**
先建立模型：limit_req 实现的是漏桶（恒定速率 + 桶容量），`limit_req_zone $binary_remote_addr zone=req_zone:10m rate=10r/s;` 定义限流维度（用 $binary_remote_addr 二进制 IP 省内存，1m 约存 1.6 万 IP）和速率；location 里 `limit_req zone=req_zone burst=20 nodelay;` 生效。rate=10r/s 表示每 100ms 释放一个请求槽位；burst=20 是桶容量，允许 20 个请求瞬时积压。nodelay 是最大考点：不加 nodelay 时，积压请求被"限速"按 rate 排队吐出（延迟会被拖长，相当于把突刺拉平）；加 nodelay 后，burst 内的突发请求立即放行不排队，只有超过 burst 的部分直接返回 503（默认）——所以 nodelay 适合"允许突发但限制峰值"的场景，这正是对抗刷接口的标配。生产实践按此分层：① 接口级限流——`limit_req_zone $binary_remote_addr$request_uri zone=api_zone:10m rate=20r/s;` 对核心接口单 IP 限 20r/s（rate 依据压测：正常用户单 IP P99 请求率 ×2 留余量）；② 并发连接限制——`limit_conn_zone $binary_remote_addr zone=conn_zone:10m; limit_conn conn_zone 20;` 防爬虫开大量连接；③ 白名单——用 `geo $whitelist { default 0; 10.0.0.0/8 1; }` + map 出空 zone 的技巧：`map $whitelist $limit_key { 1 ""; default $binary_remote_addr; }` 然后 `limit_req_zone $limit_key zone=req_zone:10m rate=10r/s;`，白名单 IP 的 key 为空字符串即不限流。④ 观测与告警——`limit_req_status 429;`（语义化状态码）+ `limit_req_log_level warn;`，日志里 429 比例突增即触发防刷告警。实践坑：rate 单位有 `r/s` 与 `r/m`，小速率必须用 r/m（rate 低于 1r/s 时直接配 r/s 会报错）；burst 设太大等于没限；nodelay 与 limit_req_zone 的 zone 名写错会静默不生效（nginx -t 不报错），务必用 429 计数验证生效。

**延伸考点：** limit_req 的 nodelay 与"令牌桶"的差异——突发后立即放行，那下一秒的速率如何恢复？如何在 Nginx 层实现"封禁名单"动态生效（reload vs 共享内存/Lua）？

---

### Q8. 重写与重定向：rewrite/return/if、rewrite 与 location 的关系、陷阱

**问题：** 站点要从 `http://www.old.com/a.php?id=1` 迁移到 `https://www.new.com/path/1`，同时老的伪静态规则要保留。rewrite、return、if 各在什么场景用？为什么说"能 return 就不 rewrite"？

**期望加分项：**
- 能写规则：`server { listen 80; server_name old.com; return 301 https://www.new.com$request_uri; }` 和路径改写 `rewrite ^/a\.php\?.*$ /redirect_new permanent;`，并说明 permanent(301)/redirect(302)/break/last 的区别
- 讲清三者的定位：return 最简单高效（直接返回状态码，配合 $request_uri 做整站跳转）；rewrite 做 URI 内部改写（不改浏览器地址栏），rewrite 后还能继续匹配 location；if 是"最后手段"（官方文档明说 if 在 location 里是邪恶的，仅用于 rewrite 模块变量判断），尽量用 map/geo 替代
- 讲透 rewrite 与 location 的关系：rewrite 发生在 location 匹配之后；`last` 会重新发起 location 匹配（rewrite 后重新选 location，可能进别的 location 导致二次 rewrite），`break` 停止 rewrite 继续处理当前 location；rewrite 里可用捕获组 $1/$2
- 讲清常见陷阱：rewrite 规则里出现 `if (-f $request_filename)` 检查文件存在；rewrite 死循环（rewrite 到自身且无终止条件，用 `break`/`last` 跳出）；`rewrite ^/(.*)$ /$1.php;` 在 nginx 里不会像 Apache 那样自动映射扩展名
- 301/302 的语义选择：迁移用 301（永久，搜索引擎收录更新），临时跳转用 302（307 保 POST 方法）；注意 301 会被浏览器/爬虫缓存，回退难
- 有线上案例：rewrite 写成死循环 500 错误、location 里 if 判断导致 500（if 里用 proxy_pass 的坑）

**减分项：**
- 把 return 与 rewrite 混为一谈，说不出何时该用哪个
- 讲不清 last 与 break 的执行差异，或不知道 rewrite 后重新匹配 location 的影响
- 在 location 里滥用 if（尤其 if 里配 proxy_pass 导致 500 的经典坑）
- 不懂 301/302/307 的语义差异和缓存影响
- 无真实改写规则调试经验（rewrite_log on 调试）

**解答：**
三条准则：整站跳转用 return、URI 内部改写用 rewrite、条件逻辑优先 map/geo 而非 if。① return：`return 301 https://www.new.com$request_uri;` 把 http 老域名整站 301 到 https 新域名（$request_uri 保留完整路径参数），性能最好（无正则、无内部跳转），这是迁移场景首选；对特定状态码还可以 `return 404;`、`return 403;`。② rewrite：做 URI 改写，`rewrite ^/a\.php$ /redirect_new permanent;` 会把 /a.php 改写成 /redirect_new 再走新 location；修饰符 last 表示"停止当前 rewrite，重新做 location 匹配"（用于改写后希望进新 location 的场景），break 表示"停止 rewrite 阶段，继续当前 location 处理"（用于代理/静态场景，避免改写后被再次匹配），redirect/permanent 则直接回 302/301 给客户端——注意只有 rewrite 带 last/break 才是内部改写，带 redirect/permanent 是外部跳转。③ if 的坑：官方文档明确 if 在 location 里"evil"——if 里用 proxy_pass 时 Nginx 会按"未用 location 的默认行为"处理（可能 500 或错误转发），推荐用 map 把条件映射成变量再在 proxy_pass 里引用。rewrite 与 location 的执行次序是高频坑：请求先匹配 location，再执行该 location 内的 rewrite；rewrite 用 last 后重新匹配 location 时可能落入别的 location 再次触发其 rewrite，设计不好会死循环（表现是 500/超时），排障时开 `rewrite_log on; error_log ... notice;` 直接看日志里每一步改写。实践案例：老站 .php 伪静态改版，在 location 里 `rewrite ^/news/(\d+)\.html$ /news.php?id=$1 break;` 配 proxy_pass——break 保证改写后不再重新匹配 location，否则会再次进该正则 location。最后：301 会被浏览器和搜索引擎长期缓存，迁移回归测试期间先用 302 观察，确认无误再切 301。

**延伸考点：** 为什么 if 在 location 中使用 proxy_pass 会出问题（Nginx 的 if 上下文限制）？rewrite 的 last 重新匹配 location 时，优先级规则（正则 vs 前缀）与第一次匹配有何不同？

---

### Q9. location 匹配：前缀/精确/正则/^~ 的匹配优先级、location 内嵌套

**问题：** 一个站点同时有 `/static/` 静态目录、`/api/` 代理、`/user/123` 这样的正则路由、根路径回退。请说明 location 的匹配优先级，并配置一个不踩坑的 server。

**期望加分项：**
- 能完整背出匹配优先级：① `=` 精确匹配（最高）；② `^~` 前缀匹配（匹配到后不再查正则）；③ 普通前缀匹配（最长优先）；④ `~`/`~*` 正则（按配置顺序，第一个匹配生效）；⑤ 无匹配时根 location `/`
- 能讲清"普通前缀匹配 + 正则"的机制：先按前缀最长匹配记住候选，再按书写顺序尝试正则，正则命中就覆盖前缀候选——这就是"前缀优先但正则最终说了算"；`^~` 的作用是打断这一流程（匹配到就不看正则了），`=` 则完全精确
- 能写示例 server：`location = /favicon.ico { access_log off; }`、`location ^~ /static/ { alias /data/www/static/; }`、`location ~* \.(js|css)$ { expires 30d; }`、`location /api/ { proxy_pass http://backend; }`、`location / { try_files $uri $uri/ /index.html; }`
- 知道 location 内的嵌套规则：Nginx 不支持 location 嵌套（正则 location 内不能嵌套；前缀 location 内嵌套实质是"该 location 内再按前缀匹配"，实践上很少用，官方不推荐），常用做法是用多个 location + include 或 try_files 组合
- 能讲反向案例：正则 location 写在前面会抢走所有请求、`location /static` 不带 ^~ 时被正则 location 覆盖的坑
- 排障手段：`nginx -T` 导出配置核对、临时开 debug 日志或在 location 里 `add_header X-Loc matched;` 验证命中

**减分项：**
- 优先级口诀背不全或顺序颠倒（如把正则放最优先）
- 不知道正则按书写顺序匹配（第一个命中即停）
- 把 ^~ 与 = 混为一谈
- 说 Nginx 支持 location 嵌套并试图嵌套配置
- 无法解释"前缀匹配到了但被后面正则覆盖"的经典现象

**解答：**
匹配流程四步：① `=` 精确匹配，命中即终止（适合精确 URL 如 `/favicon.ico`、`/healthz`）；② `^~` 前缀匹配，命中后立即终止、不再查正则；③ 普通前缀匹配，记录"最长匹配"作为候选但不终止；④ 按配置书写顺序执行正则 `~`/`~*`，第一个命中的正则覆盖候选结果。没有正则命中时用第 ③ 步的前缀候选，都没有则落到 `location /`。所以"前缀最优先、正则最终拍板"是核心记忆点。典型 server 配置：`location = /favicon.ico { log_not_found off; access_log off; }`（精确匹配防 404 噪音）；`location ^~ /static/ { alias /data/www/static/; expires 30d; }`（静态目录用 ^~ 阻断正则，避免 `~* \.(js|css)$` 正则再插一脚导致缓存/路径行为不一致）；`location ~* \.(?:js|css|png)$ { expires 30d; }`（文件后缀级规则）；`location /api/ { proxy_pass http://api_backend; }`；`location / { try_files $uri $uri/ /index.html; }`（SPA 兜底）。常见坑：正则 `~* \.php$` 若写在普通前缀之前并全局生效，会把不该处理的路径也抢走，务必限定在 `location ~* ^/api/.*\.php$`；`location /static`（无 ^~）且后面存在 `~* \.html$` 时，/static/a.html 会被正则命中——表现就是"配了 static 目录缓存却不生效"，用 ^~ 解决；反向场景：想让某前缀下的所有请求都走同一规则，就在该前缀 location 里用 try_files/include 组合，Nginx 本身不支持 location 嵌套（文档明确），不要试图在 location 里再写 location。验证手段：临时在每个 location 里 `add_header X-Hit location_name;` 用 curl -I 看响应头判断命中，或开 `error_log debug`（生产慎用）看 "using configuration" 日志行。

**延伸考点：** `location /` 与 `location = /` 的差异（所有请求的兜底 vs 只有根路径）？正则 location 里能否再写 proxy_cache/expires 之外的其他指令，嵌套限制的真正边界是什么？

---

### Q10. 日志：access_log/error_log、log_format、字段定制、日志切分（logrotate）、日志采样

**问题：** 线上 Nginx 每秒几百条访问日志，磁盘和排查需求冲突。请设计：记录哪些字段、怎么切分、怎么采样、排障时怎么快速定位一条请求。

**期望加分项：**
- 能自定义 log_format：`log_format main '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for" $request_time $upstream_response_time $upstream_addr';`，并解释各字段（$request_time 全链路、$upstream_response_time 后端耗时、$upstream_addr 命中的上游节点）
- 能说清 access_log 与 error_log 的分工：access_log 记录每次请求（含状态码/耗时），error_log 记录 Nginx 自身错误（worker 启动、配置错误、超时上游等），级别 debug/info/notice/warn/error/crit；排障先看 error_log
- 日志切分：标准做法 logrotate（`/etc/logrotate.d/nginx` 配 daily + 30 份保留 + 切分后 reload），或 Nginx 1.17.8+ 自带的时间变量文件名（`access_log /var/log/nginx/access-$logdate.log` 需要额外配置）；坑：直接 mv 日志文件后不 reload 会导致 Nginx 继续写旧 fd，文件不增长
- 采样与降本：`access_log off;` 对无意义的静态探活/健康检查关闭；按 `map $status $log_flag` 选择性记录（只记 5xx/慢请求）；或 openresty 采样到 1/100；日志量过大先接 syslog/rsyslog 或采集到集中日志平台
- 排障定位：`$request_id` 透传到后端做全链路关联、grep 一条请求、`tail -f` 实时观察、`--follow-name` 跟 logrotate 后新文件
- 有线上经验：日志打到系统盘写满导致服务异常、logrotate 配置错误（Nginx 不 reopen 日志）的案例

**减分项：**
- log_format 字段只会背默认组合，说不出 request_time 与 upstream_response_time 的区别
- 不懂 logrotate 与 Nginx reopen 的关系，切日志后日志不增长一脸懵
- 无采样/分级意识，所有请求全量落盘
- 排障只会 grep access_log，不看 error_log
- 不知道 $request_id 全链路追踪的价值

**解答：**
日志设计四件事：格式、分级、切分、采样。格式：建议在 http 块定义 `log_format main` 至少包含：`$remote_addr`（真实客户端 IP，若走代理则用 $http_x_forwarded_for 第一跳）、`$time_local`、`$request`（方法+URI+协议）、`$status`、`$request_time`（Nginx 处理全程耗时，含排队/上游/发送）、`$upstream_response_time`（后端响应耗时）、`$upstream_addr`（本次命中的上游节点，多节点时逗号分隔）、`$request_id`（Nginx 自动生成的 32 位 hex，透传到后端即可全链路关联）。两个耗时的对比是排查核心：$request_time 大而 $upstream_response_time 小 → 瓶颈在 Nginx 侧（发送慢/等待写事件）；两者都大 → 后端慢。分级与记录策略：error_log 级别默认 error，日常排障临时调到 notice/info（能看到 "upstream timed out" 这类信息），生产不要开 debug（日志爆炸）；健康检查/探活路径 `location = /healthz { access_log off; }`；用 map 做条件日志：`map $status $is_error { ~^5 1; default 0; } access_log /var/log/nginx/error_access.log error_fmt if=$is_error;`（1.7.0+ 支持 if 参数，只记 5xx），也可 `map $request_time $slow { default 0; ~^[2-9]\. 1; }` 记慢请求。切分：logrotate daily + `create 0640 nginx adm` + `postrotate /usr/sbin/nginx -s reopen`（reopen 让 Nginx 重开日志 fd），否则 mv 旧文件后 Nginx 仍写旧 inode；1.17.8+ 可用 `access_log /var/log/nginx/access-$logdate.log` 自动按时间轮转。实践坑：日志全量落到系统盘根分区，访问量上去磁盘 100% 服务假死——日志盘独立挂载 + logrotate 30 天保留 + 超量告警；集中日志平台（ELK/Loki）是最终解，Nginx 只做本地缓冲。

**延伸考点：** $request_time 与 $upstream_response_time 差值大的根因有哪些（上游连接复用/发送慢/keepalive 等待）？logrotate 的 copytruncate 与 create+reopen 两种方案各自的坑是什么？

---

### Q11. 高可用与健康检查：keepalived+VIP、nginx upstream 健康检查、故障转移

**问题：** Nginx 是入口单点，机器挂了整个站点不可用。请设计一个低成本双机高可用方案：VIP 怎么漂移、Nginx 状态怎么探测、后端故障与 Nginx 本身故障怎么区分处理。

**期望加分项：**
- 能讲方案：keepalived（VRRP）实现 VIP 漂移——两台 Nginx 主机同网段，keepalived 配相同 VIP（如 10.0.0.100），master 挂了 backup 在 VRRP 通告超时（默认 1s 间隔，3 次）后抢占接管 VIP，客户端无感知
- 能写 keepalived 关键配置：`vrrp_instance VI_1 { state MASTER; interface eth0; virtual_router_id 51; priority 100; advert_int 1; virtual_ipaddress { 10.0.0.100/24 dev eth0 } }`，backup 节点 priority 90，authentication 认证防误接
- 健康检查有层次：① 后端节点故障 → Nginx 的 upstream max_fails/fail_timeout 自动摘除（不用切 VIP）；② Nginx 进程故障 → keepalived 脚本 `nginx_pid=$(pgrep -f 'nginx: master'); [ -z "$nginx_pid" ] && systemctl restart nginx` 或直接 killall -0 nginx 探测，失败则 `systemctl stop keepalived` 释放 VIP 让 backup 接管（keepalived 通过"降低 priority 或退出"触发漂移）
- 能讲脑裂问题：VRRP 是抢占式，网络分区时两台都认为自己是 master，同时绑定 VIP → 解决：防火墙规则绑定 VIP 检查（vrrp_skip_check_adv_addr）、仲裁（3 台 keepalived 或对端存活探测后拒绝绑定）
- 有线上经验：keepalived 检查脚本要防"脚本卡死"（加 timeout）、VIP 漂移后 Nginx upstream 里的后端无需变、session 不落本机（否则漂移丢会话）
- 知道进阶替代：LVS+keepalived 做 4 层、云厂商 SLB/ALB（自带健康检查与漂移）、DNS 多 A 记录 + 客户端重试作为最弱兜底

**减分项：**
- 只会说"keepalived"三个字，写不出配置、讲不清优先级和漂移触发条件
- 把"后端挂"和"Nginx 挂"混为一谈，不设计分层健康检查
- 不知道 keepalived 检测到 Nginx 挂了要主动释放 VIP（降低 priority / stop keepalived）
- 答不出脑裂场景及对策
- 忽略 VIP 漂移后连接/session 的影响（TCP 连接在漂移后必然断开，长连接要重建）

**解答：**
分层设计：第一层是后端故障，由 Nginx 自身 upstream 的 max_fails/fail_timeout 被动检查摘除，无需惊动 VIP；第二层是 Nginx 或整机故障，才需要 VIP 漂移。keepalived 是 VRRP 实现：master（priority 100）与 backup（priority 90）在同一网段各自运行 keepalived，周期广播 VRRP 通告（advert_int 1 即 1 秒一次），backup 连续 3 个通告周期收不到 master 心跳就认为 master 挂了，自动绑定 VIP 并回应 ARP，流量切换到 backup——客户端完全无感知，但已建立的 TCP 连接会断（TCP 状态在旧机器上），所以业务要做连接重试/重连。核心配置：master `vrrp_instance VI_1 { state MASTER; interface eth0; virtual_router_id 51; priority 100; advert_int 1; virtual_ipaddress { 10.0.0.100/24 dev eth0; } }`，backup 同 id、priority 90；`authentication { auth_type PASS; auth_pass xxx; }` 防同网段别的 VRRP 干扰。健康检查脚本是灵魂：每 2 秒检查 `nginx -v` 或 `pgrep -f 'nginx: master'`，若 Nginx 挂了先尝试 `systemctl restart nginx`，重启失败则 `systemctl stop keepalived`——keepalived 停止后 master 不再通告，backup 自动接管，VIP 随之漂移；脚本务必加超时（`timeout 5`）防止脚本挂死导致 keepalived 主进程异常。脑裂是必考坑：两台之间网络抖动（交换机故障/网线问题）时双方都收不到通告，都认为自己是 master 同时绑 VIP，流量被两台机器分流导致随机失败；对策：keepalived 开 `nopreempt`/`preempt_delay` 控制抢占行为、脚本里"对端 VIP 或网关可达性"仲裁（ping 通对端且自己还是 master 则不抢）、或上三节点仲裁。实践补充：VIP 漂移与 Nginx upstream 解耦——上游配置在两台 Nginx 上保持一致（同配置管理）；会话如无状态化则漂移无感；云环境优先用厂商 SLB（自带健康检查和故障转移），自建 keepalived 更多用于物理机房。

**延伸考点：** VRRP 通告间隔与故障检测时间的换算（advert_int 1s → 约 3s 感知），多大抖动下会发生脑裂？keepalived 脚本 vrrp_script 的 rise/fall 参数含义？

---

### Q12. 性能调优：worker_processes/worker_connections、keepalive 连接、open_file_cache、内核参数

**问题：** 压测发现 Nginx 最大只能扛 2 万并发连接，再高就出现大量超时/拒绝连接。系统层面和 Nginx 配置层面分别要调什么？请按"从 Nginx 到内核"的顺序给出调优清单。

**期望加分项：**
- 能按层给清单：Nginx 层 `worker_processes auto; worker_connections 65535; worker_rlimit_nofile 65535; use epoll;`，并算账：最大并发连接 ≈ worker_processes × worker_connections（8×65535 ≈ 52 万连接上限，2 万卡住说明瓶颈不在这个公式而在别处）
- 能定位"2 万上不去"的真实瓶颈：fd 限制（ulimit -n 必须 ≥ worker_connections）、内核 `net.core.somaxconn`（listen 队列，默认 128，过小导致 accept 队列满拒绝连接）、`net.ipv4.ip_local_port_range`（出站连接端口范围，默认 32768-60999 只有 2.8 万个端口，作为代理转发到后端时连接数会被这个数卡死）、timewait 回收
- keepalive 意识：`keepalive_timeout 65;`（客户端长连接）与 upstream 的 `keepalive 32;`（Nginx 到后端的长连接池，避免每个请求新建 TCP 连接，转发场景收益巨大，配了之后 `upstream keepalive_requests 1000`）；proxy_set_header Connection "" 让后端复用连接
- 静态/文件场景：`open_file_cache max=10000 inactive=60s; open_file_cache_valid 30s; open_file_cache_min_uses 2;`、sendfile/tcp_nopush/tcp_nodelay 组合
- 内核参数有量化：`sysctl -w net.core.somaxconn=65535; net.ipv4.tcp_max_syn_backlog=65535; net.ipv4.ip_local_port_range="1024 65535"; net.ipv4.tcp_tw_reuse=1; net.ipv4.tcp_fin_timeout=15;`（tcp_tw_reuse 只对出站连接生效）
- 用 `ss -s`、`ss -lnt | grep :80 | wc -l`、`cat /proc/sys/net/core/somaxconn` 先观测再动手，压测工具 wrk/ab 对比调优前后吞吐与延迟

**减分项：**
- 只说调 worker_connections 数值，讲不清系统 fd 与端口范围等隐性瓶颈
- 不知道 somaxconn 对高并发拒绝连接的影响
- 代理场景不配 upstream keepalive（每个请求都新建上游 TCP，QPS 上不去）
- 乱调内核参数无依据、无验证
- 压测结论没有前后对比数据

**解答：**
调优按"先观测、再分层、后验证"进行。第一步算账：最大并发连接 ≈ worker_processes × worker_connections（每连接占一个 fd，8 worker × 65535 ≈ 52 万），2 万就卡说明瓶颈一定在别处：最常见是文件描述符——进程 fd 上限 `ulimit -n` 默认 1024，必须 `worker_rlimit_nofile 65535;` 并同步调系统 limit；其次是 listen 队列——`net.core.somaxconn` 默认 128，并发超过时 accept 队列满，内核直接丢弃新连接（表现为大量 connect timeout），调大 somaxconn 并在 listen 指令里显式 `listen 80 backlog=65535;`；第三是出站端口范围——Nginx 做反代转发到后端时使用本机临时端口，`net.ipv4.ip_local_port_range` 默认 32768-60999 只有约 2.8 万个，代理场景并发转发连接数被它锁死，调到 `1024 65535` 后立即翻倍；TIME_WAIT 堆积用 `tcp_tw_reuse=1`（仅出站连接生效）+ `tcp_fin_timeout=15` 缓解。第二步配置层面：`worker_processes auto; worker_connections 65535; use epoll;`（Linux 默认已是 epoll）；`keepalive_timeout 65;` 保客户端长连接；代理场景务必配 upstream keepalive：`upstream backend { server 10.0.1.1:8080; keepalive 32; }` + `proxy_http_version 1.1; proxy_set_header Connection "";`——否则每个请求都新建到后端的 TCP 连接，QPS 会被握手成本卡住，配上后同一条上游连接可复用（keepalive_requests 控制复用上限）。第三步静态文件场景：`sendfile on; tcp_nopush on;`（合并小包）、`tcp_nodelay on;`（禁 Nagle 降延迟）、`open_file_cache max=10000 inactive=60s;` 缓存文件句柄与 stat，减少磁盘 syscall。最后验证：压测用 wrk 看 QPS 与 p99，`ss -s` 对比调优前后 TCP 状态分布，`ss -lnt` 看 backlog 丢弃（Recv-Q 满说明 somaxconn 不够）。实践坑：容器里改 sysctl 常被只读挂载拦下，需宿主侧调整或特权模式；云厂商 SLB 的 4 层健康检查与 backlog 无关，别被误导。

**延伸考点：** worker_connections 与系统 fd 上限、内存的换算关系（每连接内存开销约多少 KB）？keepalive 池里连接被后端断开（后端 keepalive_timeout 更短）时 Nginx 的表现与应对？

---

### Q13. WebSocket 与长连接：Upgrade/Connection 头、超时配置、与 HTTP/2 的关系

**问题：** 要给一个实时聊天服务配置 Nginx 反向代理 WebSocket。为什么默认配置下 WebSocket 会握手失败？需要改哪些配置？长连接挂起的连接数和超时怎么控制？

**期望加分项：**
- 能解释原理：WebSocket 通过 HTTP Upgrade 机制升级协议，Nginx 默认不转发 Upgrade/Connection 头导致后端按普通 HTTP 处理、握手失败——必须显式配置 `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";`（HTTP/1.1 下 Connection 头不被默认转发）
- 能写完整 location：`location /ws/ { proxy_pass http://ws_backend; proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade"; proxy_read_timeout 3600s; proxy_send_timeout 3600s; }`，并解释为什么两个 timeout 要调大（WebSocket 是双向长连接，read_timeout 指"两个数据帧之间的空闲超时"，默认 60s 会把空闲的 ws 连接掐断）
- 讲清超时语义：$upstream 侧空闲超时是 read/send_timeout；客户端侧 `keepalive_timeout` 不影响 ws（ws 不走 keepalive 逻辑）；业务心跳（ping/pong）周期必须小于 Nginx 超时，否则被误杀
- 连接数估算：每条 ws 长连接占用 worker 一个 fd 和内存，`worker_connections`/`worker_rlimit_nofile` 按峰值 ws 连接数 × 系数配置；`limit_conn` 可限制每 IP 的 ws 连接数防滥用
- 讲清与 HTTP/2 的关系：HTTP/2 不支持 Upgrade 机制（h2 没有 Upgrade 头），WebSocket over h2（RFC 8441）需要扩展支持——Nginx 对 `http2` 监听下的 ws 处理有限制，实践中让 ws 走独立域名/独立端口（HTTP/1.1）
- 有线上经验：ws 连接被 60s 掐断的案例、后端服务 ws 连接数打满、Nginx 连接数被 ws 占满导致普通 http 请求也失败

**减分项：**
- 不知道要显式转发 Upgrade/Connection 头
- 只加头不改 timeout，上线后被 60s 掐断
- 分不清 keepalive_timeout 与 proxy_read_timeout 对 ws 的作用
- 忽略 ws 长连接对 worker_connections 的消耗（以为还是按请求算）
- 对 ws 与 HTTP/2 的兼容问题一无所知

**解答：**
WebSocket 握手本质是 HTTP GET + Upgrade 请求，Nginx 默认只转发"标准请求头"，Upgrade 和 Connection 这两个头默认不转发（Connection 是逐跳头，Nginx 默认不传给后端），后端收到普通 GET 就按 HTTP 返回，前端 ws.onopen 永远不触发。修正配置：`location /ws/ { proxy_pass http://ws_backend; proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade"; proxy_read_timeout 3600s; proxy_send_timeout 3600s; }`。原理上：`$http_upgrade` 是客户端请求里的 Upgrade 值（websocket），把它透传给后端触发后端升级协议；Connection "upgrade" 显式声明连接升级——注意必须 `proxy_http_version 1.1`，因为 HTTP/1.0 没有 Upgrade 语义，且默认转发走 1.0 时 Connection 头会被吞。超时是第二个大坑：`proxy_read_timeout 3600s` 的语义是"上游两次响应的间隔超时"，对 ws 即"两个数据帧之间的空闲时间"，默认 60s 一到就断开——这就是"ws 连上后一分钟左右必断"的经典现象；要配合业务侧心跳：客户端每 30s 发 ping、服务端回 pong，心跳周期必须小于 Nginx 空闲超时（留 3 倍余量）。连接预算：一条 ws 连接在 Nginx 上是常驻 fd + 内存（约几 KB 到几十 KB 缓冲），1 万条 ws 连接就占 1 万 fd——`worker_rlimit_nofile` 必须开够，worker_connections 要按"峰值 ws 数 + 常规 http 并发"合计规划，否则 ws 把连接池占满后普通请求全部被拒；可配 `limit_conn_zone $binary_remote_addr zone=ws_zone:10m; limit_conn ws_zone 5;` 限制单 IP 的 ws 数防恶意挂连接。HTTP/2 的坑：h2 移除了 Upgrade 机制，RFC 8441 定义的 ws-over-h2 需要 Nginx 的 http_v2 扩展支持（OpenResty/较新版本才有），生产标准做法是 ws 走独立 server（`listen 8443 ssl http2;` 之外另配 `listen 8444 ssl;` 专给 ws）或用独立子域名，避免 h2 与 ws 混用。

**延伸考点：** ws 的 ping/pong 由客户端发还是服务端发更稳妥，Nginx 层能否自动响应 ws 心跳？Nginx 在 ws 升级后对 TCP 层（如读写缓冲、半开连接探测）如何管理空闲连接？

---

### Q14. 安全加固：隐藏版本号、禁用不需要的方法、目录访问控制、防目录遍历、防 DDoS 基础

**问题：** 站点被安全扫描发现：响应头暴露 Nginx 版本、OPTIONS/TRACE 方法可访问、`/backup`、`/.git` 目录能直接浏览。请给出完整加固配置。

**期望加分项：**
- 能逐项加固：`server_tokens off;`（隐藏版本号，错误页/Server 头只显示 nginx）、`if ($request_method !~ ^(GET|HEAD|POST)$) { return 405; }` 或用 `limit_except GET POST { deny all; }` 禁 TRACE/OPTIONS/DELETE 等非必要方法
- 目录保护：`location ~ ^/(backup|\.git|\.svn|conf|logs)/ { deny all; }` 或 `location ^~ /.git/ { deny all; }`；`location ~ /\. { deny all; }`（拒绝所有点开头文件）；目录浏览 `autoindex off;`（默认关闭但要显式确认）
- 防目录遍历：Nginx 默认已防（不像 Apache 有 Options Indexes），但要注意 alias 拼接错误（`alias /data/www/files/` 少写尾部 / 导致可以穿越到上级目录的经典漏洞：`location /files { alias /data/www/files/; }` 请求 /files../ 时路径拼接问题）；`merge_slashes off` 的风险点
- 防 DDoS 基础：limit_req/limit_conn 限流、`client_body_timeout`/`client_header_timeout` 防慢速攻击（slowloris，把 header/body 读超时调小如 10s）、`client_max_body_size` 防大包、keepalive 与连接数限制、`limit_req_zone` 按 IP 限速
- 安全响应头：`add_header X-Content-Type-Options "nosniff"; add_header X-Frame-Options "SAMEORIGIN"; add_header Referrer-Policy "strict-origin-when-cross-origin";` 与 CSP（Content-Security-Policy）基础配置，且知道 add_header 在 location 继承时的坑（子 location 有 add_header 会覆盖父级全部 add_header）
- 有线上佐证：扫描报告问题清单与修复前后对比、安全头遗漏导致的问题、被 /backup 泄露文件的事故

**减分项：**
- 只会 server_tokens off，其它一问三不知
- 不会用 limit_except，或不知道 TRACE 的危害（反射型 XSS 向量）
- 说不清 alias 尾部 / 与目录穿越的关联
- 防 DDoS 只想到限流，不知道慢速攻击（slowloris）要调 header/body 超时
- add_header 继承规则讲不清（子 location 覆盖父级）

**解答：**
按"信息泄露 → 方法收敛 → 目录/文件保护 → 攻击面限流 → 安全响应头"五层做。① 隐藏版本：`server_tokens off;`（Server 头只留 "nginx"），错误页同时生效。② 方法收敛：`limit_except GET POST HEAD { deny all; }` 是正规做法（放行白名单方法，其余直接 403），比 if 判断更简洁——TRACE 必须禁（反射型 XSS/跨站追踪向量），OPTIONS 看业务是否需要；CDN/负载均衡场景注意有些探活用 HEAD，别误伤。③ 目录保护：`location ~ /\. { deny all; }` 一刀切拒绝所有点开头路径（.git/.env/.svn）；关键目录显式 `location ^~ /backup { deny all; }`；`autoindex off;` 防目录浏览。④ 目录穿越：Nginx 本身无 Apache 的 Options 机制，漏洞多来自 alias 拼接——`location /files { alias /data/www/files/; }` 若 alias 尾部漏 `/`，请求 `/files../secret` 会拼出 `/data/www/files/../secret` 越权读到上级目录，alias 尾部斜杠必须严格；静态资源路径做 `normalize` 前确保 `$uri` 已被规范化（默认 Nginx 已归一化 ../，但 alias 场景仍要警惕）。⑤ 攻击面与 DDoS 基础：慢速攻击（slowloris）用极慢速度发 header 占满连接，对策是把 `client_header_timeout 10s; client_body_timeout 10s;` 调小（默认 60s）；`client_max_body_size 10m;` 防超大请求体；`limit_req`/`limit_conn` 限速限连；必要时 `location` 层关闭无意义路径的访问。⑥ 安全头：`add_header X-Content-Type-Options "nosniff" always;`、`X-Frame-Options "SAMEORIGIN"`、`Referrer-Policy`、CSP；最大坑是 add_header 的继承语义：子 location 一旦出现 add_header，父级（http/server）的所有 add_header 全部失效，所以要么全部在 location 内写全，要么确认嵌套关系。落地流程：`nginx -t` 检查 → reload → 用 `curl -I` 逐个验证 → 复扫对比。实践补充：`X-Frame-Options` 在新浏览器已逐步被 CSP frame-ancestors 取代，两者可并存。

**延伸考点：** `merge_slashes off` 为什么是安全风险（`//etc/passwd` 类路径绕过）？CSP 配置里 frame-ancestors 与 X-Frame-Options 的关系与共存写法？

---

### Q15. 转发头与真实 IP：X-Forwarded-For/Proto、proxy_set_header、proxy_protocol、源 IP 恢复

**问题：** 架构是 客户端 → 云负载均衡（4 层）→ Nginx → 后端服务。现在后端日志里所有客户端 IP 都是负载均衡的内网 IP，限流、封禁、审计全失效。你怎么恢复真实客户端 IP？

**期望加分项：**
- 能讲清问题本质：每跳代理都会改写/追加 X-Forwarded-For，4 层 LB（如 AWS NLB、阿里云 SLB 四层）不修改 HTTP 头，后端看到的是 Nginx 的 IP；且 XFF 可被客户端伪造，直接取 XFF 不可信
- 能讲三层方案并按链路对号入座：① 7 层 LB 场景——LB 追加 XFF，Nginx 用 `set_real_ip_from <lb_cidr>; real_ip_header X-Forwarded-For;`（ngx_http_realip_module）把 $remote_addr 替换为 XFF 第一跳（客户端 IP），此时 access_log 的 $remote_addr 和 limit_req 的 $binary_remote_addr 自动生效；② 4 层 LB 场景——LB 配 PROXY protocol（每连接头带真实 IP），Nginx `listen 80 proxy_protocol;` + `set_real_ip_from <lb_cidr>; real_ip_header proxy_protocol;`；③ 直连公网场景——Nginx 直接拿 $remote_addr 就是客户端 IP，别乱配 realip
- 能讲 proxy_protocol 细节：v1 文本格式 `PROXY TCP4 <src_ip> <dst_ip> <src_port> <dst_port>\r\n`，v2 二进制；只信任来源（set_real_ip_from 限定 LB 网段），否则任何人都能伪造该头伪装任意 IP
- 转发给后端时注意：`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`（追加而非覆盖，保留链路）；若用 realip 模块，$remote_addr 已变真实 IP，直接 `proxy_set_header X-Real-IP $remote_addr;`
- 有排障案例：配了 realip 但 set_real_ip_from 网段写错导致 $remote_addr 变成"意外的伪造 IP"或 LB 探测失败；limit_req 用 $remote_addr 还是 $binary_remote_addr 的选择
- 进阶：OpenResty 下用 `http_x_forwarded_for` 解析取第一个非私网地址；nginx-ingress 里 externalTrafficPolicy: Local 保留源 IP 的方案对比

**减分项：**
- 直接"取 XFF 第一个值"当客户端 IP，不知道 XFF 可伪造
- 分不清 4 层与 7 层 LB 下各需要哪种方案（proxy_protocol vs XFF）
- 不知道 set_real_ip_from 必须限定信任网段
- 配了 realip 后限流/日志不生效，无法定位原因
- 不了解 proxy_protocol v1 格式

**解答：**
恢复真实 IP 的正确姿势是"按链路选方案 + 只信任上一跳"：任何代理都默认追加 XFF（把自己的 $remote_addr 追加到末尾），但 XFF 的起始部分完全由客户端控制——客户端可以直接发 `X-Forwarded-For: 1.2.3.4`，所以"取第一个值"是安全漏洞。方案分两种链路：① 链路里有 7 层 LB（会写 XFF）：在 Nginx 上启用 ngx_http_realip_module：`set_real_ip_from 10.0.0.0/8; real_ip_header X-Forwarded-For;`——只把"来自 LB 网段的连接"的 $remote_addr 替换成 XFF 中的客户端 IP（取第一个非本机 IP），替换后 access_log 的 $remote_addr、limit_req/limit_conn 的 key 全部自动变成真实 IP，后端配合 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` 保留完整链路。② 4 层 LB（TCP 转发，不碰 HTTP 头）：用 PROXY protocol——LB 在 TCP 连接建立后先发一行 `PROXY TCP4 <client_ip> <lb_ip> <client_port> <lb_port>\r\n` 再发 HTTP 数据，Nginx 侧 `listen 80 proxy_protocol;` 解析该头 + `set_real_ip_from <lb_cidr>; real_ip_header proxy_protocol;` 恢复源 IP；PROXY 头同样可伪造，set_real_ip_from 是唯一信任边界（只对 LB 网段来的连接启用解析）。③ 若 Nginx 直接对公网（CDN 回源到 Nginx 的场景注意区分）：CDN 回源时 XFF 是 CDN 追加的，可信任 CDN 的 XFF，同样用 set_real_ip_from 限定 CDN 回源 IP 段。常见坑：set_real_ip_from 网段写错（LB 加了新网段没更新）→ $remote_addr 变回 LB IP，限流全失效；realip 与 proxy_protocol 混用顺序（proxy_protocol 必须配在 listen 上）；后端如果还要看原始客户端 IP，务必把 XFF 按 `$proxy_add_x_forwarded_for` 追加而不是 `proxy_set_header X-Forwarded-For $remote_addr` 覆盖（后者丢链路）；容器/K8s 场景下 Ingress 要保留源 IP 需 externalTrafficPolicy: Local，否则走 SNAT 后源 IP 全变节点 IP，这是 k8s 里最常见的"IP 全不对"根因。

**延伸考点：** realip 替换后 `$remote_addr`、`$http_x_forwarded_for`、`$proxy_add_x_forwarded_for` 三者分别是什么值？PROXY protocol 在 TCP 长连接（ws）下的处理与普通 HTTP 有何不同？

---

### Q16. 大文件与请求限制：client_max_body_size、请求超时（proxy_read_timeout）、上传场景

**问题：** 业务要支持上传最大 500MB 的文件，同时下载 1GB 的安装包。上传报 413、下载到一半断流、上传大文件时 Nginx 内存暴涨。分别怎么配置和处理？

**期望加分项：**
- 能写配置：上传 `client_max_body_size 500m;`（默认 1m，超限返回 413 Request Entity Too Large），配套 `client_body_timeout 60s;`（读请求体超时）、`proxy_read_timeout 300s;`（后端处理上传超时）、`proxy_send_timeout`（转发大 body 到后端）；下载场景 `proxy_buffering off;` 或调大 `proxy_buffers`/`proxy_max_temp_file_size`，避免大响应在 Nginx 缓冲导致内存/磁盘双爆
- 讲清 413 的语义与定位：413 是 Nginx 在读取请求体阶段直接拒绝（body 超过限制），配置在 server/location 均可（location 级更精细，如只对 /upload 放大）；上传报 413 时看 error_log 的 "client intended to send too large body"
- 讲清缓冲机制：上传时 Nginx 默认 `client_body_buffer_size 8k/16k`，超过后落临时文件（`client_body_temp_path`），配置 `client_body_buffer_size 1m` 可以减少小文件落盘，但大文件无论怎样都要走临时文件；下载时 `proxy_buffering off` 直通（后端流式响应直接给客户端，内存占用小、支持断点续传），`proxy_buffering on` 时响应先缓冲到内存（proxy_buffers 8 4k/8k）超过后落 temp（proxy_max_temp_file_size 默认 1GB 磁盘缓冲）
- 超时与连接关系：`proxy_read_timeout`（两次读操作间隔）、`proxy_send_timeout`（两次写操作间隔）——大文件慢传场景这两个超时必须是"空闲间隔"而非总时长，Nginx 是按间隔算的，所以大文件慢慢传不会被掐（只要间隔内有数据流动）；被掐断要查 `upstream timed out` 日志
- 配套调优：`proxy_request_buffering off`（OpenResty/1.7.11+ 上游流式读，减少 Nginx 侧缓冲整包）；`sendfile` 与 `aio`/`directio`（大文件用 `aio on; directio 4m;` 直接 IO 绕过 page cache 防缓存污染）；`limit_rate` 限速；上传校验：`client_max_body_size` 与业务方超时（前端、后端网关、CDN）要一致对齐，否则 Nginx 没卡后端卡
- 有线上案例：413 排查、上传 500MB 时磁盘打满（temp 目录）、大文件下载导致 Nginx 内存翻倍

**减分项：**
- 只会调 client_max_body_size，不知道它只限制请求体，且不限制"后端响应体"
- 分不清 client_body_timeout/proxy_read_timeout/proxy_send_timeout 各自管哪段
- 不知道大响应默认要缓冲，`proxy_buffering off` 的适用与代价说不清
- 忽略了 Nginx 临时文件路径的空间（默认在 / 分区）
- 无大文件场景的实际调优验证

**解答：**
分上传/下载两条链路分别处理。上传链路：`client_max_body_size 500m;` 是必须的（默认 1m，超限直接 413），建议放在 `location /upload/` 内精细控制而不是全局放开；413 定位看 error_log："client intended to send too large body" 就是它；配套超时——`client_body_timeout 60s`（读请求体的间隔超时）、`proxy_read_timeout 300s`（后端处理上传期间的读间隔）、`proxy_send_timeout 300s`（Nginx 转发 body 给后端的写间隔），注意这三个都是"空闲间隔"语义：只要数据在流动就不会被掐，大文件慢传（比如 500MB 走 2Mbps）不会被固定时长限制误杀。上传缓冲：`client_body_buffer_size` 默认 8k/16k，超过即写临时文件（`client_body_temp_path` 默认在 Nginx 编译路径下，很可能在系统盘！），大批量上传必须把 temp_path 指到大分区磁盘并监控空间；想要 Nginx 不缓冲整包转发用 `proxy_request_buffering off`（此时 Nginx 边收边转，减少磁盘 IO，但上游必须支持流式读，且会慢速客户端直接拖住上游连接）。下载链路：`proxy_buffering on`（默认）时大响应先攒到内存（`proxy_buffers 16 4k` 等）超出落 `proxy_temp_path`，1GB 文件会先写满 temp 磁盘再发给客户端，延迟高且磁盘双写；大文件直通场景 `proxy_buffering off;` 让上游数据直接流向客户端（内存占用固定、首字节更快），配合 `proxy_max_temp_file_size`（缓冲仍开时限制落盘量）。IO 层：大文件加 `aio on; directio 4m;` 用直接 IO 绕过 page cache（避免大文件污染缓存挤掉热数据），`output_buffers 2 1m;` 控制发送缓冲。实践坑：前端/后端应用/CDN 各层 max body 与超时必须对齐（如后端 Tomcat maxPostSize 默认 2MB，Nginx 放开了后端还拒）；`limit_rate 5m;` 可给下载限速防带宽被单连接打满；上传量大时监控 temp 磁盘与 inode，日志里大量 413/504 要能快速区分是 Nginx 拒的（413）还是后端没处理完（504 与 proxy_read_timeout）。

**延伸考点：** proxy_request_buffering off 与慢速客户端攻击（slowloris 用 body）的关系——为什么关掉缓冲会放大慢速攻击风险？断点续传（Range）在 proxy_buffering off 与 CDN 场景下的配合点？

---

### Q17. 灰度与分流：按 header/cookie/IP 分流到不同 upstream、蓝绿入口、权重灰度

**问题：** 新版本后端要上线，先在 10% 流量上验证，且希望灰度用户（带指定 header）稳定打到新版本，失败能一键回滚。请用 Nginx 实现灰度分流方案。

**期望加分项：**
- 能写 map 分流：http 块 `map $http_x_grayscale $upstream_pool { default blue; grayscale green; }` 或按 header 值 + 权重：`split_clients "$remote_addr$http_user_agent" $pool { 10% green; * blue; }`，location 里 `proxy_pass http://$pool;`（upstream 名字用变量时注意 proxy_pass 不能带 URI）
- 能讲三种灰度维度及取舍：header 灰度（白名单用户精确可控、适合内测）、cookie 灰度（`map $cookie_grayscale ...`，适合按用户灰度、可前端写 cookie 决定是否进灰）、IP 灰度（`geo $whitelist` + map，适合办公网内测、不适合 NAT 大网段）；按权重随机（split_clients，适合无差别的比例流量）
- 能讲 split_clients 原理：对 key（如 $remote_addr）做 MurmurHash 后按百分比区间映射，同一 key 稳定落在同一区间 → 同一用户始终命中同一池（灰度一致性）；`split_clients "${remote_addr}${http_user_agent}" $pool { 10% green; * blue; }`
- 有回滚/蓝绿意识：蓝绿部署是两个完整环境 + 入口切换（Nginx upstream 指向全切），灰度是渐进式；回滚 = 一键把 upstream 指回 blue（改配置 reload 或变量开关），要求灰度前"新旧环境并存且数据兼容（DB 回滚/双写）"
- 有观测配套：`add_header X-Pool $upstream_pool;` 或日志记录命中池，配合监控对比新版本错误率/延迟，决策是否放量；放量节奏：10% → 30% → 50% → 100%，每档观察 ≥ 一个业务周期
- 知道进阶实现：openresty 按业务属性动态分流、nginx-ingress canary annotation（nginx.ingress.kubernetes.io/canary）、APISIX 等网关的灰度插件

**减分项：**
- 只会"改 upstream 权重"一种方式，讲不清多种灰度维度
- map/split_clients 语法写错（如 split_clients 必须 http 上下文、proxy_pass 带变量不能带 URI）
- 灰度一致性没意识（random 转发导致同一用户在新旧版本间跳变）
- 没有回滚预案（灰度环境数据不兼容，回滚要清数据）
- 不配观测（不知道灰度流量打到了哪个池）

**解答：**
先选维度再选机制，机制上 Nginx 通用做法是"map/split_clients 生成 upstream 变量 + proxy_pass 引用"。① 按 header 精确灰度：`map $http_x_canary $pool { default blue; on green; }` 然后 `proxy_pass http://$pool;`——业务/网关把指定用户带上 `X-Canary: on` 头即进新版本，最适合内测白名单；② 按 cookie 灰度：`map $cookie_canary $pool { canary green; default blue; }`——适合按用户维度放量，前端可写 cookie 决定用户是否体验新功能；③ 按 IP 灰度：`geo $canary_ip { default 0; 10.0.0.0/24 1; }` + map 组合，办公网内测常用；④ 按权重随机：`split_clients "${remote_addr}${http_user_agent}" $pool { 10% green; * blue; }`——split_clients 对 key 做一致性哈希，同一用户始终落在同一区间（灰度稳定性），10% 流量进 green，这是"无差别比例放量"的标准答案。注意 proxy_pass 的坑：upstream 名用变量后 `proxy_pass http://$pool;` 不能带 URI（带 URI 时 Nginx 无法解析变量+URI 组合，要么 502 要么直接报配置错误），转发路径需要精确控制时用 rewrite 配合。放量与回滚节奏：10% → 30% → 50% → 100%，每档观察至少一个完整业务周期，对比 green/blue 的 p99 延迟、错误率（5xx 比例）、核心指标（下单成功率/支付回调）；观测手段：`add_header X-Pool $pool;` 让响应头直接标记命中池，日志里记录 $pool 便于对账。回滚 = 入口切换：把 map 里 green 全改回 blue reload（秒级、无连接中断）即完成；前提是灰度前做好数据兼容——新旧版本共享 DB 且新版本变更可逆（表结构/缓存 key 做好兼容），否则"回滚"会变成数据事故。进阶：K8s 场景直接用 nginx-ingress 的 canary annotation（`nginx.ingress.kubernetes.io/canary: "true"` + `canary-weight: "10"`），其原理就是动态生成上面的 map/split_clients；大规模动态灰度（改配置不 reload）要用 OpenResty 或 API 网关。

**延伸考点：** split_clients 与"random 轮询权重"的灰度一致性差异在哪，为什么随机转发会导致同一用户新老版本间抖动？灰度环境的数据兼容如何设计（DB 变更回滚、缓存双写）？

---

### Q18. 排障：502/504/499 各自含义与排查、worker 连接耗尽、配置不生效、日志定位

**问题：** 线上突发大面积 502 和 504，同时伴随 499，压测停止后恢复。请按"状态码 → 定位方向 → 工具命令"给出完整排查思路。

**期望加分项：**
- 能精确区分三码：502 Bad Gateway（Nginx 连不上上游/上游 TCP 层失败或返回非法响应）、504 Gateway Timeout（上游连接成功但响应超时，触发 proxy_read_timeout）、499 Client Closed Request（客户端在 Nginx 拿到上游响应前主动断开，通常是客户端等不及超时了或业务取消，Nginx 主动记 499 且不再等待上游）
- 排查 502：`error_log` 定位（"connect() failed (111: Connection refused)" 是端口没监听、"connect() timed out"、"no live upstreams" 是 upstream 全被熔断）、`netstat/ss -lntp` 看后端端口、`curl -v http://backend:8080/healthz` 直接打后端确认服务本身是否活着、`nginx -t` 排除配置错误（upstream 写错端口/域名解析失败）
- 排查 504：看日志 "upstream timed out (110: Connection timed out) while reading response header"——是读响应超时；先确认后端是否真的慢（upstream_response_time 打点），再决定调 proxy_read_timeout 还是优化后端；注意"慢接口 + 短超时"配错导致正常慢请求全 504
- 排查 499：499 是 Nginx 主动行为，看是不是客户端超时太短（前端 axios 默认、LB 超时 30s/60s、CDN 回源超时）——常发生在"后端处理 2s，客户端只等 1s"；也见于爬虫取消、用户刷新；499 比例高的根因通常要改客户端超时或优化接口
- 连接耗尽场景：`ss -s` 看 TIME_WAIT/ESTAB 数量、`worker_connections` 是否打满（`ss -lnt | wc -l` 对比）、后端 fd 是否爆（应用层报 too many open files）、Nginx 日志 "accept() failed (24: Too many open files)"
- 配置不生效排查：`nginx -t` 只查语法不查语义、`nginx -T` 导出最终合并配置（include 展开）核对、`nginx -s reload` 后是否真的生效（`ps -ef | grep nginx` 看 worker 是否替换、`nginx -s reload` 失败会回滚）、缓存配置改了没加 `proxy_cache_path` 层面校验

**减分项：**
- 502/504/499 三码含义讲混（尤其把 499 说成服务端错误）
- 排障只停留在"看状态码"不落命令，或只会 curl 不会看 error_log
- 不看 access_log 里的 upstream_response_time/upstream_status 就下结论
- 不知道 502 里 no live upstreams 与 connection refused 的差异
- 配置不生效只知道 reload，不知道 -T 核对与 worker 替换确认

**解答：**
按状态码分流定位。502 是"连不上/连上但非法"：error_log 给方向——`connect() failed (111: Connection refused)` → 后端端口没监听（服务挂了/端口配置错）；`connect() failed (110: Connection timed out)` → 后端有端口但网络不通（防火墙/后端过载不响应）；`no live upstreams while connecting to upstream` → upstream 里的节点全部被 max_fails 熔断了（后端持续故障）；还有一类是 upstream 域名解析失败。动作：`ss -lntp | grep 8080` 看端口、`curl -v` 直连后端、确认后重启后端或修配置。504 是"连上了但响应超时"：error_log 特征 `upstream timed out (110: Connection timed out) while reading response header`，说明等待后端第一个响应字节超了 proxy_read_timeout；先看 access_log 的 `$upstream_response_time` 判断后端真实耗时——若后端确实 3s+ 而超时 60s 还 504，多半是后端排队/GC 慢；若大部分请求正常仅个别慢接口 504，给该 location 单独调大 proxy_read_timeout 或优化接口；也要警惕"后端在处理但 Nginx 等不及"与"后端压根没开始处理"的区别（配合后端日志核对）。499 最容易被误读：它不是 Nginx/后端错误，而是"客户端先断开了"，Nginx 记录 499 后不再等上游响应；常见根因是客户端（前端/Axios/LB/CDN）超时设置短于接口实际耗时——排查先看 access_log 该请求的 $upstream_response_time：如果上游已经返回（比如 3s）而客户端 2s 就断，改客户端超时；如果上游还在处理就断了，优化后端或分流。连接耗尽场景：`ss -s` 看连接状态分布，TIME_WAIT 大量堆积是短连接风暴（配 keepalive），`accept() failed (24: Too many open files)` 是 fd 上限（worker_rlimit_nofile + ulimit），ESTAB 打满 worker_connections 则算账扩容。配置不生效四步：`nginx -t`（语法）→ `nginx -T`（看 include 展开后的最终生效配置，排除"改错文件/被覆盖"）→ `nginx -s reload` → `ps -ef | grep nginx` 确认 worker 已换成新进程（reload 失败时旧 worker 仍在，配置未生效且无报错）。万能辅助：临时开 `error_log /var/log/nginx/debug.log debug;`（低峰期）看完整请求处理流程，用完即关。

**延伸考点：** 502 里 "connect() failed" 与 "no live upstreams" 的排查方向差异？压测后 499 暴增且 upstream_response_time 很短，说明瓶颈在哪（Nginx 层连接排队）？

---

### Q19. 配置管理：include 组织、nginx -t 检查、平滑重载（reload）与热升级、配置审计

**问题：** 现在有 50 个站点、上千行配置，还要支持多团队自助提交配置。请设计配置的组织结构、变更流程（含检查与回滚）和热升级方案，保证改配置不炸线上。

**期望加分项：**
- 能讲 include 分层：`/etc/nginx/nginx.conf`（主配置，仅保留 http 核心与全局项）+ `include /etc/nginx/conf.d/*.conf;`（server 级）+ `include /etc/nginx/snippets/*.conf;`（可复用片段，如 ssl 模板、安全头模板、gzip 模板）+ 按站点目录 `include /etc/nginx/sites-enabled/*;`（符号链接指向 sites-available，启停站点只动链接）
- 变更流程：改完必 `nginx -t`（语法校验）→ `nginx -T` 查看合并结果 → 发布用 `nginx -s reload`；reload 语义讲清：master 重读配置、fork 新 worker、旧 worker 处理完存量请求后优雅退出（处理中的请求不受影响，长连接会等 keepalive_timeout 或处理完）——这就是"平滑"的机制；注意 `nginx -s reload` 失败时 Nginx 继续用旧配置运行
- 热升级（不中断升级二进制）：`kill -USR2 <master_pid>` 启动新二进制 → `kill -WINCH` 停旧 worker → `kill -QUIT` 结束旧 master；升级失败回滚用 `kill -HUP` 重读旧配置；知道系统包管理升级的坑（RPM 直接替换二进制会断连接，必须用上述流程或官方动态模块）
- 配置审计与灰度：配置版本化（git 管理 + 提交前 CI 跑 nginx -t）、上线前在预发环境验证、按集群分批 reload（先边缘节点观察再全量）、配置 diff 审查（变更单与回滚版本绑定）、定期 `nginx -T` 对比漂移
- 有线上案例：include 顺序覆盖坑（同名指令后者生效）、改错配置文件 reload 后大面积 502（语法对但语义错：upstream 名字写错导致 no live upstreams）、无备份回滚困难

**减分项：**
- 配置全写一个文件不拆分，或不知道 include 展开顺序的覆盖规则
- reload 语义讲不清（以为 reload=重启、连接全断，或不知道失败回滚）
- 不知道热升级信号流程（USR2/WINCH/QUIT）
- 无变更流程（不 nginx -t、无备份、无回滚、无分批）
- 审计缺失：配置与代码分离、版本化、审批都没有

**解答：**
配置组织：主配置 nginx.conf 只放全局（user/worker_processes/events/http 公共段如 log_format、gzip、proxy_cache_path），server 全部外置：`include /etc/nginx/conf.d/*.conf;`（每个站点一个文件，文件名即 server_name，如 `www.example.com.conf`）；可复用片段放 snippets（`ssl-params.conf` 统一 TLS 配置、`security-headers.conf` 统一安全头——配合 add_header 继承规则整体引入，避免子 location 覆盖问题）；多站点场景用 sites-available/sites-enabled + 符号链接（启停站点=加删链接，比改配置更安全）。变更流程必须工程化：① 改配置（git 分支）→ ② `nginx -t` 语法校验（有错立即发现）→ ③ `nginx -T` 人工审阅 include 合并后的最终结果（防改错文件、防被后 include 覆盖——Nginx 同指令后出现者生效，upstream 名重复定义会报错，location 同名继承是"合并"而非覆盖，最容易误判）→ ④ 分批 reload：先在 1 台边缘/灰度 Nginx 上 `nginx -s reload` 观察 10 分钟无异常再全量 → ⑤ 备份上一版配置并保留回滚文档（回滚=恢复旧文件 + reload，秒级）。reload 机制讲透：master 收到信号后重新解析配置，解析成功则 fork 一组新 worker 接手新连接，旧 worker 完成手中存量请求后退出——整个过程中已建立的连接不断（这就是"平滑"），但 reload 本身不保证长连接立即切换（长连接会等到 keepalive 超时后由新 worker 接管）；若新配置解析失败，master 直接拒绝 reload 并沿用旧 worker（日志报 "test failed"），线上不中断。热升级二进制：`kill -USR2 <pid>`（master fork 出新二进制进程）→ 验证新 worker 正常 → `kill -WINCH <pid>`（旧 worker 优雅退出）→ `kill -QUIT <pid>`（旧 master 退出）；升级中出问题则 `kill -HUP <pid>` 用旧配置拉起旧版本。审计：配置入 git、提交 CI 里自动 `nginx -t`、变更与发布单绑定、定期用 `nginx -T` 对比生产与版本库漂移，50 个站点场景这是底线。

**延伸考点：** reload 时正在处理中的大文件下载/WebSocket 长连接会被强制断开吗（旧 worker 优雅退出的边界）？include 同名指令与同名 server_name 的解析冲突规则？

---

### Q20. Nginx 在云原生中的角色：Ingress Controller（nginx-ingress）工作原理、与 Service 的关系、落地实践

**问题：** 公司 K8s 集群里要暴露 30 个微服务给外部访问，你们选了 nginx-ingress。请说明它和"裸 Nginx"的关系、如何把流量路由到 Service，以及多集群/灰度/证书管理的落地经验。

**期望加分项：**
- 能讲清 Ingress 本质：nginx-ingress 是"控制器 + Nginx 二进制"的组合——控制器 watch k8s 的 Ingress/Service/Endpoint/Secret 等对象，把它们翻译成 Nginx 配置，再生成配置文件并 `nginx -s reload`；对外表现就是一个跑在集群里的 Nginx Pod（通常是 Deployment 多副本 + NodePort/LB 暴露）
- 讲清路由链路：客户端 → 集群入口（LB/NodePort）→ Ingress Controller Pod（Nginx）→ 按 Ingress 规则（host + path）选 Service → Service 做负载均衡 → 转发到后端 Pod IP（直接转发到 Endpoint 而非 ClusterIP 是默认行为，避免二次转发）
- 知道两种部署模式：Deployment + Service 类型（NodePort/LoadBalancer/直连 HostNetwork）的选择与 externalTrafficPolicy: Local 对源 IP 保留的影响；DaemonSet + HostNetwork 模式适合大流量（每节点一个 Nginx 实例）
- 能讲高级特性与落地：annotation 灰度（canary/weight 基于 split_clients 原理）、TLS 自动签发（cert-manager + Ingress 的 tls secret）、限流（nginx.ingress.kubernetes.io/limit-rps）、重写（rewrite-target annotation 与 $1 捕获组的坑）、自定义 upstream 与金丝雀、`nginx.ingress.kubernetes.io/proxy-read-timeout` 等超时 annotation
- 多集群/规模经验：Ingress Controller 的副本数与流量预估（一个 Pod 支撑几万 QPS）、给 Ingress Controller 单独资源配额（CPU/内存/大磁盘放日志）、配置热加载风暴问题（controller 每次变更都 reload，变更频繁时用 `nginx.ingress.kubernetes.io/reload` 或 OpenResty 变体避免全量 reload）
- 对比其他方案：traefik（动态路由强）、istio ingress gateway（服务网格场景）、APISIX/云厂商 ALB Ingress；知道什么时候该用哪种
- 排障经验：503 与 upstream 找不到（Pod 未就绪/网络策略）、annotation 拼写错误静默不生效（看 controller 日志与 events）

**减分项：**
- 把 Ingress Controller 当"云上装的 Nginx"，讲不清 controller 与 Nginx 的关系
- 路由链路讲错（经过 ClusterIP 二次转发或直接说"Ingress 就是 Service"）
- 不知道 annotation 与裸 Nginx 配置的对应关系
- 没有多副本/资源配额/灰度/证书的落地细节
- 只会用不会排障（annotation 不生效不知道看 events）

**解答：**
先拆概念：Ingress（k8s API 对象，声明式路由规则）与 Ingress Controller（实现者，这里即 nginx-ingress）是两回事。nginx-ingress 的 controller 进程 watch k8s API：任何 Ingress/Service/Endpoint/Secret 变更都会触发它重新生成 Nginx 配置（模板渲染）并 reload——所以"改 Ingress 规则 = 改 Nginx 配置并平滑 reload"，reload 机制与裸 Nginx 完全一致。路由链路：外部请求 → 集群入口（LoadBalancer/NodePort/HostNetwork）→ Ingress Controller Pod（Nginx 监听 80/443）→ 按 Ingress 的 host + path 匹配 → 转发到后端 Pod 的 IP:Port（nginx-ingress 默认直接转发到 Endpoint，即 Pod IP，绕开 ClusterIP 的 kube-proxy 二次转发，性能更好且保留更多信息）→ Service 只负责"选择 Endpoint 集合"这件事。部署模式：Deployment + Service(LoadBalancer) 最常见；大流量场景用 DaemonSet + HostNetwork（每节点一个实例，宿主机直接监听 80/443，少一跳）；源 IP 保留靠 `externalTrafficPolicy: Local`（否则 SNAT 后源 IP 变节点 IP，限流/审计全废——这是 k8s 里"真实 IP 问题"的源头，与裸 Nginx 的 realip/proxy_protocol 配合解决）。落地实践要点：① 灰度——`nginx.ingress.kubernetes.io/canary: "true"` + `canary-weight: "10"`，controller 内部正是用 Nginx 的 split_clients/map 实现按权重/header/cookie 分流；② 证书——cert-manager 签发证书写进 Ingress 的 tls secret，controller 自动挂载到 Nginx 的 ssl_certificate；③ 限流/超时——annotation 如 `limit-rps`（对应 limit_req）、`proxy-read-timeout`（对应 proxy_read_timeout）、`rewrite-target`（对应 rewrite，注意 `rewrite-target: /$2` 与捕获组的配合是经典坑）；④ 规模——Controller 副本数按流量预估（单 Pod 数万 QPS 量级），单独配 resource 限额与日志盘，变更频繁场景关注 reload 风暴（每次 Ingress 变更都 reload，可用 `--sync-period` 合并或换 OpenResty 变体）。排障：路由不生效先 `kubectl get ingress` + `kubectl describe ingress` 看 events，annotation 拼错会静默忽略；后端 503 排查 Pod 就绪探针与 Endpoint 是否正常生成。选型：轻量动态路由选 traefik，服务网格选 istio gateway，大规模网关治理（限流/鉴权/灰度插件）选 APISIX 或云厂商 ALB Ingress。

**延伸考点：** externalTrafficPolicy: Local 与 Cluster 对源 IP、负载均衡、连接复用的影响差异？Ingress 变更频繁时 reload 风暴如何量化与规避（每变更一次 reload 的代价、OpenResty/动态 upstream 方案）？

---

### Q21. 一次 502/504 大面积故障的入口层排查复盘，怎么定位根因？

**问题：** 某天下午线上大面积 502/504，首页都打不开，入口 Nginx 错误率飙到 60%。你负责入口层排查。请复盘这次事故的排查路径：502 与 504 各指向什么问题、upstream/超时/worker 状态/连接数四个层面分别怎么查，以及事后怎么沉淀防护？

**期望加分项：**
- 先定性再圈范围：502 = Nginx 从 upstream 拿不到有效响应（后端挂/拒连/空响应），504 = 请求发出但超时窗口内没等到响应（后端慢/排队）；先分 502 还是 504，再看全局还是单个 upstream、是否与某次后端发布窗口重合（发布引发 502 是最高频剧本）
- upstream 检查：error.log 的明确报错（connect() failed / upstream timed out / no live upstreams）、后端端口连通性与进程状态、`max_fails`/`fail_timeout` 被动宕机制（后端局部异常会被标记 down 导致全部 502）；`curl -v` 本机探活 + 后端日志交叉验证
- 超时检查：proxy_connect_timeout/proxy_read_timeout/proxy_send_timeout 的合理值；强调"盲调大超时是饮鸩止渴"——连接挂起越久、worker 连接越早耗尽、雪崩越猛，正确动作是后端侧抓慢日志定位根因，超时给合理值并配 upstream keepalive 复用连接
- worker 状态：stub_status 看 active connections/reading/writing/waiting——waiting 大量而 writing 少说明后端慢、连接挂起；连接打满时 error.log 报 "worker_connections are not enough"；reload 瞬间新 worker 接管、长连接不迁移造成的瞬时 spike
- 连接数检查：`ss -s`/`ss -tan state established` 看与 upstream 的建连数是否顶到后端上限（进程 fd 限制、DB 连接池）、TIME_WAIT 堆积说明连接未复用；后端最大连接数配置
- 复盘沉淀：监控先行建立错误率/延迟/连接数时间线；分层定位法（先 nginx 后后端）写成文档；防护落地——upstream 健康检查、后端发布与入口联动摘流量、超时治理与容量预估

**减分项：**
- 502/504 分不清，只会说"后端挂了"
- 只调大超时参数了事，不查根因（慢查询/连接堆积/被动宕）
- 不看 worker 状态与连接数，定位不到"容量打满"这类真根因
- 排查靠猜，没有监控时间线与分层方法
- 没有复盘沉淀（健康检查/发布联动/超时治理）

**解答：**

复盘第一步定性：502 表示 Nginx 作为反向代理从 upstream 拿不到有效响应——典型是后端进程挂掉、端口拒绝连接、或 `max_fails` 被动健康检查把节点标记 down 导致 "no live upstreams"；504 表示请求发出去了但 Nginx 在超时窗口内没等到响应——后端活着但处理太慢或队列堵塞。两种错误在监控时间线上先分开，再确认影响面：全部 upstream 还是单个、是否与某次后端发布窗口重合（发布引发的 502 是最高频剧本——新版本刚上、进程没起来、旧节点又被摘掉）。第二步分层定位：先查 Nginx 侧——`error.log` 会给出明确的 upstream 报错（connect() failed / upstream timed out），`stub_status`（或 vts 页面）看 active connections、writing、waiting：waiting 大量而 writing 少说明后端慢、连接挂起；然后本机 `curl -v http://127.0.0.1:<upstream_port>` 探活，能通说明后端进程在，再进后端看日志（慢查询/GC/线程池耗尽）。第三步查四个关键面：① upstream 状态——nginx 的 `max_fails`/`fail_timeout` 机制：某 upstream 连续失败会被动标记 down，此时全部请求 502，但后端可能只是局部异常，配合 active health check（商业版/OpenResty 或独立健康检查脚本）尽早发现；② 超时参数——proxy_connect_timeout（默认 60s，通常过大）、proxy_read_timeout、proxy_send_timeout，如果是后端慢导致 504，盲调大超时是饮鸩止渴：连接挂起越久、worker 连接越早耗尽、雪崩越猛，正确动作是后端侧抓慢日志定位根因（慢查询、锁等待、GC），超时参数给合理值（如 connect 5s/read 30s）并配 upstream keepalive 复用连接，减少每请求建连放大压力；③ worker 状态——worker_processes × worker_connections 是连接天花板，打满时 error.log 报 "worker_connections are not enough"，此时新请求 502/504 齐发；reload 瞬间新 worker 接管、存量长连接不迁移，也可能造成瞬时 spike；④ 连接与容量——`ss -s`/`ss -tan state established` 看与 upstream 的建连数是否顶到后端上限（进程 fd 限制、DB 连接池），TIME_WAIT 大量堆积说明没开 keepalive。第四步复盘沉淀：监控先行建立"错误率/延迟/连接数"的时间线图，把"先分类型、再看影响面、按层定位、用日志与连接数说话"的排查纪律写成文档；防护措施落地——upstream 健康检查、后端发布与入口联动（发布时自动摘流量、就绪后再加回）、超时参数治理与容量预估。复盘的核心结论：502/504 是"现象"，根因永远在下游，入口层排查的纪律是"先定性、再圈范围、按层定位、用证据说话"。

**延伸考点：** "no live upstreams" 与 "connect() failed" 分别说明被动健康检查走到了哪一步？后端慢导致的 504 与 worker 连接耗尽叠加时，为什么"先调大超时"会加速雪崩（连接生命周期与 worker 连接池的关系）？

---

### Q22. 秒杀/抢购场景，Nginx 层防刷与限流方案怎么设计？

**问题：** 你们要做一场秒杀活动，预期瞬时 QPS 是平时的 50 倍，技术侧要求在 Nginx 入口先挡一道：防刷、限流、动态封禁。请设计入口层方案：limit_req/limit_conn 怎么配、IP 维度怎么做、动态封禁怎么落地、与业务侧怎么配合，才能既不误伤正常用户又扛住脚本？

**期望加分项：**
- limit_req 配置：zone 定义（`limit_req_zone $binary_realip zone=seckill_req:20m rate=30r/s`）+ `limit_req zone=seckill_req burst=20 nodelay`；讲透 rate/burst/nodelay 语义（rate 令牌补充速率、burst 突发队列长度、nodelay 突发不排队立即放行），秒杀场景通常 rate 小 + burst 适中 + nodelay，返回 503 并配 limit_req_log_level 记录
- 接口维度：用 map 按 $uri 区分秒杀接口与普通接口，秒杀单独 zone，其他接口不受牵连；limit_conn 控单 IP 并发（`limit_conn_zone $binary_realip zone=seckill_conn:20m; limit_conn seckill_conn 5;`）
- CDN 真实 IP：`set_real_ip_from <CDN网段>; real_ip_header X-Forwarded-For;` 取真实 IP，否则按 CDN 节点 IP 限流会全误伤或全放过；同时防伪造 XFF 头（可信代理才生效）
- 动态封禁：不靠手改 conf reload（流量瞬断+滞后）——OpenResty/nginx+lua 每请求查 Redis 黑名单命中即 403，封禁/解封实时生效；或 ngx_http_geo + 定时脚本生成 geo 文件并 reload；封禁策略分级（触发限流 N 次短封防脚本、恶意长封）、白名单单独放行
- 与业务侧配合：入口是"第一道闸"不是全部——压测得单机安全 QPS、按预估峰值横向扩容、后端再设本地限流与队列削峰、DB/Redis 连接池设安全水位；限流返回码与降级页统一（429/503 + 重试语义），防用户疯狂刷新二次风暴
- 阈值科学化：用历史活动数据 + 压测 + 正常用户行为建模（普通用户秒杀操作频率上限）定 rate，灰度上线（先 20% 流量观察误伤率再全量）

**减分项：**
- 只会 limit_req 一个指令，说不清 burst/nodelay 语义
- 不知道 CDN 场景取真实 IP 的坑（set_real_ip_from/real_ip_header），按节点 IP 全误伤
- 动态封禁只会"改配置 reload"（流量瞬断 + 封禁滞后）
- 以为 Nginx 限流就够了，不设计业务侧配合与容量规划
- 阈值拍脑袋，没有压测与正常用户行为模型

**解答：**

入口层方案按"挡—限—封—配"四层设计。第一层挡（基础防刷）：limit_req 是核心，先定义 zone——`limit_req_zone $binary_realip zone=seckill_req:20m rate=30r/s;`，关键点是用 `$binary_realip` 而不是 `$binary_remote_addr`：CDN 场景下 $binary_remote_addr 是 CDN 节点 IP，所有用户会限到同一批节点上，要么全误伤要么全放过，必须先 `set_real_ip_from <CDN网段>; real_ip_header X-Forwarded-For;` 取真实 IP（且只信任可信代理传的头，否则攻击者伪造 XFF 直接绕过）。接口维度用 map 区分：`map $uri $seckill_zone { ~^/api/seckill seckill_req; default ''; }`，秒杀接口单独 zone（rate 按压测得的安全水位），其他接口不受牵连。limit_req 三参数语义必须吃透：rate 是令牌补充速率、burst 是允许的突发队列长度、nodelay 表示突发请求不排队、立即放行（超过 rate+burst 的才拒绝）——秒杀场景通常"rate 小 + burst 适中 + nodelay"（排队会拖垮后端连接池）；超限返回 503 并配 `limit_req_log_level warn` 记录命中日志。第二层限（并发控制）：`limit_conn_zone $binary_realip zone=seckill_conn:20m; limit_conn seckill_conn 5;` 防单 IP 并发刷，再收紧客户端请求体大小与超时防慢速攻击。第三层封（动态封禁）：封禁不能靠手改 conf reload——流量瞬断且滞后（秒杀是分钟级的对抗），两个成熟路径：OpenResty/nginx+lua 每请求查 Redis 黑名单，命中即 return 403，封禁/解封实时生效，适合短时强对抗；或 ngx_http_geo + 定时脚本生成 geo 文件并 reload，适合分钟级封禁。封禁策略：触发限流 N 次的 IP 自动进黑名单，封禁时长分级（短封防脚本、恶意流量长封），内部测试/合作方 IP 单独白名单 geo。第四层配（与业务侧协同）：入口是"第一道闸"不是全部——先压测得出单机安全 QPS（Nginx 到后端全链路），按预估峰值横向扩容 Nginx 并同步评估后端承受力；后端入口再设一层本地限流与队列削峰（秒杀的本质是"放进来但业务侧排队处理"），DB/Redis 连接池设安全水位，防止打穿存储；限流返回码与降级页统一（429/503 + 明确重试语义），避免用户疯狂刷新造成二次风暴。阈值不能拍脑袋：用历史活动数据 + 全链路压测 + 正常用户行为建模（普通用户秒杀操作频率上限、从点击到结果页的正常间隔）定 rate，并灰度上线——先放 20% 流量观察误伤率，正常再全量，活动后复盘（限流命中率、误伤率、峰值真实 QPS 与预估对比）反哺下次阈值。

**延伸考点：** burst + nodelay 下为什么限流是"概率拦截"而非精确计数（令牌桶的突发吸收）？秒杀场景如何区分"手速快的正常用户"与"脚本"（UA/行为频次/验证码多因子联动）？限流命中日志量巨大时如何采样防刷爆日志？

---

### Q23. 几百个站点的 Nginx 配置治理与变更安全，怎么设计？

**问题：** 你们管理 300+ 个站点（多业务线的 Nginx vhost），配置散落在几十台 Nginx 上。最近出过两次事故：一次改错 upstream 名字导致大面积 502，一次配置语法对但语义错、reload 后才炸。请设计一套配置治理与变更安全体系：include 怎么组织、模板化怎么做、nginx -t 门禁怎么落地、回滚怎么设计？

**期望加分项：**
- 组织拆分：主配置 nginx.conf 只留全局（user/worker/events/http 公共段），站点配置全部外置——`include /etc/nginx/conf.d/*.conf;` 每站点一文件（文件名即域名）、sites-available/sites-enabled + 符号链接启停；公共片段收敛到 snippets（ssl-params/security-headers/limit-req-common），改一处生效全部——但要意识到这是双刃剑，改公共片段影响面是全站，走高危变更
- 模板化：300+ 站点由模板引擎（Ansible template/Confd + CMDB）生成——域名/upstream/SSL 证书路径/限流阈值全部来自数据源变量，人只改数据不碰文件；模板渲染后 `nginx -T` 检查合并结果，防"改错文件"（改了未 include 的孤儿文件）与"语义错"（upstream 名引用不存在的组，语法对但 reload 后 502）
- nginx -t 门禁：git 提交即 CI——`nginx -t`（语法）+ 渲染后 diff（与上一版本对比只允许预期变更）+ 自定义 lint（server_name 唯一性、upstream 引用存在性、监听端口冲突、证书路径存在）；发布前灰度机上再 `nginx -t -c /path` 验一遍
- 版本化与回滚：配置全部进 git（分支 + review + tag 即发布版本），发布=合入+分发+reload，回滚=切回上一 tag + reload 秒级完成；每台机器保留上一版 .bak；关键纪律——nginx -t 失败绝不 reload（nginx 会继续用旧配置运行，这是内置安全网，要利用它）
- 分批发布：reload 先上边缘/灰度 Nginx，观察无异常再全量；高危变更（改公共 snippets/upstream 大改动）强制人工确认；发布窗口固定
- 漂移审计：定期（每日）把生产 `nginx -T` 结果与 git 版本对比，抓"绕过版本库的手改漂移"——有人 ssh 上去手改配置没入库是配置治理最大的敌人

**减分项：**
- 配置全堆一个文件，无 include 拆分
- 改配置不 nginx -t 直接 reload（语法错有兜底，语义错直接 502）
- 无版本化与回滚，出事靠"记得上一版长啥样"
- 300 站点无模板化，全靠人肉改文件
- 无分批发布与漂移审计，改公共片段不看影响面

**解答：**

300 个站点的治理核心是"把配置当代码"：拆分、模板、门禁、版本化、分批、审计六件事。第一拆分：主配置 nginx.conf 只放全局（user/worker_processes/events/http 公共段如 log_format、gzip、proxy_cache_path），站点配置全部外置——`include /etc/nginx/conf.d/*.conf;` 每站点一个文件、文件名即域名，启停站点用 sites-available/sites-enabled 符号链接（加删链接比删配置安全）；公共片段收敛到 snippets（ssl-params.conf 统一 TLS、security-headers.conf 统一安全头、limit-req-common.conf 统一限流），一个站点引用一行——但必须清醒：改 snippets 是"改一处生效 300 处"，影响面是全站，要走高危变更流程，且要注意 add_header 的继承规则（location 里重新定义会丢弃父级头）。第二模板化：300 个站点不再手写，由模板引擎（Ansible template/Confd + CMDB 数据源）生成——域名、upstream、SSL 证书路径、限流阈值全部来自变量，人只改数据不碰文件；模板渲染后必须 `nginx -T` 检查 include 合并后的最终结果，重点防两类错：改错文件（改了没被 include 的孤儿文件，nginx -t 过但根本没生效）与语义错（upstream 引用了不存在的组——语法完全合法，reload 后所有请求 502，这正是上次事故的根因）。第三 nginx -t 门禁：git 提交即 CI——`nginx -t`（语法校验）+ 渲染后 diff（与上一版本对比，只允许预期内的变更，多出来的 diff 就是事故预警）+ 自定义 lint（server_name 唯一性、upstream 引用存在性、监听端口冲突、证书路径存在性），全绿才允许合入；发布前在灰度机上用 `nginx -t -c /path/to/new.conf` 再验一遍。第四版本化与回滚：配置全部进 git（分支 + review + tag 即发布版本），发布 = 合入 + 分发 + reload，回滚 = 切回上一 tag 的配置 + reload，秒级完成；分发工具自动保留每台机器上一版 .bak 兜底；关键纪律：nginx -t 失败时绝对不 reload——nginx 的 master 在配置解析失败时会拒绝 reload 并继续用旧配置运行，这是内置的安全网，必须利用而不是绕过（有些人为了"强制生效"手动重启，反而把唯一的安全网拆了）。第五分批：reload 先上边缘/灰度 Nginx，观察 10 分钟无异常再全量；高危变更（改公共 snippets、upstream 大改动）强制人工确认，发布窗口固定。第六审计：每日把生产 `nginx -T` 结果与 git 版本对比，检测"绕过版本库的手改漂移"——有人 ssh 上去手改了配置没入库，是配置治理最大的敌人，审计发现立即纠正入库。这套体系落地后，事故从"改错名字导致 502"变成"CI 拦截 + 灰度发现 + 秒级回滚"，变更从"玄学"变成"可审计的工程操作"。

**延伸考点：** include 的同名指令"后出现者生效"与 server_name 重复时的解析冲突规则，在 nginx -T 输出里怎么识别？用 Ansible 把配置分发到 30 台 Nginx 时，如何保证"某台失败不影响其他台"（串行 + 分批 + 验活）且回滚只需一条命令？

---
