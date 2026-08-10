# 数据库 · SQL（面试题库）

本文件聚焦 SQL 在真实后端与数据工程中的落地能力，覆盖多表 join 与执行计划、聚合与 group by 的边界处理、窗口函数实战（排序、同环比、累计、topN）、去重与深分页、慢 SQL 定位与改写、数据倾斜、CTE/集合操作/递归查询、分析型 SQL（漏斗、留存、复购）、日期时间与时区、数据质量校验及数仓场景下的分区裁剪与谓词下推。题目均为场景化提问，要求候选人直接给出可运行 SQL、能讲清方案取舍并用量化证据定位问题，不考概念背诵。难度自 Q1 至 Q20 循序渐进，从实践基础逐步过渡到中阶调优与架构级开放性思考题。

---

### Q1. 统计每个用户的订单数，LEFT JOIN 后结果却少了用户，为什么？

**问题：** 业务上要统计每个用户的订单数和累计金额，你写了 `SELECT u.id, COUNT(o.id) FROM users u LEFT JOIN orders o ON o.user_id = u.id WHERE o.created_at >= '2026-01-01' GROUP BY u.id`，发现结果里很多用户不见了，怎么排查和修正？

**期望加分项：**
- 能一眼看出 WHERE 条件放在 JOIN 之后会把 LEFT JOIN 变成 INNER JOIN 语义，导致无订单用户被过滤
- 能给出正解：把过滤条件下沉到 ON 子句，或先用子查询把订单表过滤再 JOIN
- 能解释 COUNT(o.id) 与 COUNT(*) 在 LEFT JOIN 下的差异：前者按订单主键计数，NULL 不计
- 能联系线上实践：报表数对不上账时先检查 JOIN 语义而不是怀疑数据
- 能主动考虑边界：JOIN 前先缩维度/事实表的量级，减少中间结果膨胀

**减分项：**
- 只会背「LEFT JOIN 保留左表所有行」，但看不出 WHERE 已经破坏了语义
- 用 COUNT(*) 去数订单导致无订单用户记成 1
- 不写任何可运行 SQL，只口头解释
- 没有提到用 EXPLAIN 看实际执行计划验证

**解答：**
先判断问题大概率不在数据而在 SQL 语义。LEFT JOIN 保留左表全部行，但 WHERE 里对右表的列做过滤发生在 JOIN 之后，等价于把没有匹配订单的用户行剔除——这一步就把它变成了内连接语义。修正方式是让过滤发生在 JOIN 过程中：把时间条件放进 ON 里，或先对订单表做子查询预过滤。同时数订单必须用 `COUNT(o.id)`（主键，NULL 不计），不能用 `COUNT(*)`，否则无订单用户会被记成 1 单。

```sql
-- 写法一：过滤下沉到 ON，语义正确
SELECT u.id, u.name, COUNT(o.id) AS order_cnt, COALESCE(SUM(o.amount), 0) AS total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
                  AND o.created_at >= '2026-01-01'
GROUP BY u.id, u.name;

-- 写法二：先缩事实表再 JOIN，配合索引通常更快
SELECT u.id, u.name, COUNT(o.id) AS order_cnt, COALESCE(SUM(o.amount), 0) AS total_amount
FROM users u
LEFT JOIN (SELECT * FROM orders WHERE created_at >= '2026-01-01') o
       ON o.user_id = u.id
GROUP BY u.id, u.name;
```

实践中的坑：一是 JOIN 后过滤是排障时最高频的「看起来没错其实错了」场景，拿到 SQL 先按 JOIN 语义逐条过 WHERE；二是别对两张几千万行的表直接做全量 JOIN，先按业务窗口缩量（本例只取近半年订单）；三是加 `(user_id, created_at)` 复合索引让 JOIN 走 Index Scan，再用 EXPLAIN ANALYZE 核对 rows 估算，确认优化器没有把 JOIN 顺序调成灾难性的笛卡尔膨胀。

**延伸考点：** 把 WHERE 条件写进 ON 和写进 WHERE 在 INNER JOIN 下等价，为什么在 LEFT/RIGHT JOIN 下不等价？ORDER BY 对 GROUP BY 结果排序时，NULL 组的默认排位是什么？

---

### Q2. 按品类汇总销售额，NULL 分类和 HAVING 过滤各踩了什么坑？

**问题：** 用 `SELECT category, SUM(amount) FROM orders GROUP BY category` 汇总销售额，发现：① 部分订单 category 是 NULL，结果里多出一行 NULL 组；② 加 `HAVING SUM(amount) > 1000` 后 NULL 组不见了，业务却要求 NULL 归为「未分类」且不能被过滤掉，怎么写？

**期望加分项：**
- 能说出 SUM 忽略 NULL、而 COUNT(*) 不忽略 NULL、AVG 遇到全 NULL 组返回 NULL 这些聚合语义差异
- 能给出 COALESCE(category, '未分类') 先归一化再分组，且让 HAVING 同时生效
- 能指出 HAVING 是分组后的过滤，与 WHERE 的行级过滤作用阶段不同
- 能联系实践：ETL 清洗时应在源头补全 NULL 分类，而不是靠报表层兜底
- 能给出可运行 SQL 并说明性能：分组列上避免函数包裹、避免 per-row 函数调用放大

**减分项：**
- 不知道 GROUP BY 会把 NULL 单独成组
- 用 WHERE category IS NOT NULL 直接砍掉 NULL 组，违背业务语义
- 混淆 HAVING 与 WHERE 的过滤时机
- 只讲思路不写 SQL

**解答：**
核心是先把 NULL 归一化为业务可见的值，再分组、再过滤。`GROUP BY` 会把 NULL 单独成组，直接用 `WHERE category IS NOT NULL` 会丢数据，违背「未分类也要统计」的需求。正确顺序是：投影阶段用 `COALESCE(category, '未分类')` 完成归一化 → 分组 → `HAVING` 过滤。注意聚合语义：`SUM` 忽略 NULL（NULL 行不参与求和），`COUNT(*)` 计所有行而 `COUNT(amount)` 不计 NULL，`AVG` 遇到全 NULL 组返回 NULL，这些差异在数据质量差的表上会直接导致报表对不上账。

```sql
SELECT COALESCE(category, '未分类') AS category,
       COUNT(*)                       AS order_cnt,
       SUM(amount)                    AS total_amount,
       AVG(amount)                    AS avg_amount
FROM orders
WHERE created_at >= '2026-01-01'          -- WHERE 先做行级过滤
GROUP BY COALESCE(category, '未分类')     -- 或 GROUP BY 1，按归一化后的值分组
HAVING SUM(amount) > 1000;                -- 分组后再过滤，NULL 组若总额达标会保留
```

实践中的坑：一是别用 `GROUP BY 1` 之类的序号写法糊弄过去，代码评审不友好且列序调整就崩；二是如果表上 `category` 有索引，`COALESCE` 会挡住索引，量级大时优先在 ETL 层把 NULL 清洗成 'unknown' 再落表；三是确认业务对「无销售额的品类」要不要展示——`GROUP BY` 只输出存在的组合，需求要全品类表时得先建维度表再 LEFT JOIN，而不是在结果里补零。

**延伸考点：** `HAVING` 里能不能用 `WHERE` 里已经过滤过的列别名？为什么 `SELECT category ... HAVING category = 'x'` 在有些数据库能过、在另一些会报错？

---

### Q3. 每个品类销量 Top3，并列排名时用 ROW_NUMBER 还是 RANK？

**问题：** 商品表 product_daily（品类、商品、当日销量）要出每个品类销量前 3 的商品，且并列的都要保留（比如第一有 3 个并列，就应全部展示），怎么写？三种排序窗口函数有什么区别？

**期望加分项：**
- 能准确说出 ROW_NUMBER / RANK / DENSE_RANK 对并列值的处理：行号不断开、排名断开、排名不断开
- 能根据「并列保留」语义选 RANK，并解释为什么不能用 ROW_NUMBER
- 能写出可运行 SQL 并说明外层 WHERE rn <= 3 的过滤方式（窗口函数在 WHERE 之后执行，必须套子查询）
- 能主动考虑边界：ORDER BY 列有 NULL 时的排序位置（默认 NULLS LAST）、并列 tie 时是否需要二次排序键
- 能联系实践：榜单类需求用 RANK，严格取 N 条用 ROW_NUMBER，连续名次展示用 DENSE_RANK

**减分项：**
- 三种窗口函数区别讲不清，混为一谈
- 直接在 WHERE 里写 rn <= 3，不知道窗口函数执行顺序在 WHERE 之后
- 不会处理并列，或并列时直接用 ROW_NUMBER 造成丢单
- 不写 SQL

**解答：**
先根据需求定语义：取「严格前 3 行」用 `ROW_NUMBER()`；「并列算同一名次、且名次断开」用 `RANK()`；「并列算同一名次、名次连续」用 `DENSE_RANK()`。题目要求并列全保留，第一有 3 个并列也要全部展示，所以选 `RANK()`，`ROW_NUMBER()` 会把并列者随机排走导致丢商品。窗口函数在 WHERE 之后执行，不能直接在 WHERE 里引用 `rn`，必须包一层子查询再过滤：

