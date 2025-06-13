下面给出一份 **“电商-分析” 端到端脚本**（伪数据 + 查询示例），几乎把前面思维导图 **三级节点** 提到的所有关键 SQL 语法都串进来了。你可以整段复制到 PostgreSQL 16 / Snowflake / BigQuery 等较新方言里跑（个别方言差异已在注释说明），或按需拆分到面试白板中演示。

> **结构**
>
> 1. 业务场景与表设计
> 2. 建表 & 数据准备（含索引、分区、JSON 列）
> 3. 40 段示例查询（📌标签 = 三级节点关键语法）
> 4. 事务+锁 & 调优补充
> 5. 练手指令

---

## 1 业务设定

* **用户表 `dim_user`**：注册信息、渠道、层级 (递归例子用)。
* **商品表 `dim_product`**：品类层级（用于自引用层级）。
* **订单事实 `fact_order`**：一行一笔支付。
* **事件事实 `fact_event`**：埋点数据（`JSONB`），用于 DAU / 漏斗 / A/B。
* **时间表 `dim_date`**：用于分区与日期维度分析。

---

## 2 建表 & 样本数据

```sql
-- 2.1 维度表
CREATE TABLE dim_user (
    user_id       BIGINT PRIMARY KEY,
    email         TEXT,
    signup_ts     TIMESTAMP,
    channel       TEXT,
    parent_user   BIGINT,          -- 邀请人
    CONSTRAINT fk_parent FOREIGN KEY(parent_user)
        REFERENCES dim_user(user_id)
);

CREATE TABLE dim_product (
    product_id    BIGINT PRIMARY KEY,
    product_name  TEXT,
    category_id   BIGINT,
    parent_cat    BIGINT  -- 递归分类
);

-- 2.2 分区 + JSONB 事实表
CREATE TABLE fact_event (
    event_id      BIGINT,
    user_id       BIGINT,
    event_date    DATE,
    event_ts      TIMESTAMP,
    event_type    TEXT,
    payload       JSONB,          -- 半结构化
    CONSTRAINT pk_event PRIMARY KEY(event_id, event_date)  -- ☆分区键
) PARTITION BY RANGE (event_date);       -- PostgreSQL 10+ / BigQuery 按列分区

CREATE TABLE fact_order (
    order_id      BIGINT,
    user_id       BIGINT,
    order_ts      TIMESTAMP,
    amount        NUMERIC(12,2),
    status        TEXT,
    coupon_code   TEXT
);

-- 2.3 索引示例
CREATE INDEX idx_evt_user_date ON fact_event (user_id, event_date);
CREATE INDEX idx_order_status_partial ON fact_order (order_ts)
        WHERE status = 'PAID';          -- 部分索引
CREATE INDEX idx_event_json_amount ON fact_event
        USING GIN (payload jsonb_path_ops);   -- JSON 索引

-- 2.4 样本行（仅演示）
INSERT INTO dim_user VALUES
  (1,'alice@acme.com','2025-01-02 10:00','ads',NULL),
  (2,'bob@acme.com'  ,'2025-01-03 12:00','organic',1),
  (3,NULL             ,'2025-01-04 08:00','ads',1);  -- NULL 邮箱测试

-- 订单 & 事件可用 COPY/LOAD 或手动插几行
```

---

## 3 查询大全（40 个片段）

