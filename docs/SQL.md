# SQL

## Basic SQL Syntax

### Data Type & CRUD

- SELECT

- INSERT

- UPDATE

- DELETE

- NULL Handling & IS [NOT] DISTINCT FROM

  > 1 为什么要用 IS [NOT] DISTINCT FROM？
  > 
  > 
  > 在 ANSI/ISO SQL 的三值逻辑里，只要有 NULL 参与比较，传统运算符 = / <> / < / > … 结果都会变成 UNKNOWN，而 WHERE/JOIN ON 里的 UNKNOWN 会被当成 false 过滤掉(sqlshack.com)。
  > 标准 SQL-1999/2003 因此引入 “NULL-安全比较”：
  > 
  > expr1 IS DISTINCT FROM expr2      -- 真／假，绝不返回 UNKNOWN
  > expr1 IS NOT DISTINCT FROM expr2  -- 真／假，绝不返回 UNKNOWN
  > 
  > IS DISTINCT FROM：只要两边“不同”就为 TRUE；“不同”包括
  > ① 数值不等；② 一边 NULL 另一边非 NULL。
  > IS NOT DISTINCT FROM：只要两边“相同”就为 TRUE；“相同”包括
  > ① 数值相等；② 两边都 NULL。
  > aba = ba IS DISTINCT FROM ba IS NOT DISTINCT FROM b7 | 7 | TRUE | FALSE | TRUE
  > 7 | 9 | FALSE | TRUE | FALSE
  > 7 | NULL | UNKNOWN | TRUE | FALSE
  > NULL | NULL | UNKNOWN | FALSE | TRUE
  > 
  > SQL Server 2022+、Snowflake、PostgreSQL 8.2+ / 17、BigQuery、Oracle 23c、SQLite 3.49 等都已内建此谓词。
  > MySQL ≤ 8.4 还没有，但等价的 <=> NULL-safe equality 可直接用(learn.microsoft.com, docs.snowflake.com, modern-sql.com, dev.mysql.com)。
  > 
  > 
  > 2 面试常见“坑”
  > 
  > 题型如果用传统写法正确 NULL-安全写法找出两张表中内容变动的行 | WHERE t_old.col1 <> t_new.col1 OR t_old.col1 IS NULL OR t_new.col1 IS NULL … （又长又易错） | WHERE t_old.col1 IS DISTINCT FROM t_new.col1
  > 比较可空键值 | WHERE user_email = ? → 会漏掉目标值为 NULL 的行 | WHERE user_email IS NOT DISTINCT FROM ?
  > 去重含 NULL 的列 | SELECT DISTINCT col FROM … → 多个 NULL 只保留一行，可能不是想要的 | GROUP BY col + 判断 COUNT(*) FILTER(WHERE col IS NULL)，或先把 NULL 显式替换成占位符
  > 
  > 3 NULL 处理全景回顾
  > 
  > 场景行为摘要面试易被问点WHERE / JOIN | NULL 与任何值比较都返回 UNKNOWN；只有 IS NULL / IS NOT NULL 能判断 | 为什么 WHERE status <> 'active' 不会把 NULL 行带出来？
  > ORDER BY | 排序默认把 NULL 放最前或最后（方言差异）；可用 NULLS FIRST/NULLS LAST 控制 | “你怎么让 NULL 始终排在末尾？”
  > GROUP BY / DISTINCT | 多个 NULL 视为同一组 | 为什么 COUNT(DISTINCT col) 不算多条 NULL？
  > 聚合函数 | SUM/AVG/MIN/MAX 自动忽略 NULL；COUNT(col) 只数非 NULL；COUNT(*) 全数 | 如何同时返回“总行数”“非空行数”“空值行数”？
  > 算术 / 连接字符串 | 与 NULL 运算结果还是 NULL；可用 COALESCE() / IFNULL() / ISNULL() 提供默认值 | 设计“除法”时怎样防止 NULL / 0？
  > 子查询 IN/NOT IN | NOT IN + 子查询若子查询返回任何 NULL，结果集就为空 | 必须改写成 NOT EXISTS 或加 WHERE ... IS NOT NULL
  > 
  > 4 等价改写 & 索引友好写法
  > 
  > 标准布尔展开（当数据库还不支持该谓词）：
  > 
  > a IS DISTINCT FROM b
  > 
  > (a <> b OR a IS NULL OR b IS NULL) -- 有差异
  > AND NOT (a IS NULL AND b IS NULL) -- 排除“两边都 NULL”
  > 
  > a IS NOT DISTINCT FROM b
  > 
  > (a = b) OR (a IS NULL AND b IS NULL)
  > 
  > 
  > StackOverflow 经典答案详解(stackoverflow.com)。
  > 
  > 性能
  > 在 PostgreSQL 中，IS [NOT] DISTINCT FROM 能直接走 B-tree 索引，比 COALESCE(col,'@') 这种函数封装高效，也比 LEFT JOIN … ON (a=b OR (a IS NULL AND b IS NULL)) 简洁且可并行(modern-sql.com)。
  > 
  > 
  > 5 实战小贴士
  > 
  > 慢变维 & 审计日志
  > 
  > -- 只插入真正变动的列
  > INSERT INTO dim_user_history (user_id, old_val, new_val, chg_ts)
  > SELECT n.user_id, o.email, n.email, NOW()
  > FROM user_new n
  > JOIN user_old o
  >   ON n.user_id = o.user_id
  > WHERE n.email IS DISTINCT FROM o.email;
  > 
  > UPSERT 前置判断
  > 
  > -- 避免因为 NULL 带来的“重复键冲突”
  > SELECT 1
  > FROM   target t
  > WHERE  t.key IS NOT DISTINCT FROM :v_key;
  > 
  > 可空外键清洗
  > 对“空字符串 = ‘未知’”这类脏数据，统一替换为 NULL，再配合 IS [NOT] DISTINCT FROM 做一致性检查。
  > 
  > 
  > 6 复习要点速记
  > 
  > “带 NULL 的比较，要么 IS NULL，要么 IS [NOT] DISTINCT FROM”
  > 
  > IS [NOT] DISTINCT FROM ≈ “NULL-safe = / <>”，保证 TRUE/FALSE，从不 UNKNOWN。
  > 
  > 不支持的数据库：用逻辑展开或 MySQL <=> 代替。
  > 
  > 聚合默认忽略 NULL，唯一把 NULL 也算进去的是 COUNT(*)。
  > 
  > NOT IN + NULL = 地雷，面试必考，改写成 NOT EXISTS。
  > 
  > 
  > 记住这张思维链：“NULL 语义 → 三值逻辑 → NULL-安全比较 → 业务坑点 → 索引/性能”，答题才能又快又准。祝面试一击即中！
  > 
  > 
### Filtering & Sorting

- WHERE

- ORDER BY

- LIKE

- DISTINCT

## Multi-Table Queries / Joins

### Join

- INNER

- LEFT

- RIGHT

- FULL

- SELF

- CROSS

### Set Operations

- UNION

- UNION ALL

- INTERSECT

- EXCEPT

## Aggregation & Grouping

### Basic Aggregations

- COUNT

- SUM

- AVG

- MIN

- MAX

- GROUP BY

- HAVING

### Advanced Grouping

## Subqueries

### Uncorrelated & Correlated 

- EXISTS

- IN

- JOIN