```sql
-- 商品销量表 product_daily(category, product_id, sales, dt)
SELECT category, product_id, sales, rk
FROM (
    SELECT category, product_id, sales,
           RANK() OVER (PARTITION BY category ORDER BY sales DESC) AS rk
    FROM product_daily
    WHERE dt = '2026-08-01'
) t
WHERE rk <= 3
ORDER BY category, rk;
```

实践中的坑：一是并列相同时结果行数可能超过 3，前端「Top3」展示要接受这个语义，否则应明确需求是「取 3 条」而改用 `ROW_NUMBER()`；二是 `ORDER BY sales DESC` 遇到 NULL 销量默认排最前（PG 为 NULLS LAST，MySQL 为 NULLS FIRST），榜单场景要显式处理；三是在大表上 `PARTITION BY category` 需要 `(category, sales DESC)` 或 `(category, dt, sales DESC)` 索引配合，否则每个分区都要全表扫描；四是多个排序键时（如并列按商品 ID 升序）直接在 ORDER BY 里加 tie-breaker，保证结果稳定可复现。

**延伸考点：** 如果按「每品类前 3 个不同的销量档位」取数（第 4 名销量和第 1 名相同也不展示），该用哪种函数？DENSE_RANK 用在什么业务上？

---

### Q4. 日报环比和同比都错了，问题出在 LAG 按行数偏移？

**问题：** 用 `LAG(amount, 1) OVER (ORDER BY dt)` 算每日销售额环比，发现某几天环比率明显异常；再算同比时直接 `LAG(amount, 365)`，结果对不上。数据是销售日报表 daily_sales(dt, amount)，哪里出问题了？

**期望加分项：**
- 能指出 LAG 按行偏移、按 dt 排序的隐含假设是「日期连续无缺失」，报表有缺口（节假日、补录）就会错位
- 同比场景更明显：闰年、节假日调休导致按 365 行偏移直接串行
- 能给出更稳的写法：用时间区间做自连接或 window 配合 INTERVAL 条件，而不是固定行数
- 能主动考虑边界：dt 有重复值、dt 不是唯一键时窗口 ORDER BY 结果不稳定
- 能联系实践：环比计算要用「上一实际交易日」还是「上一日历日」，业务定义要先对齐

**减分项：**
- 只会背 LAG/LEAD 语法，看不出行数偏移的坑
- 同比直接写 LAG(amount, 365) 且不解释缺陷
- 不处理日期缺口，不给可运行 SQL
- 不区分窗口 ORDER BY 与表内主键的唯一性约束

**解答：**
`LAG(amount, 1) OVER (ORDER BY dt)` 是按「排序后的行位置」取前一行的值，隐含假设 dt 连续无缺口。日报表只要缺一天（节假日无单、任务漏跑、补录），前一行的日期就不是昨天，环比就错位了；按 365 行偏移算同比更是把「365 天前」偷换成了「365 个排序列之前」，闰年直接错一天。正确做法是拿「上一实际存在记录的日期」，即按时间区间做自连接或加条件过滤：

```sql
-- 环比：用时间差做连接，天然抗日期缺口
SELECT cur.dt, cur.amount,
       prev.amount AS prev_amount,
       ROUND((cur.amount - prev.amount) * 100.0 / NULLIF(prev.amount, 0), 2) AS mom_pct
FROM daily_sales cur
LEFT JOIN daily_sales prev
       ON prev.dt = (SELECT MAX(dt) FROM daily_sales WHERE dt < cur.dt);

-- 同比：按 INTERVAL 关联去年同日，而不是 LAG 365
SELECT cur.dt, cur.amount,
       ly.amount AS last_year_amount
FROM daily_sales cur
LEFT JOIN daily_sales ly
       ON ly.dt = cur.dt - INTERVAL '1 year';
```

实践中的坑：一是环比失败多为「上一行≠昨天」，加唯一约束 `UNIQUE(dt)` 且补数任务保证写满每一天，能根治大部分错位；二是除数为 0 或 NULL 时要 `NULLIF(prev.amount, 0)`，否则整列报错；三是同比若业务要「去年同一周」或「最近 4 个完整周」，INTERVAL 写法要按业务口径微调，先和需求方对齐定义再写 SQL；四是窗口函数里 `ORDER BY dt` 若 dt 有重复值，结果不稳定，应保证排序列唯一。

**延伸考点：** 移动平均与环比都要「每行参考前 7 行」，为什么移动平均用 ROWS 窗口更合适而环比用自连接？两种写法在 5 亿行表上性能差异如何？

---

### Q5. 要累计销售额和 7 日移动平均，一条 SQL 怎么出？

**问题：** 运营要一张表：每天一行，含当日销售额、年初至今累计销售额、近 7 日移动平均。要求 7 日窗口是「今天往前 6 天」，且报表可以每天补跑（历史日期会回填数据），怎么写？

**期望加分项：**
- 能用 SUM OVER (ORDER BY dt ROWS BETWEEN ... ) 与 AVG OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) 一条 SQL 出齐
- 能说清 ROWS 与 RANGE 的区别：RANGE 默认按值合并窗口，dt 有重复值或想严格按行时要用 ROWS
- 能主动考虑边界：窗口不足 7 天时平均值偏小，是否要返回 NULL 或用累计天数做分母
- 能联系实践：历史回填会导致重跑结果变化，报表层要说明口径；累计用「自然年起始」还是「表内最早日期」要定义清楚
- 能说明大表上窗口函数一次全量排序的成本，及用物化视图/增量累计替代的可能性

**减分项：**
- 不知道窗口函数框架子句 ROWS/RANGE 的区别
- 窗口边界写错，或对头部不足 7 天的处理无感知
- 用自连接写累计导致 O(n²) 性能灾难
- 不写 SQL 或 SQL 不可运行

**解答：**
这类「每行都要看到历史窗口」的需求正是窗口函数的用武之地，别用自连接（O(n²)）也别用标量子查询。累计用 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，移动平均用 `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`。关键坑在框架子句：默认 `RANGE` 是把「值相等」的行并入同一窗口，若 dt 有重复或想严格按物理行推进，必须显式写 `ROWS`。

```sql
SELECT dt, amount,
       SUM(amount)  OVER (ORDER BY dt ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_amount,
       AVG(amount)  OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)        AS ma7,
       COUNT(amount) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)       AS ma7_days
FROM daily_sales
ORDER BY dt;
```

实践中的坑：一是头部 7 天窗口不满，AVG 会按实际天数平均，若业务要「不足 7 天显示 NULL」，外面套一层 `CASE WHEN ma7_days = 7 THEN ma7 END`；二是「累计」的起点要和需求对齐——是自然年、财年还是表内最早一天，别默认；三是数据回填：昨天跑完的累计值今天补了历史数据会变，报表要注明「截至 XX 日重跑结果」，或对历史分区做冻结；四是 5 亿行全表排序一次开销不小，频率高的报表可落成物化视图按天增量更新，只重算尾部窗口。

**延伸考点：** 若每天有多行（dt 不唯一），ROWS 与 RANGE 的结果差异具体长什么样？移动平均要「排除当天、只算前 7 天」怎么写窗口边界？

---

### Q6. 求每个用户最近一次下单的商品和金额，怎么写最高效？

**问题：** orders 表有 2 亿行（user_id, product_name, amount, order_time, id），要出每个用户最近一次下单的商品和金额，用户数几百万。候选人有三种写法：相关子查询、窗口函数、LATERAL JOIN，怎么选？为什么？

**期望加分项：**
- 能给出可运行的窗口函数写法并说明它是这类问题的主流最优解（一次排序、一趟扫描）
- 能对比三种写法的执行计划：相关子查询每用户扫一次索引、LATERAL 配合 (user_id, order_time) 索引可走 Index Scan
- 能说清 tie 问题：同一用户同一秒多单时必须以 id 或其它唯一键做 tie-breaker，否则结果不稳定
- 能给出索引建议：(user_id, order_time DESC, id DESC) 使窗口排序与索引序一致
- 能联系实践：量级大到单机内存装不下时，改用数据湖/数仓引擎的分区排序思路

**减分项：**
- 只会背「窗口函数快」，讲不清执行代价与适用条件
- 忽略同秒并列的 tie-breaker，结果不可复现
- 给不出索引与 EXPLAIN 层面的验证
- SQL 写错（PARTITION BY 放错列、子查询位置错）或不可运行

**解答：**
这类「每组取最新一条」是 topN per group 的典型，主推窗口函数：`ROW_NUMBER()` 按 `(user_id)` 分区、按 `(order_time DESC, id DESC)` 排序后取 rn=1，一趟全表扫描加一次排序即可，比相关子查询（每个用户单独扫一次）和先 `GROUP BY user_id` 再回表（两次扫描）都更可控，执行计划也好验证：