> **写法**：每段/**📌标签** 说明对应的三级节点关键语法；必要时只保留核心 SELECT 句，方便面试手写。

### 3.1 基础语法 / 过滤排序 / CRUD

```sql
-- ① 📌SELECT + WHERE + LIKE + ORDER BY + NULLS LAST
SELECT user_id, email
FROM   dim_user
WHERE  channel LIKE 'ad%'        -- 模式匹配
ORDER  BY signup_ts DESC NULLS LAST;
```

```sql
-- ② 📌INSERT / RETURNING
INSERT INTO dim_product (product_id, product_name, category_id, parent_cat)
VALUES (101, 'Noise-Canceling Headphones', 10, NULL)
RETURNING product_id;
```

```sql
-- ③ 📌UPDATE with COALESCE(NULL 处理)
UPDATE dim_user
SET    email = COALESCE(email,'unknown+'||user_id||'@example.com')
WHERE  email IS NULL;
```

```sql
-- ④ 📌DELETE + USING 子句
DELETE FROM fact_order o
USING dim_user u
WHERE o.user_id = u.user_id
  AND u.signup_ts < '2024-01-01';
```

### 3.2 多表查询

```sql
-- ⑤ 📌INNER JOIN
SELECT o.order_id, u.email, o.amount
FROM   fact_order o
JOIN   dim_user u USING (user_id);
```

```sql
-- ⑥ 📌LEFT JOIN（空值行演示）
SELECT u.user_id, o.order_id
FROM   dim_user u
LEFT   JOIN fact_order o USING (user_id);
```

```sql
-- ⑦ 📌FULL JOIN（比较库存与销量）
SELECT p.product_id, p.product_name, s.sold_cnt
FROM   dim_product p
FULL   JOIN (SELECT product_id, COUNT(*) sold_cnt
             FROM fact_order GROUP BY product_id) s
       USING (product_id);
```

```sql
-- ⑧ 📌UNION / INTERSECT / EXCEPT
SELECT user_id FROM dim_user
UNION
SELECT user_id FROM fact_order;      -- 所有出现过的用户

SELECT user_id FROM dim_user
EXCEPT
SELECT user_id FROM fact_order;      -- 从未下单的注册用户
```

### 3.3 聚合 & 分组

```sql
-- ⑨ 📌GROUP BY + HAVING
SELECT status, COUNT(*) cnt
FROM   fact_order
GROUP  BY status
HAVING COUNT(*) > 10;
```

```sql
-- ⑩ 📌ROLLUP（多维报表）
SELECT channel, DATE_TRUNC('month', signup_ts) AS mth,
       COUNT(*) regs
FROM   dim_user
GROUP  BY ROLLUP (channel, mth);    -- 共 1+N+1 行
```

```sql
-- ⑪ 📌CUBE（维度交叉）
SELECT channel, status, COUNT(*) orders
FROM   dim_user u
JOIN   fact_order o USING (user_id)
GROUP  BY CUBE (channel, status);
```

```sql
-- ⑫ 📌GROUPING SETS
SELECT channel, DATE(signup_ts) AS d, COUNT(*)
FROM   dim_user
GROUP  BY GROUPING SETS
         ((channel, d), (channel), ());   -- 明细 + 按渠道 + 总计
```

### 3.4 子查询 & CTE

```sql
-- ⑬ 📌EXISTS vs IN（空值陷阱）
-- A. 错误示范：如果 subquery 返回 NULL，NOT IN 全空
SELECT user_id
FROM   dim_user
WHERE  user_id NOT IN (SELECT user_id FROM fact_order WHERE coupon_code IS NULL);

-- B. 推荐：NOT EXISTS
SELECT u.user_id
FROM   dim_user u
WHERE  NOT EXISTS (SELECT 1 FROM fact_order o WHERE o.user_id = u.user_id);
```

```sql
-- ⑭ 📌IS DISTINCT FROM：比对新旧快照
SELECT *
FROM   fact_order_hist h
JOIN   fact_order_new  n USING (order_id)
WHERE  h.amount IS DISTINCT FROM n.amount;
```

```sql
-- ⑮ 📌非关联子查询 + TOP-N-Per-Group（窗口见后）
SELECT *
FROM   fact_order o
WHERE  amount = (SELECT MAX(amount)
                 FROM fact_order
                 WHERE user_id = o.user_id);
```

```sql
-- ⑯ 📌递归 CTE（邀请树）
WITH RECURSIVE referral AS (
  SELECT user_id, parent_user, 0 AS depth
  FROM   dim_user
  WHERE  user_id = 1
  UNION ALL
  SELECT u.user_id, u.parent_user, r.depth+1
  FROM   dim_user u
  JOIN   referral r ON u.parent_user = r.user_id
)
SELECT * FROM referral;
```

### 3.5 窗口函数

```sql
-- ⑰ 📌ROW_NUMBER vs RANK
SELECT order_id, user_id, amount,
       ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY amount DESC) AS rn,
       RANK()       OVER(PARTITION BY user_id ORDER BY amount DESC) AS rk
FROM   fact_order;
```

```sql
-- ⑱ 📌LAG/LEAD 留存曲线
SELECT user_id,
       event_date,
       LAG(event_date) OVER(PARTITION BY user_id ORDER BY event_date)  AS prev_date,
       DATEDIFF('day', prev_date, event_date)                          AS gap_days
FROM   fact_event
WHERE  event_type = 'login';
```

```sql
-- ⑲ 📌移动平均 + ROWS BETWEEN
SELECT DATE_TRUNC('day', order_ts) AS d,
       SUM(amount) AS daily_sales,
       AVG(SUM(amount)) OVER(ORDER BY d
                             ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM   fact_order
GROUP  BY d
ORDER  BY d;
```

```sql
-- ⑳ 📌NTILE 分箱
SELECT user_id, amount,
       NTILE(4) OVER(ORDER BY amount DESC) AS quartile
FROM   fact_order;
```

### 3.6 JSON / 半结构化 & QUALIFY

```sql
-- ㉑ 📌JSONB →> 文本抽取 + GIN 索引命中
SELECT event_id, payload->>'page'     AS page,
       (payload->'products')->>0      AS first_prod
FROM   fact_event
WHERE  payload @> '{"page":"checkout"}';
```

```sql
-- ㉒ 📌UNNEST/ARRAY_EXTRACT（BigQuery 风格演示）
SELECT fe.user_id, prod
FROM   fact_event fe,
       UNNEST(JSON_EXTRACT_ARRAY(payload,'$.products')) AS prod;
```

```sql
-- ㉓ 📌QUALIFY（Snowflake / BigQuery）：窗口过滤
SELECT user_id, order_ts, amount,
       ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY order_ts) AS rn
FROM   fact_order
QUALIFY rn = 1;         -- 每人首单
```

### 3.7 业务指标：DAU / 留存 / 漏斗 / Cohort / A-B

```sql
-- ㉔ 📌DAU
SELECT event_date, COUNT(DISTINCT user_id) dau
FROM   fact_event
WHERE  event_type = 'login'
GROUP  BY event_date;
```

```sql
-- ㉕ 📌次日留存率（窗口 + 条件聚合）
WITH first_login AS (
  SELECT user_id, MIN(event_date) AS d0
  FROM   fact_event
  WHERE  event_type = 'login'
  GROUP  BY user_id
)
SELECT d0,
       COUNT(*)                       AS new_users,
       COUNT(*) FILTER(
         WHERE EXISTS (SELECT 1
                       FROM fact_event fe2
                       WHERE fe2.user_id = fl.user_id
                         AND fe2.event_date = fl.d0 + INTERVAL '1 day'
                         AND fe2.event_type='login')) AS retained_d1
FROM   first_login fl
GROUP  BY d0;
```

```sql
-- ㉖ 📌漏斗（多次 LEFT JOIN）
WITH p AS (
  SELECT user_id,
         MIN(CASE WHEN event_type='view_product' THEN event_ts END) AS view_ts,
         MIN(CASE WHEN event_type='add_cart'     THEN event_ts END) AS cart_ts,
         MIN(CASE WHEN event_type='checkout'     THEN event_ts END) AS ck_ts
  FROM fact_event
  WHERE event_type IN ('view_product','add_cart','checkout')
  GROUP BY user_id
)
SELECT
  COUNT(*)                            AS view_cnt,
  COUNT(cart_ts)                      AS cart_cnt,
  COUNT(ck_ts)                        AS ck_cnt,
  COUNT(ck_ts)::DECIMAL / NULLIF(COUNT(*),0) AS conv_rate
FROM p;
```

```sql
-- ㉗ 📌Cohort（注册月份 × N 月留存）
WITH l AS (
  SELECT user_id,
         MIN(event_date) d0,
         DATE_TRUNC('month', MIN(event_date)) cohort_mth
  FROM   fact_event
  WHERE  event_type='login'
  GROUP  BY user_id
), login_month AS (
  SELECT l.user_id, l.cohort_mth,
         DATE_TRUNC('month', fe.event_date) AS login_mth
  FROM   l
  JOIN   fact_event fe USING (user_id)
  WHERE  fe.event_type='login'
)
SELECT cohort_mth,
       DATE_PART('month', login_mth - cohort_mth)      AS month_num,
       COUNT(DISTINCT user_id)                         AS active_users
FROM   login_month
GROUP  BY cohort_mth, month_num
ORDER  BY cohort_mth, month_num;
```

```sql
-- ㉘ 📌A/B Test 样本采集 + Stratified Sampling
WITH bucket AS (
  SELECT user_id,
         NTILE(2) OVER(ORDER BY user_id) AS grp      -- 简单哈希桶
  FROM   dim_user
)
SELECT grp,
       COUNT(*)       AS users,
       AVG(amount)    AS avg_order
FROM   bucket b
JOIN   fact_order o USING (user_id)
GROUP  BY grp;
```

### 3.8 索引 / EXPLAIN / 物化视图 / 分区裁剪

```sql
-- ㉙ 📌EXPLAIN ANALYZE + 索引命中
EXPLAIN ANALYZE
SELECT *
FROM   fact_order
WHERE  order_ts >= NOW() - INTERVAL '7 days'
  AND  status = 'PAID';         -- 走部分索引
```

```sql
-- ㉚ 📌覆盖索引示例（仅索引用字段）
SELECT user_id, amount
FROM   fact_order
WHERE  status = 'PAID';         -- idx_order_status_partial 已覆盖
```

```sql
-- ㉛ 📌物化视图（每日 GMV）
CREATE MATERIALIZED VIEW mv_daily_gmv
AS
SELECT DATE(order_ts) AS d, SUM(amount) gmv
FROM   fact_order
GROUP  BY d
WITH NO DATA;      -- 延迟刷新

-- 刷新 & 查询
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_gmv;
SELECT * FROM mv_daily_gmv WHERE d >= CURRENT_DATE - 7;
```

```sql
-- ㉜ 📌分区裁剪（WHERE 触发 RANGE partition）
SELECT COUNT(*)
FROM   fact_event
WHERE  event_date = CURRENT_DATE;
```

### 3.9 事务 & 隔离 / 死锁

```sql
-- ㉝ 📌事务块 + SERIALIZABLE 隔离（演示库存扣减）
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  UPDATE dim_product
  SET    stock = stock - 1
  WHERE  product_id = 101
    AND  stock > 0;

  INSERT INTO fact_order(order_id, user_id, order_ts, amount, status)
  VALUES (NEXTVAL('order_seq'), 2, NOW(), 199.00, 'PENDING');
COMMIT;
```

> 面试补刀：如何避免死锁？
>
> * 按固定顺序锁表
> * 使用 `SELECT … FOR UPDATE SKIP LOCKED`
> * 拆小事务、乐观锁版本号

---

## 4 调优提示一览

| 场景      | 要点                                                                     |
| ------- | ---------------------------------------------------------------------- |
| 查询慢     | `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)`；关注 Seq Scan vs Bitmap/Index Scan |
| 窗口函数    | 能用 `PARTITION BY` + `ORDER BY` 的列建复合索引                                 |
| 递归 CTE  | 加 `WHERE depth <= N` 或选用 ltree/hierarchyid                             |
| JSON 查询 | 建 GIN/JSON\_PATH\_OPS；用 `jsonb_path_query_first()`                     |
| 分区      | 尽量让 `WHERE` 带分区列；避免跨分区 `ORDER BY`                                      |

---

## 5 练手指令

1. **改写练习**：把示例中的 `LEFT JOIN … ON` 重写成 `USING` + `IS DISTINCT FROM`；再反向改写看看结果是否一致。
2. **方言替换**：把 JSON 片段换成 MySQL 8.4 (`->>'$.key'`) 或 SQL Server 2022 (`JSON_VALUE`)。
3. **Explain 对比**：在有/无索引情况下跑示例 ㉙，截图执行计划比大小。
4. **生成题库**：随机挑 5-10 段，遮住关键字，让同学补全并解释性能差异。

---

### 一句话总结

> **这一套脚本 = “SQL 面试知识图谱” 的可执行 Demo**
> —— 你可以靠它同时展示 **写得快、讲得清、调得顺** 三大能力。祝你在白板前手起刀落，Offer 到手！