```sql
-- 方式一：窗口函数（通用、推荐）
SELECT user_id, product_name, amount, order_time
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id
                                 ORDER BY order_time DESC, id DESC) AS rn
    FROM orders
) t
WHERE rn = 1;

-- 方式二：LATERAL + 索引，配合 (user_id, order_time DESC, id DESC) 可走 Index Scan
SELECT u.id AS user_id, o.product_name, o.amount, o.order_time
FROM (SELECT DISTINCT user_id FROM orders) u
CROSS JOIN LATERAL (
    SELECT product_name, amount, order_time
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.order_time DESC, o.id DESC
    LIMIT 1
) o;
```

实践中的坑：一是 tie-breaker——同一用户同一秒有多单时必须再按 id 排序，否则每次执行返回不同行，测试直接挂；二是给 `(user_id, order_time DESC, id DESC)` 复合索引，让窗口排序免于 filesort；三是别用「先 GROUP BY user_id 取 MAX(order_time) 再自连接」的写法，同秒并列同样会炸，且多扫一次；四是若这是每日例行任务且只关心增量，加 `WHERE order_time >= now() - interval '1 day'` 把全表扫描缩成增量扫描，收益远大于在写法上抠细节。

**延伸考点：** 若数据分布极度不均（一个用户 5000 万单），LATERAL 与窗口函数谁更稳？PG 里 DISTINCT ON 也能做「每组取一条」，它和窗口函数的取舍是什么？

---

### Q7. 明细表有重复，去重统计用 DISTINCT 还是 GROUP BY 还是 ROW_NUMBER？

**问题：** 埋点事件表 events 因上游 Kafka 重发有重复行（同一 user_id + event_time 出现多次），要做两件事：① 统计去重后的独立用户数；② 把重复行清洗掉只保留一条。怎么选写法？

**期望加分项：**
- 统计独立用户数用 `COUNT(DISTINCT user_id)`，但要说出它在超大表/高基数下慢、且在分布式引擎下倾斜的代价
- 清洗去重给出 `ROW_NUMBER() OVER (PARTITION BY user_id, event_time ORDER BY id)` 保一条的写法，或 `DISTINCT ON`（PG）
- 能讲清 DISTINCT、GROUP BY、窗口函数三者的适用边界：仅去重统计用 DISTINCT，去重并取列/清洗用窗口函数，去重并聚合用 GROUP BY
- 能先定位重复键再动手，明确「哪几个列算重复」是业务决策而非 SQL 技术问题
- 能联系实践：清洗结果要与源表对账（去重前后行数对比），别删完才发现漏删或误删

**减分项：**
- 三种写法区别讲不清，只会说「都能去重」
- 不考虑 COUNT(DISTINCT) 在数据量翻倍时的性能崩塌
- DELETE 去重 SQL 不可运行或会误删（比如 PARTITION BY 键定义错）
- 不校验去重效果

**解答：**
先分清两个需求：统计类去重和物理清洗类去重。统计独立用户数，`COUNT(DISTINCT user_id)` 语法最直接；但你要知道它的代价——内存里维护巨大的 hash set，数据翻倍后速度劣化明显，Spark/Hive 里还会因为高基数倾斜，这时可先 `GROUP BY user_id` 再外层 `COUNT(*)`，或接受近似算法（HyperLogLog）。物理清洗「保留一条」最稳的是窗口函数取最小 id 行：

```sql
-- ① 找出重复键（先确认去重口径）
SELECT user_id, event_time, COUNT(*) AS cnt
FROM events
GROUP BY user_id, event_time
HAVING COUNT(*) > 1;

-- ② 删除重复，保留每条 user_id+event_time 的 id 最小行
DELETE FROM events
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY user_id, event_time
                                      ORDER BY id) AS rn
        FROM events
    ) t WHERE rn > 1
);

-- ③ 对账：清洗前后行数差 == 第①步 cnt 的和 - 重复键个数
```

实践中的坑：一是「重复」的定义（哪些列组合唯一）是业务口径，先和上游对 Kafka 重发机制，能修源头就别在数仓里反复擦；二是 DELETE 大表会写大量 WAL/日志，建议落新表 `INSERT INTO events_clean SELECT ... WHERE rn = 1` 再原子换名，比原地删安全；三是 `GROUP BY` 去重的输出没有顺序保证，若还要保留某列（如首次事件时间），必须用窗口函数或 PG 的 `DISTINCT ON (user_id, event_time) ... ORDER BY id`；四是对账是强制项，缺这一步无法证明清洗正确。

**延伸考点：** `COUNT(DISTINCT user_id, event_time)` 是合法的吗？和 `COUNT(DISTINCT user_id) + COUNT(DISTINCT event_time)` 是什么关系？HLL 近似去重的误差边界在什么量级可用？

---

### Q8. 列表接口翻到 100 万页后越来越慢，OFFSET 深分页怎么救？

**问题：** 订单列表接口用 `SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 999980` 翻页，数据量 2 亿，翻得越深越慢，MySQL 下 EXPLAIN 显示扫了 100 万行。怎么改写？

**期望加分项：**
- 能解释 OFFSET 深分页慢的本质：数据库要把前 N 行全部扫出来再丢弃，越深越贵
- 能给出键集分页（keyset pagination / seek method）写法：用上一页最后一条的 (created_at, id) 作为起点
- 能说清配套要求：排序列必须唯一（created_at 加 id 做 tie-breaker）、建复合索引 (created_at, id)
- 能说明产品层面的取舍：键集分页无法随机跳页，翻页 UI 要改造成「加载更多/上一页下一页」
- 能联系实践：索引下推后从全表扫变成只扫 20 行索引项，性能从秒级到毫秒级

**减分项：**
- 只会说「用缓存/加索引」这种正确的废话，给不出具体改写
- 不处理 created_at 同秒并列导致的漏行/重行
- 用子查询 `WHERE created_at > (SELECT created_at FROM orders ORDER BY created_at DESC LIMIT 1 OFFSET 999980)` 这种治标不治本（还是深 offset）的写法
- 不提索引设计与接口协议改动

**解答：**
先讲清原理：OFFSET 深分页的代价来自「丢弃」，数据库必须把 offset+limit 行全部扫出来再丢掉前 offset 行，offset 越大浪费越大。正解是键集分页（seek method）：记住上一页最后一条记录的排序键，下一页从它之后继续，扫描量恒定等于一页数据量：

```sql
-- 老写法（深分页越来越慢）
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 999980;

-- 新写法：客户端传上一页最后一条 (last_created_at, last_id)
SELECT * FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)   -- 行值比较，等价于 created_at<.. OR (created_at=.. AND id<..)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- 配套索引
CREATE INDEX idx_orders_created_id ON orders (created_at DESC, id DESC);
```

实践中的坑：一是排序键必须唯一——只有 created_at 会因同秒并列导致翻页漏行/重行，必须拼上 id 或其它唯一键，MySQL 不支持行值比较时写成 `WHERE created_at < ? OR (created_at = ? AND id < ?)`；二是第一页无上一页起点，接口要兼容「首页直取」；三是键集分页无法支持「跳到第 100 页」，要跟产品对齐改成「上一页/下一页」或「加载更多」，这是功能取舍不是纯技术问题；四是有 WHERE 条件时（如按状态过滤），排序键起点要携带相同过滤条件并验证索引；五是 offset 在总行数很少或后端管理页可接受慢时也够用，别为小表引入复杂度。

**延伸考点：** MySQL 与 PostgreSQL 对 `WHERE (a, b) < (x, y)` 的支持差异是什么？业务坚持要「跳到任意页」，数据量又大，还有什么折中方案（如预计算每页偏移、跳表式缓存）？

---

### Q9. 数据量翻倍后统计 SQL 从 2 秒变 20 秒，怎么定位和改写？

**问题：** 一条按天统计订单的 SQL 上线时 2 秒，一年后数据量翻了 10 倍变成 20 秒。EXPLAIN 显示 `SELECT * FROM orders WHERE DATE(created_at) = '2026-08-01'` 走的是全表扫描，怎么定位根因并改写？

**期望加分项：**
- 能按「看 EXPLAIN → 找低效算子 → 分析索引失效原因 → 改写 → 复测」的流程推进
- 能指出 `DATE(created_at) = ...` 对列做函数包裹导致索引失效（非 SARGable），改为范围查询 `>= ... AND < ...`
- 能读出 EXPLAIN 关键字段：type（ALL/range/ref/eq_ref）、key、rows 估算、Extra（Using filesort/Using temporary）
- 能主动查统计信息：行数翻了 10 倍是否触发 ANALYZE、索引是否被删/失效
- 能联系实践：先确认是「统计信息过期导致选错计划」还是「真的全表扫」，两者处置不同

**减分项：**
- 拿到慢 SQL 直接加索引，不加分析，可能加错索引或加了没用
- 看不出函数包裹列是索引失效元凶
- 只会说「建索引」，讲不清 why 和怎么验证
- 不写改写后的 SQL

**解答：**
先量化再动手，别上来就加索引。步骤一：`EXPLAIN ANALYZE` 看执行计划，重点读 `type`（ALL=全表扫描）、`key`（实际用的索引）、`rows`（估算行数）、`Extra`（Using where 扫了多少行）。步骤二：找根因——本例 `WHERE DATE(created_at) = '2026-08-01'` 对索引列套了函数，MySQL 无法对 `DATE(col)` 的结果做 B-tree 检索（非 SARGable），只能全表扫再过滤；此外还要 `ANALYZE TABLE` 排除统计信息过期选错计划的情况。步骤三：改写为范围谓词，让索引可命中：

```sql
-- 差：函数包裹列，索引失效，全表扫描
SELECT COUNT(*), SUM(amount)
FROM orders
WHERE DATE(created_at) = '2026-08-01';

-- 好：等值改写为范围，可走 (created_at) 索引
SELECT COUNT(*), SUM(amount)
FROM orders
WHERE created_at >= '2026-08-01 00:00:00'
  AND created_at <  '2026-08-02 00:00:00';

CREATE INDEX idx_orders_created ON orders (created_at);
```

实践中的坑：一是同类非 SARGable 写法还包括隐式类型转换（varchar 列跟数字比、utf8 列跟 latin1 比）、`LIKE '%xxx%'` 前导通配、`col + 1 = 5` 这种算术包裹，排查时按这个清单过一遍；二是「数据量翻倍后变慢」先怀疑统计信息过期，`rows` 估算严重偏小时直接 ANALYZE 复测，很可能不用改 SQL；三是优化后必须用 EXPLAIN 验证 `type` 从 ALL 变成 range/ref、`rows` 从千万级降到几万级，再用同样参数对比耗时，才算闭环；四是这类按天查询如果命中历史分区（分区表），范围谓词还能顺带触发分区裁剪，收益叠加。

**延伸考点：** 除了改写谓词，MySQL 里 `created_at` 上建 `DATE(created_at)` 生成列索引（函数索引）可行吗？PG 的表达式索引和它对比，适用场景有何不同？

---

### Q10. GROUP BY user_id 时头部用户占了 80% 数据，单个任务卡死，怎么破？

**问题：** 按 user_id 聚合统计（如用户消费总额），数据分布严重倾斜：top 0.1% 用户贡献了 80% 行数，Hive/Spark 里单个 reduce 卡几个小时，PG 里并行聚合也慢。怎么改写解决数据倾斜？

**期望加分项：**
- 能给出两阶段聚合方案：先对 key 加随机前缀打散（`concat(substr(md5(random())),1,2), user_id`），局部聚合后再去掉前缀二次聚合
- 能解释两阶段聚合的原理：把热点 key 先拆成多个分区并行聚合，避免单 key 落单节点
- 能说出什么时候没必要做：数据整体均匀时两阶段反而多一轮 shuffle，先看是否真有 skew key
- 能给出 Hive/Spark 和 PG 两套写法（PG 用 GROUP BY 内层嵌套，Spark 用 map 端预聚合）
- 能主动考虑边界：随机前缀粒度要覆盖热点（前缀桶数大于单热点并行度才有意义）、结果正确性不受影响

**减分项：**
- 不知道倾斜的根本是「单 key 无法并行」
- 只会背「两阶段聚合」却写不出可运行 SQL
- 不加判断对所有 key 都加前缀，白白多一轮 shuffle
- 不考虑数据正确性验证（加前缀后聚合结果要和原来一致）

**解答：**
先定性：倾斜的根因是单 key 的全部数据只能落在一个分区里串行处理，加多少资源都没用。通用解法是两阶段聚合：第一阶段给 key 拼一个随机前缀把热点拆散，让同 key 的不同行散到多个分区并行做局部聚合；第二阶段去掉前缀再做一次聚合。前缀桶数要明显大于热点 key 的潜在并行度，否则拆了等于没拆：

```sql
-- Hive/Spark SQL：两阶段聚合
SELECT user_id, SUM(cnt) AS total
FROM (
    SELECT user_id,
           SUBSTR(MD5(RAND()), 1, 4) AS tag,   -- 4 位随机前缀 ≈ 65536 桶
           COUNT(*) AS cnt
    FROM user_events
    GROUP BY user_id, SUBSTR(MD5(RAND()), 1, 4)
) t
GROUP BY user_id;

-- PostgreSQL 12+：并行聚合，倾斜严重时可显式两阶段
SELECT user_id, SUM(cnt) AS total
FROM (
    SELECT user_id, (random() * 256)::int AS tag, COUNT(*) AS cnt
    FROM user_events
    GROUP BY user_id, (random() * 256)::int
) t
GROUP BY user_id;
```

实践中的坑：一是先确认真的倾斜——`SELECT user_id, COUNT(*) FROM t GROUP BY user_id ORDER BY 2 DESC LIMIT 10` 看头部占比，均匀分布不要做两阶段；二是随机前缀会让「带排序/去重的下游」语义变化，只适用于纯聚合；三是对结果做正确性校验：改造前后总行数与关键 key 的聚合值逐一对齐，防前缀逻辑 bug；四是极端热点（如双十一单 key 千万行）还可走「热点 key 单独抽出来做广播/单独分区」的异构策略；五是 Spark 里注意两阶段前先 `repartition` 或用 map 端聚合（combiner）减少网络 shuffle 量，而 PG 里更常见的是先加 `WHERE` 缩小数据或改成增量聚合。

**延伸考点：** 随机前缀的桶数如何根据热点占比和集群并行度推算？两阶段聚合对 COUNT(DISTINCT) 这类「不可加」的聚合还成立吗，怎么处理？

---

### Q11. 一条报表 SQL 里同一个子查询被引用了三次，怎么写才不重复？

**问题：** 运营报表里「活跃用户」定义是一个子查询（近 30 天有登录），它在主 SQL 里要被用来 JOIN、做存在性判断、算占比，出现三次。直接复制粘贴子查询对吗？CTE 和子查询怎么选？

**期望加分项：**
- 能给出 WITH CTE 命名复用的写法，讲清可读性和单点维护收益
- 能说清 CTE 的代价：PG 12 前默认物化（子查询只算一次但无法下推谓词），12+ 自动 inline，必要时用 MATERIALIZED/NOT MATERIALIZED 显式控制
- 能对比「标量子查询」「EXISTS 子查询」「派生表」各自的适用场景
- 能联系实践：先看执行计划里 CTE 被扫描了几次、是否可下推，别盲目认为 CTE 一定更快
- 能主动考虑边界：同一个子查询被引用多次且每次都带不同过滤条件时，CTE 物化可能反而慢

**减分项：**
- 只会说「用 CTE 好」，讲不出原理和代价
- 不知道 PG 版本间 CTE 物化行为的差异
- 复制粘贴子查询，接受三份代码漂移风险
- 不写 SQL 或不看执行计划验证

**解答：**
复用逻辑优先用 WITH CTE 命名，收益是可读性和单点维护：活跃用户定义改一处即可。但别神话 CTE 的性能：PostgreSQL 12 之前 WITH 默认「物化」（独立算一次、无法与外部谓词下推合并），12+ 自动判断 inline，可用 `MATERIALIZED` / `NOT MATERIALIZED` 提示干预。同一子查询被引用多次且引用处过滤条件不同时，物化可能反而不如直接 inline 让优化器合并。正确姿势是写出 CTE 后用 EXPLAIN 看它被算了几次：

```sql
WITH active_users AS (
    SELECT DISTINCT user_id
    FROM login_log
    WHERE login_time >= CURRENT_DATE - INTERVAL '30 day'
)
SELECT
    (SELECT COUNT(*) FROM active_users) AS active_cnt,
    COUNT(DISTINCT o.user_id) AS buying_active_cnt,
    ROUND(COUNT(DISTINCT o.user_id) * 100.0 / NULLIF((SELECT COUNT(*) FROM active_users), 0), 2) AS pct
FROM orders o
WHERE o.user_id IN (SELECT user_id FROM active_users)
  AND o.created_at >= CURRENT_DATE - INTERVAL '30 day';
```

实践中的坑：一是「子查询被引用三次」若每次还要拼不同的额外过滤（如时间窗口不同），硬公用一个 CTE 反而逼优化器物化全量，此时按引用点分别 inline 更优——用 EXPLAIN ANALYZE 对比两种写法的实际耗时再定；二是 `IN (SELECT ...)` 与 `EXISTS` 在优化器下往往等价，但子查询返回 NULL 时 IN 语义会变（见集合操作题），注意别踩；三是 CTE 里做了聚合/去重后引用时不能再沿用原表字段名，SQL 语义要自洽；四是多引擎差异：MySQL 8 的 CTE 每次都物化到临时表，引用多次大结果集时内存压力大，CTE 在 MySQL 里更偏「可读性」而非「性能优化」。

**延伸考点：** PG 里同一个 CTE 被引用两次且各带不同 WHERE，MATERIALIZED 与 inline 各发生什么？在什么数据量下 CTE 物化反而比子查询快？

---

### Q12. 用一条 SQL 把用户按消费行为分层并出分布，怎么写？

**问题：** 用户表 users + 订单表 orders，要出「近 90 天消费分层分布」：活跃（有单且最近一单在 30 天内）、沉睡（有单但最近一单在 30~90 天前）、流失（最近一单在 90 天前或从未下单），以及各层人数和人均消费。怎么写？

**期望加分项：**
- 能给出「先聚合用户级指标（订单数、最近下单时间、总金额）→ CASE WHEN 打标 → 按标签 GROUP BY」的分步写法
- 能指出 CASE WHEN 分支顺序很重要：互斥分层时必须先判最严格的条件（如活跃在前），否则被前面的分支吞掉
- 能主动处理从未下单的用户（LEFT JOIN 后 COALESCE，归属「流失」）
- 能给出可运行 SQL 并说明性能：用户级聚合先缩行再打标，避免对明细行重复打标
- 能联系实践：分层口径要和业务对齐，不同的「活跃」定义会显著改变分布结论

**减分项：**
- 直接把 CASE WHEN 写在明细行上再 GROUP BY 用户，逻辑混乱
- 分支顺序写错导致分层互相吞并，结果分布和预期差很多
- 未处理从未下单的用户
- 只讲思路不写 SQL

**解答：**
套路是「先做用户级汇总，再打标，再统计」，三层层层递进，别在明细行上直接 CASE WHEN——同一用户多行会重复计。用户级指标用 LEFT JOIN 保证没下过单的用户也在，`MAX(order_dt)` 取最近一单时间，`COALESCE` 兜底。打标时分支顺序很关键：活跃、沉睡、流失是互斥档位，必须把「最近 30 天有单」判在最前，否则活跃用户会被后面的条件误判：

```sql
WITH user_metric AS (
    SELECT u.id,
           COUNT(o.id)                    AS order_cnt,
           MAX(o.created_at)              AS last_order_at,
           COALESCE(SUM(o.amount), 0)     AS total_amount
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
       AND o.created_at >= CURRENT_DATE - INTERVAL '90 day'
    GROUP BY u.id
)
SELECT CASE
           WHEN order_cnt = 0                              THEN '流失(从未下单)'
           WHEN last_order_at >= CURRENT_DATE - INTERVAL '30 day' THEN '活跃'
           WHEN last_order_at >= CURRENT_DATE - INTERVAL '90 day' THEN '沉睡'
           ELSE '流失'
       END AS segment,
       COUNT(*) AS user_cnt,
       ROUND(AVG(total_amount), 2) AS avg_amount
FROM user_metric
GROUP BY 1
ORDER BY 2 DESC;
```

实践中的坑：一是分支顺序——把「流失」放最前会把所有人判成流失，测试时用「各层人数之和 == 总用户数」做自检；二是时间口径要和业务对齐：活跃是「最近 30 天」还是「最近 30 天有消费」，差的定义会完全改变分布；三是本例把订单时间窗口限制在 90 天，导致 `order_cnt`/`total_amount` 是「近 90 天」口径，报表命名要带窗口避免歧义；四是 `GROUP BY 1` 可读性差且依赖列序，正式代码写 `GROUP BY segment` 或完整表达式；五是分层如果后续要复用（如每日跑一次存表），把 user_metric 落成中间表，避免每个报表重复扫 orders。

**延伸考点：** 分层想支持「同时属于多个标签」（多标签而非互斥），CASE WHEN 还够用吗？要不要用一行一个标签的宽表/长表结构？

---

### Q13. 两张订单表对账，找出「A 有 B 没有」的订单，为什么 NOT IN 查出来是空？

**问题：** 上月末的订单导成两个文件进了表 orders_a 和 orders_b，要对账找出「A 有而 B 没有」的 order_id。同事写了 `SELECT order_id FROM orders_a WHERE order_id NOT IN (SELECT order_id FROM orders_b)`，结果一行都没有，但实际差异很大，哪错了？怎么改？

**期望加分项：**
- 能点出经典坑：NOT IN 的子查询结果若含 NULL，整个表达式恒为 UNKNOWN，结果集为空
- 能给出 NOT EXISTS 改写（NULL 安全），以及 EXCEPT 写法（两表去重取差集）
- 能说清 UNION 与 UNION ALL 的语义差异：UNION 隐式去重（有排序/哈希开销），明确无重复时用 UNION ALL
- 能主动考虑边界：order_id 本身是否允许 NULL、两表 order_id 类型/字符集是否一致（隐式转换会让索引失效）
- 能联系实践：对账 SQL 要双向做（A∖B、B∖A）并核对行数总和，还要考虑大表下 NOT EXISTS vs LEFT JOIN 的执行计划差异

**减分项：**
- 遇到空结果不查原因，直接怀疑数据
- 只会改写成 `NOT IN` 加 `IS NOT NULL`，不解释根因
- 分不清 UNION 与 UNION ALL 的开销差异
- 不写 SQL 或不给集合操作的等价写法对比

**解答：**
根因是 SQL 三值逻辑：`x NOT IN (子查询)` 等价于 `x <> 所有子查询值`，只要子查询结果里混入一个 NULL，`x <> NULL` 就是 UNKNOWN，逐条比较全是 UNKNOWN，`NOT IN` 整体为 UNKNOWN，一行都选不出来。修复有两条路：一是换 NULL 安全的 `NOT EXISTS`；二是直接用集合差运算 `EXCEPT`（MySQL 8.0.31+ 才支持 EXCEPT，老版本用 NOT EXISTS/LEFT JOIN）。顺手把 UNION vs UNION ALL 一起讲掉：UNION 去重需要排序或哈希、有额外开销，两表结构明确不重复时用 UNION ALL：

```sql
-- 修复一：NOT EXISTS（NULL 安全，兼容所有引擎）
SELECT a.order_id
FROM orders_a a
WHERE NOT EXISTS (SELECT 1 FROM orders_b b WHERE b.order_id = a.order_id);

-- 修复二：EXCEPT 集合差（PG/SQL Server/MySQL 8.0.31+）
SELECT order_id FROM orders_a
EXCEPT
SELECT order_id FROM orders_b;

-- 顺带：对账要双向 + 核对总数
-- SELECT 'A_only' AS side, COUNT(*) FROM (SELECT order_id FROM orders_a EXCEPT SELECT order_id FROM orders_b) t
-- UNION ALL
-- SELECT 'B_only', COUNT(*) FROM (SELECT order_id FROM orders_b EXCEPT SELECT order_id FROM orders_a) t;
```

实践中的坑：一是 order_id 若本身允许 NULL，`NOT EXISTS` 里 NULL 行的语义仍要人工确认；二是两表 order_id 的字符集/类型不一致会触发隐式转换，可能吃掉索引，对账前先 `\d` 或 SHOW CREATE TABLE 核对列定义；三是 `NOT EXISTS` 与 `LEFT JOIN ... WHERE b.order_id IS NULL` 在优化器下常等价，但 NOT EXISTS 语义更直白、不易被 JOIN 膨胀误导；四是大表对账先看有没有 `UNIQUE(order_id)` 索引，否则 A 全表扫 × B 索引探测，量级再大就得换大数据引擎做。

**延伸考点：** `UNION` 与 `UNION ALL` 的结果差异在什么场景会造成「行数对不上」的假象？`INTERSECT` 找共同订单时，NULL 的处理和 EXCEPT 有什么不同？

---

### Q14. 漏斗和次日留存各怎么写？一个用户可能跨多天/多次访问，怎么防重复计数？

**问题：** 行为表 user_events(user_id, event_name, event_time)，事件有 register、add_to_cart、checkout。① 求「注册→加购→下单」三步漏斗各环节人数；② 求每日 DAU 的次日留存率。注意一个用户可能一天内多次加购，怎么保证不重复计？

**期望加分项：**
- 漏斗用「事件是否发生」而非「事件次数」，计数前先按 user_id 去重，用 MAX(CASE WHEN ...) 或 EXISTS 判断
- 给出可运行漏斗 SQL：先宽表化（按用户 pivot 出各事件最早发生时间），再按链路条件计数
- 留存用「第 N 天活跃的用户，第 N+1 天是否活跃」的自连接或条件计数，同样按 user_id+日期去重
- 能主动考虑边界：漏斗要按事件先后排序（先注册后加购），时间顺序写进判断条件
- 能联系实践：事件埋点重复上报先做去重层，漏斗与留存口径要和业务确认（注册后 7 天内加购算不算漏斗内）

**减分项：**
- 漏斗直接 COUNT(event_name)，把一人多次加购重复计
- 留存算成「次日活跃事件数/当日活跃事件数」，分母分子口径不一致
- 不处理事件先后顺序
- 只讲思路不写 SQL

**解答：**
两个问题的共同陷阱是「同一用户同一统计口径只能计一次」。漏斗先按 user_id 把每个事件是否发生、最早发生时间做成宽表，再逐环节过滤——注册环节计注册用户数，加购环节计「已注册且加购」的用户数，严格漏斗还要求时间上注册早于加购早于下单：

```sql
-- 漏斗：按 user_id 宽表化，MAX/MIN 只做存在性判断，不会重复计数
WITH funnel AS (
    SELECT user_id,
           MIN(event_time) FILTER (WHERE event_name = 'register')    AS t_register,
           MIN(event_time) FILTER (WHERE event_name = 'add_to_cart') AS t_cart,
           MIN(event_time) FILTER (WHERE event_name = 'checkout')    AS t_checkout
    FROM user_events
    WHERE event_time >= CURRENT_DATE - INTERVAL '30 day'
    GROUP BY user_id
)
SELECT
    COUNT(*)                                       AS register_cnt,
    COUNT(*) FILTER (WHERE t_cart    IS NOT NULL AND t_cart    >= t_register) AS cart_cnt,
    COUNT(*) FILTER (WHERE t_checkout IS NOT NULL AND t_checkout >= t_cart)   AS checkout_cnt
FROM funnel;

-- 次日留存：DAU 表或活跃事件去重后自连接
WITH dau AS (
    SELECT DISTINCT user_id, event_time::date AS dt
    FROM user_events
    WHERE event_name IN ('any_active_event')   -- 按业务定义活跃事件
)
SELECT a.dt,
       COUNT(DISTINCT a.user_id) AS dau,
       COUNT(DISTINCT CASE WHEN b.user_id IS NOT NULL THEN a.user_id END) AS retained
FROM dau a
LEFT JOIN dau b ON a.user_id = b.user_id
              AND b.dt = a.dt + 1              -- 次日；7 日留存改 b.dt = a.dt + 7
GROUP BY a.dt
ORDER BY a.dt;
```

实践中的坑：一是漏斗宽表里 `MIN(event_time)` 同时承担「是否存在」与「最早时间」两个职责，条件写错（如没有 `>= t_register` 的时间顺序约束）会把「先下单后注册」的异常用户算进漏斗；二是留存分母 DAU 按「活跃事件去重后的用户」，若埋点把页面 PV 也当活跃事件，DAU 会虚高、留存率虚低——口径要先定；三是大表上 `GROUP BY user_id` 宽表化在 PG 可用 FILTER 语法（MySQL 没有 FILTER，要写成 `MIN(CASE WHEN event_name='register' THEN event_time END)`），多引擎团队要注意方言差异；四是漏斗/留存通常每日增量计算并落表，别让每个报表现场全量扫行为表。

**延伸考点：** 7 日留存、30 日留存怎么扩展？「注册后 7 天漏斗」的窗口约束要加在哪个位置？宽表化漏斗在事件种类很多（几十个事件）时有什么替代方案？

---

### Q15. 订单时间存的是 UTC，按北京时间出日报，结果每天串了 8 小时的数据，怎么办？

**问题：** 线上订单表 orders 的 created_at 是 UTC 时间（timestamptz），要按北京时间（UTC+8）出「每日销售额日报」，结果发现每天 8 点到 12 点的订单被算到前一天或后一天，怎么正确写？如果还要按「业务周」（周一起始）统计呢？

**期望加分项：**
- 能正确使用时区转换：PG 里 `created_at AT TIME ZONE 'Asia/Shanghai'` 得到北京时间 timestamptz 再取 date
- 能讲清 PG 的 AT TIME ZONE 双语义：timestamptz 转本地（换算时间）与 timestamptz 转 timestamp（贴标签），并指出这是最容易写反的坑
- 能主动考虑边界：跨天订单、夏令时（业务库一般固定 +8）、月份/季度分组时 `date_trunc` 的配合
- 能提醒「报表层统一时区」与「DB 连接会话时区一致」两种工程手段，避免每张报表各写各的
- 能给出可运行 SQL 且避免对索引列套函数导致分区裁剪/索引失效的问题

**减分项：**
- 直接 `::date` 或 DATE(created_at)，把 UTC 当北京时间取日，串 8 小时
- 分不清 AT TIME ZONE 转换的两个方向，写反了时间
- 不知道业务周从周一起算要用 date_trunc('week', ...) 或 ISODOW
- 对索引列套函数后不补偿（如用生成列/表达式索引）
- 不写 SQL

**解答：**
根因是「取日」的时区不对：`created_at::date` 用的是会话时区（通常是 UTC 或服务器本地），把 UTC 时间直接截成北京日期，8:00~12:00 的订单就跑到昨天去了。正解是先把 UTC 转成北京时间再取日。PG 里 `AT TIME ZONE` 有双语义：`timestamptz AT TIME ZONE 'Asia/Shanghai'` 返回「北京时间」的 timestamp（时间已换算，此后取 date 就是北京日期）；反过来 `timestamp AT TIME ZONE 'Asia/Shanghai'` 是给一个 naive 时间贴 +8 标签转回 UTC。写反了时间会差 8 小时，这是高频坑：

```sql
-- 正确：先换算成北京时间再取日
SELECT (created_at AT TIME ZONE 'Asia/Shanghai')::date AS biz_date,
       COUNT(*), SUM(amount)
FROM orders
WHERE created_at >= (CURRENT_DATE - 1) AT TIME ZONE 'Asia/Shanghai'  -- 注意避免对列套函数
GROUP BY 1;

-- 业务周：周一起始，用 date_trunc + ISODOW
SELECT date_trunc('week', created_at AT TIME ZONE 'Asia/Shanghai')::date AS week_start,
       SUM(amount)
FROM orders
GROUP BY 1;
```

实践中的坑：一是别在 WHERE 里对列写 `(created_at AT TIME ZONE 'Asia/Shanghai')::date = '2026-08-01'`——对索引列套函数会废掉索引和分区裁剪，应像上面示例把转换放到比较值一侧（`created_at >= 北京时间起始时刻换算回 UTC`）；二是 MySQL 无原生 timestamptz 概念，用 `CONVERT_TZ(created_at, 'UTC', '+08:00')` 且参数是字符串时同样废索引，量级大建议落一列 `biz_date`（生成列+索引）直接按业务日查询；三是夏令时地区业务时间戳不要用 'US/Pacific' 这种会跳小时的时区做取日，明确「业务统一 +8」；四是日报的「天」到底按消费时刻的北京日还是支付成功的北京日，口径要和财务对齐，SQL 只负责落地定义。

**延伸考点：** PG 里 `SELECT '2026-08-01 20:00:00+00' AT TIME ZONE 'Asia/Shanghai'` 和 `AT TIME ZONE 'UTC'` 分别返回什么？会话时区 `SET TIME ZONE` 会影响 `::date` 的结果吗？

---

### Q16. 埋点参数是 JSON 字符串，要提取 campaign 并按手机号脱敏统计，怎么写？

**问题：** 事件表 events 的 params 列是 JSON 字符串（如 `{"campaign": "summer", "mobile": "13812345678"}`），要做两件事：① 按 campaign 统计事件数；② 报表里手机号要脱敏显示成 `138****5678`。怎么写？如果 params 里混着非法 JSON 怎么办？

**期望加分项：**
- 能用 `params::jsonb ->> 'campaign'`（PG）或 JSON_EXTRACT（MySQL）提取，并给出可运行 SQL
- 能用正则 `regexp_replace` 做手机号脱敏，注意正则写对（保留前 3 后 4，中间打码）且对非手机号值不误伤
- 能主动处理非法 JSON：先 `WHERE jsonb_typeof(...) IS NOT NULL` 或捕获异常，说明生产数据必然有脏数据
- 能提性能：params 全表解析成本高，量级大应在上游抽成列（ETL 落明细字段）而不是报表现场解析
- 能联系实践：脱敏是权限问题而非纯 SQL 问题，DB 层做掩码视图/权限控制更彻底

**减分项：**
- 不会 JSON 提取语法，或者写出来的 SQL 在真实库上跑不通
- 正则写错（如把手机号中间全打码、或把非手机号字段也打了）
- 忽略非法 JSON / NULL 直接崩或统计口径错
- 只讲思路不写 SQL

**解答：**
先用 JSON 提取把字段抽出来，再用正则脱敏。PG 的 `->>` 取文本、`->` 取 jsonb，提取后 `regexp_replace` 做脱敏：把中间 4 位替换为星号。关键点是容忍脏数据：真实埋点里 params 会混入空串、`{}`、非法 JSON，`->>` 拿不到键时返回 NULL，统计时按业务口径 `COALESCE(campaign, 'unknown')` 归一化；如果是非法的 JSON 文本，PG 的 `::jsonb` 会直接报错，先用 `jsonb_typeof` 或异常捕获过滤：

```sql
-- 提取 + 脱敏（PG）
SELECT COALESCE(params::jsonb ->> 'campaign', 'unknown') AS campaign,
       COUNT(*) AS cnt,
       MAX(regexp_replace(params::jsonb ->> 'mobile',
                          '^(\d{3})\d{4}(\d{4})$', '\1****\2')) AS sample_mobile
FROM events
WHERE params IS NOT NULL
  AND params != ''
GROUP BY 1;

-- 兼容非法 JSON：先按能否解析过滤（PG）
SELECT COALESCE((params::jsonb ->> 'campaign'), 'unknown') AS campaign, COUNT(*)
FROM events
WHERE jsonb_typeof(NULLIF(params, '')::jsonb) IS NOT NULL
   OR params = ''
GROUP BY 1;
-- 说明：NULLIF(params,'')::jsonb 对非法串会抛错，更稳的做法是自定义函数或上游清洗，
-- 生产环境请在 ETL 层就过滤非法 JSON，报表层不背这个锅。
```

实践中的坑：一是正则 `^(\d{3})\d{4}(\d{4})$` 用 `^...$` 锚定整串，避免把 11 位之外的号码误伤；脱敏只能挡展示层，真正合规要「最小权限 + 脱敏视图 + 审计」，纯 SQL 打码挡不住有 DB 权限的人；二是对 `params` 全表做 `::jsonb` 解析成本不低，百万行级还凑合，亿级事件表必须在写入时或 ETL 里把 campaign/mobile 抽成独立列并建索引，报表直接查列；三是不同引擎方言差异要提前对齐：PG 用 `->>/->`，MySQL 用 `JSON_EXTRACT(params, '$.campaign')` 或 `params->>'$.campaign'`（8.0 起）、Spark 用 `get_json_object`，写进代码评审规则里避免混用；四是 `MAX()` 只是取个样例展示，真实报表应把脱敏列放进明细表/视图，别在聚合里做。

**延伸考点：** 正则脱敏遇到手机号带空格/国家码（+86）时怎么处理？JSON 里嵌套多层（params.extra.mobile）的提取语法是什么？

---

### Q17. 组织架构表是 parent_id 自关联，怎么查出某个部门下的所有层级成员？

**问题：** 部门表 dept(id, name, parent_id) 是一棵多级树（最大 8 层），要查「ID=5 的部门及其所有子孙部门」的全部员工（员工表 employee.dept_id 关联）。怎么写？如果数据里有环（parent_id 指回自己或成环）会怎样？

**期望加分项：**
- 能给出 WITH RECURSIVE 递归 CTE 的可运行写法，带层级号（lvl）和路径（path）字段
- 能解释递归 CTE 的语义：anchor（基集）+ recursive term（自引用）+ UNION 去重防重复展开
- 能主动考虑防环：递归里检查「新节点 id 不在已有 path 中」或限制最大深度，避免死循环
- 能给出先得部门集、再 JOIN 员工的完整 SQL
- 能联系实践：深度固定（≤8 层）时也可以用 N 次 join 或「维护祖先链列」的闭包表/物化路径方案，比较取舍

**减分项：**
- 只会说「用递归 CTE」，写不出可运行 SQL 或语法错误（UNION ALL 死循环、缺 anchor）
- 不知道数据成环时递归会无限展开把库拖死
- 不给层级字段，报表无法缩进展示
- 不讨论闭包表/物化路径这些替代建模

**解答：**
递归 CTE 是树查询的标准解：anchor 先取根（ID=5 自身），recursive 部分每次向上一步找子节点，`UNION`（去重）保证已展开的节点不再重复进入递归。带上 `lvl` 和 `path` 方便报表展示层级和做防环判断。数据有环时（如 A 的 parent 是 B、B 的 parent 又是 A），不加防护会无限递归直到撞上 `max_recursive_iterations`/内存爆炸，必须在递归里检查「新节点是否已出现在 path 中」：

```sql
WITH RECURSIVE dept_tree AS (
    -- anchor：起始部门
    SELECT id, name, parent_id, 1 AS lvl, ARRAY[id] AS path
    FROM dept
    WHERE id = 5

    UNION

    -- recursive：向下一层
    SELECT d.id, d.name, d.parent_id, t.lvl + 1, t.path || d.id
    FROM dept d
    JOIN dept_tree t ON d.parent_id = t.id
    WHERE NOT (d.id = ANY(t.path))          -- 防环：已出现过的节点不再进递归
)
SELECT dt.id, dt.name, dt.lvl,
       e.name AS employee_name
FROM dept_tree dt
LEFT JOIN employee e ON e.dept_id = dt.id
ORDER BY dt.path;                            -- path 天然给出深度优先的展示顺序
```

实践中的坑：一是必须用 `UNION` 而不是 `UNION ALL`——递归语义要求去重，否则重复路径会指数级展开；二是 `WHERE NOT (d.id = ANY(t.path))` 是防环关键，生产环境组织架构表必然出现过脏环（历史数据迁移导致），递归上限 `SET max_recursive_iterations` 只是兜底；三是 MySQL 8 的 `WITH RECURSIVE` 同样适用但无 path 类型语法差异，老 MySQL 只能 N 次 join 或程序循环；四是当树是「静态、读多写少」时，更稳的建模是闭包表（closure table，直接存所有祖先后代对）或物化路径列（path 字符串），查询变普通 join 且免递归风险，写操作的复杂度换查询的确定性，规模大了值得重构；五是别把递归 CTE 当万能，它在每个递归步都要重扫 dept，深树 + 大部门表时执行计划要 EXPLAIN 验证。

**延伸考点：** 递归 CTE 里 `UNION` 和 `UNION ALL` 对「路径去重」的影响具体是什么？如果树的深度不确定且可能超 100 层，递归 CTE 的栈/迭代限制怎么设？

---

### Q18. 数仓每张表加工完要自动校验质量，行数、空值、重复、边界怎么写？

**问题：** 数仓 DAG 里每张表（如 dwd_orders）加工完要做数据质量校验，校验项包括：总行数、主键重复数、关键列空值率、金额异常值（负数/超上限）、与上游源表的行数差。用 SQL 怎么写一份「一次查出所有指标」的校验脚本？跑完怎么判定通过？

**期望加分项：**
- 能给出单条 SQL 聚合出多个校验指标的写法（COUNT、COUNT FILTER、COUNT(DISTINCT)、MIN/MAX）
- 能给出「断言表」或「校验结果表」设计：指标值 + 阈值 + 是否通过，接入调度失败告警
- 能主动考虑边界：主键重复用 `COUNT(*) - COUNT(DISTINCT 主键)` 或 GROUP BY HAVING；空值率分母要不要排除 NULL 整行；金额负数用 `MIN(amount) < 0` 或 FILTER 计数
- 能给出源/目标行数对比 SQL（按分区键核对）
- 能联系实践：校验要与分区绑定（按 dt 过滤），避免全表扫描整张历史表

**减分项：**
- 只写一条 `SELECT COUNT(*)` 完事，覆盖不了空值/重复/边界
- 校验结果没有阈值和通过/失败判定，起不到告警作用
- 空值率、重复数的口径含糊
- 不写 SQL 或 SQL 在真实库上跑不通

**解答：**
质量校验的套路是「一条聚合 SQL 出全部指标 + 一行断言判定」。PG 的 `FILTER` 子句特别适合多条件计数，一行算完所有检查项；没有 FILTER 的引擎（MySQL）改成 `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`。按分区过滤是硬要求——校验脚本必须带 `WHERE dt = :biz_date`，否则每次全表扫历史。判定用「期望值 vs 实际值」比较，产出 0/1 供调度系统当退出码用：

```sql
-- dwd_orders 校验（PG 方言，MySQL 把 FILTER 换成 CASE WHEN 等价写法）
SELECT
    :biz_date AS biz_date,
    COUNT(*)                                                       AS total_rows,
    COUNT(*) - COUNT(DISTINCT order_id)                            AS dup_order_cnt,
    COUNT(*) FILTER (WHERE user_id IS NULL)                        AS null_user_cnt,
    ROUND(100.0 * COUNT(*) FILTER (WHERE user_id IS NULL) / COUNT(*), 4) AS null_user_pct,
    COUNT(*) FILTER (WHERE amount < 0)                             AS neg_amount_cnt,
    COUNT(*) FILTER (WHERE amount > 1000000)                       AS over_limit_cnt,
    MIN(amount), MAX(amount)
FROM dwd_orders
WHERE dt = :biz_date;

-- 判定：任何一个阈值超限即失败（返回行数>0 → 调度告警）
SELECT 'FAIL' AS check_result
WHERE (SELECT COUNT(*) - COUNT(DISTINCT order_id) FROM dwd_orders WHERE dt = :biz_date) > 0
   OR (SELECT COUNT(*) FROM dwd_orders WHERE dt = :biz_date AND amount < 0) > 0
UNION ALL
SELECT 'PASS' WHERE NOT EXISTS (SELECT 1 FROM dwd_orders WHERE dt = :biz_date AND (order_id IS NULL OR amount < 0));
```

实践中的坑：一是「空值率」分母定义要先统一——除以总行数还是非空行数，报表与校验要同一口径；二是重复判定在 order_id 允许 NULL 时 `COUNT(DISTINCT)` 会忽略 NULL，主键重复检测应同时查 `order_id IS NULL` 的条数；三是行数对比要和上游源表按「分区键 + 去重口径」对齐（源表可能本来就带重复，这时差异阈值要放宽并单独看重复项）；四是校验结果落一张 `dqc_check_result` 表（biz_date、table_name、check_name、actual、threshold、passed），历史趋势能反哺「哪些表质量在恶化」；五是别只校验加工后的表，源头输入表也要校验，否则下游永远在为上游脏数据买单，正确姿势是 DAG 里每个节点的入边都挂校验。

**延伸考点：** 金额异常（负数）在「退款单」业务里本身就是合法的，这类业务型例外怎么在通用校验框架里配置白名单？跨分区「昨日 vs 今日」的波动率校验（如行数变化 ±20%）怎么写？

---

### Q19. 大宽表按天分区，报表 SQL 却扫了全表，分区裁剪和谓词下推哪里没生效？

**问题：** fact_orders 是按 dt 分区的 5 亿行大宽表（数仓/Hive 或 PG 分区表），报表 SQL `SELECT ... FROM fact_orders f JOIN dim_product p ON f.product_id = p.id WHERE p.category = 'x'` 本意是看某天数据，EXPLAIN 却显示扫了所有分区，为什么？怎么改才能让分区裁剪和谓词下推生效？

**期望加分项：**
- 能点出根因：对分区列没写等值/范围谓词，或谓词被写在了「无法裁剪」的位置（如 JOIN 另一表的过滤条件无法回推、对分区列套了函数）
- 能给出改写：把 `f.dt = '2026-08-01'` 显式写进最内层，JOIN 前先过滤事实表和维度表（谓词下推）
- 能说明 EXPLAIN 里如何验证分区裁剪是否生效（PG 的 Partition Pruning 提示、Hive 的 partitions 扫描数、扫描字节数对比）
- 能主动考虑边界：谓词下推对「先 JOIN 后过滤 vs 先过滤后 JOIN」的结果一致性没有影响，但性能差异巨大；维度表小表也应先过滤再参与 JOIN
- 能联系实践：报表 SQL 里常见的「表别名列未带分区谓词」「条件写在 CASE WHEN/子查询里导致无法回推」两类反模式

**减分项：**
- 只知道「分区表要带分区条件」，讲不清谓词写在哪一层才能被裁剪
- 不知道对分区列套函数（如 `f.dt + 1 = '2026-08-02'`、`substr(f.dt,1,7)='2026-08'`）会让裁剪失效
- 不会用 EXPLAIN 验证是否真的只扫了目标分区
- 不写改写后的 SQL

**解答：**
先定位裁剪失效的原因。分区裁剪的前提是「分区列上出现可静态求值的等值/范围谓词」；本例的 SQL 只过滤了维度表的 category，JOIN 后谓词能否回推到事实表取决于优化器能力——Hive/Spark 里这种回推常失败（扫描阶段根本不知道要哪些 dt），PG 里也要看 JOIN 顺序。再加上谓词若被函数包裹（`f.dt + 1 = ...`）或藏在子查询/CASE 里，裁剪必然失效。正确写法是「谁过滤谁先缩」，把分区谓词显式写在事实表最内层，维度表也先按 category 过滤成小集合再 JOIN，让下推成为确定行为而不是依赖优化器发挥：

```sql
-- 差：只有维度表过滤条件，事实表分区谓词缺失/不可回推，扫全部分区
SELECT f.dt, f.order_id, SUM(f.amount)
FROM fact_orders f
JOIN dim_product p ON f.product_id = p.id
WHERE p.category = 'x'
GROUP BY f.dt, f.order_id;

-- 好：事实表先做分区裁剪，维度表先过滤，谓词显式可下推
SELECT f.dt, f.order_id, SUM(f.amount)
FROM (SELECT dt, order_id, product_id, amount
      FROM fact_orders
      WHERE dt = '2026-08-01') f          -- 分区谓词写在最内层，裁剪确定生效
JOIN (SELECT id FROM dim_product
      WHERE category = 'x') p ON f.product_id = p.id
GROUP BY f.dt, f.order_id;

-- 验证：PG 看 EXPLAIN 里 Partition Pruning: via index/constraint；Hive 看扫描的 partitions 数（应=1）
```

实践中的坑：一是「裁剪」和「下推」是两件事——裁剪管扫描多少分区，下推管过滤在 JOIN 前还是 JOIN 后执行，都要 EXPLAIN 验证，别只改一个；二是对分区列做运算（`dt + interval '1 day'`、`to_date(dt,'YYYYMMDD')`）会直接禁用裁剪，这是 Hive/PG 分区表上最隐蔽的坑；三是分区键与 WHERE 值类型不一致（字符串日期 '20260801' vs date）会触发隐式转换同样失效，先对齐类型；四是日期过滤如果来自「昨天」这类动态值，要保证每天重跑都能换成新分区，模板化 SQL 时把动态分区条件放进同一层；五是量级大时「先过滤后 JOIN」不只是优化，还决定能不能跑完——全分区扫描在 PB 级表上直接 OOM，必须当成硬约束。

**延伸考点：** Hive 与 PG 分区表在「JOIN 谓词回推」上的行为差异是什么？列裁剪（只 SELECT 需要的列）对 Parquet/ORC 存储的扫描量影响有多大？

---

### Q20. 300 行的大报表 SQL 改一个字段要半天，怎么拆分复用？测试环境要造 1000 万行数据呢？

**问题：** 同事留了一张 300 行的日报 SQL：多段子查询嵌套、三处重复定义「活跃用户」、日期参数散落在 7 个位置，每改一个口径要小心翼翼。① 怎么重构这张报表 SQL？② 测试环境要造 1000 万行订单数据（user_id、金额、时间分布接近线上），怎么用 SQL 快速生成？

**期望加分项：**
- 能给出「明细层 → 轻度汇总层 → 报表层」的分层拆分思路：把可复用的用户/订单指标下沉为视图或中间表，报表层只做组装
- 能说清视图与物化视图的取舍：普通视图是「SQL 宏」不存数据（重跑时仍全量算），物化视图存结果、刷新有成本，报表复用频繁选物化，口径多变选视图
- 能给出 generate_series + random 造数的可运行 SQL，并说明要按线上分布造（金额对数分布、时间近多远少、用户数级可控）
- 能主动考虑边界：造数要保证主键唯一、外键有效（user_id 要落在用户表里），否则下游 join 全是 NULL
- 能联系实践：报表参数化（日期宏）与口径统一（指标字典）比 SQL 技巧更能治本，拆分层级要在数仓模型设计里提前定

**减分项：**
- 只会说「拆成多个视图」，讲不清视图 vs 物化视图的代价差异
- 造数 SQL 跑不通（generate_series 语法错、主键重复、随机数分布不可控）
- 不考虑外键一致性和数据分布，造出来的数据没法用于压测
- 不写 SQL 只讲思路

**解答：**
重构思路是「按复用点分层，而不是按人习惯分段」。第一步把重复定义的「活跃用户」、订单明细级指标抽成视图（口径单点维护）；第二步把重计算且稳定不变的轻度汇总落成物化视图（按天增量刷新），报表只消费汇总结果——300 行的嵌套就塌缩成「查两张小表」；第三步日期参数统一成会话变量或调度参数，杜绝散落。视图是「SQL 宏」，查询时展开、不存结果，口径经常改时用它；物化视图存结果、查询快，但要负担刷新策略（全量/增量/过期判定），两者不是替代关系。造数用 `generate_series` 一行一亿级都没问题，关键是分布要像线上：金额用对数分布更像真实客单，时间按「越近越多」衰减，user_id 落在用户表范围内保证外键有效：

```sql
-- 分层重构示意：口径单点维护
CREATE OR REPLACE VIEW v_active_users AS
SELECT DISTINCT user_id FROM login_log
WHERE login_time >= CURRENT_DATE - INTERVAL '30 day';

-- 报表层只做组装，不再重复定义活跃用户
SELECT 'active' AS seg, COUNT(*) FROM v_active_users
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
WHERE user_id IN (SELECT user_id FROM v_active_users) AND created_at >= CURRENT_DATE - 1;

-- 造数：1000 万订单，用户 10 万，金额对数分布，时间近多远少
INSERT INTO orders (id, user_id, amount, created_at)
SELECT g,
       (g * 7919) % 100000 + 1,                        -- 伪随机但可复现的用户分布
       ROUND((EXP(RANDOM() * 5.5))::numeric, 2),       -- 对数分布：0~245 元，多数小额
       now() - (RANDOM() ^ 1.5 * INTERVAL '365 day')   -- ^1.5 让近期数据更密
FROM generate_series(1, 10000000) g;
```

实践中的坑：一是视图嵌套超过三四层后优化器难以下推谓词，视图「复用」要克制，优先用物化视图截断层级；二是物化视图的刷新要纳入调度并校验，忘记刷新出的「旧数据」比没有数据更危险——刷新失败要告警而不是静默；三是造数不能只求「行数够」，外键要落在维表范围内（上例 mod 100000 保证 user_id ≤ 10 万）、主键唯一、时间窗口覆盖分区边界，否则 join 全空、压测结论全废；四是参数化：把 `:biz_date` 这类参数收口到一处（调度系统变量或 `SET` 语句），报表评审时一眼能看出口径位置；五是最佳实践永远是「指标字典 + 数据建模在前，SQL 在后」——报表 SQL 长到 300 行往往是数仓模型缺层的信号，治本在模型而不是再拆一个视图。

**延伸考点：** 物化视图的增量刷新怎么判断哪些分区过期（如上游补数）？造数时如何保证「用户消费金额分布」与线上一致，用什么 SQL 校验造出来的分布曲线？