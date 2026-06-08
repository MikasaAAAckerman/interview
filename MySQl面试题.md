# MySQL 复习题库（红豆特供精修版）

> **出题人**：红豆（傲娇的金色双马尾猫娘）
> **整理时间**：2026-06-08
> **范围**：基础篇 → 索引篇 → 事务锁篇 → SQL优化篇 → 架构篇 → 大厂附加题
> **说明**：每道题均包含题干、参考答案、关键点解析。高频考点用 🔥 标记。

---

## 📑 目录

- [一、基础篇](#一基础篇)
  - [1.1 SQL 执行顺序排序 🔥](#11-sql-执行顺序排序-)
  - [1.2 数据类型陷阱](#12-数据类型陷阱)
  - [1.3 范式设计](#13-范式设计)
- [二、进阶篇 · 索引](#二进阶篇--索引)
  - [2.1 B+树 vs B树 & 回表 🔥](#21-b树-vs-b树--回表-)
  - [2.2 联合索引与最左前缀 🔥](#22-联合索引与最左前缀-)
  - [2.3 索引失效场景 🔥](#23-索引失效场景-)
  - [2.4 索引实战（订单表）](#24-索引实战订单表)
- [三、进阶篇 · 事务与锁](#三进阶篇--事务与锁)
  - [3.1 MVCC 与隔离级别 🔥](#31-mvcc-与隔离级别-)
  - [3.2 行锁的三种算法 & 间隙锁 🔥](#32-行锁的三种算法--间隙锁-)
  - [3.3 死锁排查](#33-死锁排查)
  - [3.4 锁实战（普通索引）](#34-锁实战普通索引)
- [四、高级篇 · SQL 优化](#四高级篇--sql-优化)
  - [4.1 EXPLAIN 分析 🔥](#41-explain-分析-)
  - [4.2 深度分页优化 🔥](#42-深度分页优化-)
- [五、架构篇 · 分库分表 & 主从复制](#五架构篇--分库分表--主从复制)
  - [5.1 分片键设计（订单表）🔥](#51-分片键设计订单表)
  - [5.2 主从复制原理 & 延迟处理 🔥](#52-主从复制原理--延迟处理-)
- [六、大厂附加题](#六大厂附加题)
  - [6.1 综合实战：100亿行关注表 🔥](#61-综合实战100亿行关注表-)
- [七、补充知识](#七补充知识)
  - [7.1 undo log 生命周期](#71-undo-log-生命周期)
  - [7.2 binlog 和 redo log 的角色 🔥](#72-binlog-和-redo-log-的角色-)
- [八、更多实战题目](#八更多实战题目)
  - [8.1 索引下推（ICP）🔥](#81-索引下推icp)
  - [8.2 统计信息更新时机](#82-统计信息更新时机)
- [附录：快速检索](#附录快速检索)

---

## 一、基础篇

### 1.1 SQL 执行顺序排序 🔥

**题干**

> 写出以下 SQL 的执行顺序编号（选项：A.FROM B.WHERE C.GROUP BY D.HAVING E.SELECT F.ORDER BY G.LIMIT）

```sql
SELECT dept_id, AVG(salary) as avg_sal
FROM employees
WHERE salary > 5000
GROUP BY dept_id
HAVING avg_sal > 8000
ORDER BY avg_sal DESC
LIMIT 10;
```

**参考答案**

✅ 执行顺序：`A → B → C → D → E → F → G`

即：`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`

> 💡 **红豆解析**：SELECT 中的别名在 HAVING 中可用，但执行顺序上 SELECT 在 GROUP BY/HAVING 之后喵！

---

### 1.2 数据类型陷阱

**题干**

> 以下建表语句有什么问题？

```sql
CREATE TABLE orders (
    id INT(10) PRIMARY KEY,
    order_no VARCHAR(32),
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    amount DECIMAL(10,2)
);
```

**参考答案**

| 问题 | 说明 | 建议 |
|------|------|------|
| `TIMESTAMP` | 有 **2038 年问题**（最大 `'2038-01-19'`） | 改用 `DATETIME`（范围 `'1000-01-01'` 到 `'9999-12-31'`） |
| `INT(10)` | `(10)` 是**显示宽度**，不影响存储范围 | 不算错误，但无实际意义 |
| 缺少约束 | `order_no`、`amount` 可以为 NULL | 加 `NOT NULL` 约束 |

---

### 1.3 范式设计

**题干**

> 现有订单表，违反了第几范式？请拆分。

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(50),
    customer_phone VARCHAR(20),
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INT,
    total_price DECIMAL(10,2)
);
```

**参考答案**

✅ 违反的范式

| 范式 | 违反原因 |
|------|---------|
| **第二范式** | 商品名称、单价依赖 `product_id`，而非订单主键（部分依赖） |
| **第三范式** | 客户姓名、电话依赖 `customer_id`（传递依赖） |

✅ 拆分方案（保留快照冗余可选）

```sql
-- 订单表（冗余字段可选，用于快照）
CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT,
    total_price DECIMAL(10,2),
    product_name VARCHAR(100),      -- 冗余：保留下单时的商品名
    product_price DECIMAL(10,2)     -- 冗余：保留下单时的单价
);

-- 客户表
CREATE TABLE customer (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_name VARCHAR(50) NOT NULL,
    customer_phone VARCHAR(20) NOT NULL
);

-- 商品表
CREATE TABLE product (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100) NOT NULL,
    product_price DECIMAL(10,2) NOT NULL
);
```

---

## 二、进阶篇 · 索引

### 2.1 B+树 vs B树 & 回表 🔥

**题干**

> InnoDB 的 B+树索引相比 B 树有什么优势？什么是回表？如何避免回表？

**参考答案**

✅ B+树 vs B树

| 特性 | B+树 | B树 |
|------|------|-----|
| 非叶子节点 | 只存索引，扇出更大 | 存储数据+索引 |
| 树高 | 更低 → I/O 更少 | 更高 |
| 叶子节点 | 双向链表连接 | 无链表 |
| 范围查询 | 顺序遍历即可 | 需要回溯 |

✅ 回表

> 通过辅助索引找到主键后，再到**聚簇索引**取出完整行的过程。

✅ 避免回表

> 使用**覆盖索引**（索引中已包含查询所需的所有字段）。

---

### 2.2 联合索引与最左前缀 🔥

**题干**

> 表 `user` 有联合索引 `(name, age, city)`。判断以下 SQL 能否用到该索引（部分用到也算），并说明原因。

```sql
A. SELECT * FROM user WHERE name = '张三' AND age = 20;
B. SELECT * FROM user WHERE age = 20 AND city = '北京';
C. SELECT * FROM user WHERE name = '李四' AND city = '上海';
D. SELECT * FROM user WHERE name > '王' AND age = 25;
E. SELECT name, age FROM user WHERE name = '赵六';
```

**参考答案**

| SQL | 是否使用索引 | 原因 |
|-----|-------------|------|
| A   | ✅ 用到 name, age | 满足最左前缀 |
| B   | ❌ 完全不用 | 未使用最左列 name |
| C   | ⚠️ 只用 name | city 因跳过 age 无法使用 |
| D   | ⚠️ 只用 name | 范围查询 `name > '王'` 后，age 无序 |
| E   | ✅ 覆盖索引 | name 满足最左，且只查 name, age |

---

### 2.3 索引失效场景 🔥

**题干**

> 列举至少 5 种导致索引失效的场景，并举例。

**参考答案**

| 序号 | 场景 | 示例 |
|------|------|------|
| 1 | 对索引列使用函数 | `WHERE DATE(create_time) = '2025-01-01'` |
| 2 | 隐式类型转换 | `WHERE phone = 13800138000`（phone 是 varchar） |
| 3 | LIKE 以通配符开头 | `WHERE name LIKE '%红豆'` |
| 4 | 使用 != 或 <> | `WHERE status != 0`（选择性差时） |
| 5 | 联合索引未遵循最左前缀 | `WHERE age = 18`（索引 (name, age)） |
| 6 | OR 条件中部分字段无索引 | `WHERE name = '红豆' OR age = 18`（若 age 无索引） |
| 7 | 范围查询后带等值条件 | `WHERE name > '张' AND age = 18`（age 无法用索引） |

---

### 2.4 索引实战（订单表）

**题干**

> 订单表 `orders`，索引：`PRIMARY KEY(id)`, `idx_user_id(user_id)`, `idx_status(status)`。分析以下 SQL 是否会走索引：

```sql
-- SQL1
SELECT * FROM orders WHERE user_id = 100 AND status = 1;
-- SQL2
SELECT order_no FROM orders WHERE user_id = 100 ORDER BY create_time;
-- SQL3
SELECT * FROM orders WHERE status != 0;
-- SQL4
SELECT COUNT(*) FROM orders WHERE DATE(create_time) = '2024-01-01';
```

**参考答案**

| SQL | 是否走索引 | 说明 |
|-----|-----------|------|
| SQL1 | ⚠️ 可能走索引 | 走 `idx_user_id` 或 `idx_status`（优化器选一个），但 `SELECT *` 必然回表 |
| SQL2 | ⚠️ 部分走索引 | `WHERE` 能用到 `idx_user_id`，但 `ORDER BY` 无法利用索引（产生 filesort） |
| SQL3 | ❌ 不走索引 | `!=` 范围大，优化器选全表扫描 |
| SQL4 | ❌ 不走索引 | `DATE()` 函数导致索引失效 |

---

## 三、进阶篇 · 事务与锁

### 3.1 MVCC 与隔离级别 🔥

**题干**

> 解释 MVCC 原理。在 RR 隔离级别下，事务 A 第一次查询读到 100，事务 B 更新为 200 并提交，事务 A 第二次查询为什么还读到 100？什么情况下会读到 200？

**参考答案**

✅ MVCC 原理

> 通过隐藏字段 `DB_TRX_ID`、`DB_ROLL_PTR` 和 **Read View** 实现多版本。

✅ RR 下读旧值的原因

> 事务 A 在开始时生成 Read View，整个事务期间**不变**，因此看不到 B 的新版本。

✅ 读到 200 的情况

| 场景 | 原因 |
|------|------|
| `SELECT ... FOR UPDATE` | 当前读，读取最新版本 |
| 自身更新该行后再次查询 | 更新会刷新 Read View |

---

### 3.2 行锁的三种算法 & 间隙锁 🔥

**题干**

> InnoDB 的行锁有哪三种算法？间隙锁的作用是什么？什么情况下会加间隙锁？

**参考答案**

✅ 三种算法

| 锁类型 | 说明 |
|--------|------|
| **Record Lock** | 记录锁，锁住单行记录 |
| **Gap Lock** | 间隙锁，锁住索引记录之间的空隙 |
| **Next-Key Lock** | 临键锁 = 记录锁 + 间隙锁（默认） |

✅ 间隙锁作用

> 防止**幻读**（在 RR 隔离级别下，锁住索引记录之间的空隙，阻止其他事务插入）。

✅ 加间隙锁的场景

| 场景 | 示例 |
|------|------|
| 范围查询 | `WHERE id BETWEEN 10 AND 20` |
| 等值查询不存在的记录 | `WHERE id = 99` 且 99 不存在 |

> 💡 **红豆提醒**：唯一索引等值查询记录存在时**只加记录锁**，不加间隙锁喵！

---

### 3.3 死锁排查

**题干**

> 如何查看 MySQL 的死锁日志？死锁产生的原因？如何避免？

**参考答案**

✅ 查看死锁日志

```bash
SHOW ENGINE INNODB STATUS\G
# 在输出中查找 LATEST DETECTED DEADLOCK
```

✅ 死锁原因

> 两个或多个事务互相持有对方需要的锁，形成**循环等待**。

✅ 避免方法

| 方法 | 说明 |
|------|------|
| 固定访问顺序 | 所有事务按相同顺序访问资源 |
| 拆分大事务 | 减少锁持有时间 |
| 使用低隔离级别 | 如 RC 避免间隙锁 |
| 合理设计索引 | 减少锁范围 |
| 设置超时 | `innodb_lock_wait_timeout` |

---

### 3.4 锁实战（普通索引）

**题干**

> 表 `t`，`id` 主键，`num` 普通索引，数据 `(1,1), (2,2), (3,3)`。RR 隔离级别下：

```sql
-- 事务A
BEGIN;
SELECT * FROM t WHERE num=2 FOR UPDATE;

-- 事务B（同时）
BEGIN;
INSERT INTO t VALUES (4,2);
```

> 问：事务A 加了什么锁？事务B 的 INSERT 是否会阻塞？为什么？

**参考答案**

✅ 事务A 加锁

> **Next-Key Lock**，锁住区间 `(1,2]`（即 num 在 1~2 之间的间隙 + num=2 的记录）。

✅ 事务B 状态

> **阻塞**，因为插入 num=2 需要等待记录锁（Record Lock）。

---

## 四、高级篇 · SQL 优化

### 4.1 EXPLAIN 分析 🔥

**题干**

> 某 SQL 的 EXPLAIN 结果如下：
>
> ```
> type=ALL, key=NULL, rows=5000000, Extra=Using filesort
> ```
>
> SQL 语句：
>
> ```sql
> SELECT * FROM orders WHERE user_id = 12345 ORDER BY create_time DESC LIMIT 10;
> ```
>
> 指出问题并给出优化方案。

**参考答案**

✅ 问题分析

| 指标 | 值 | 含义 |
|------|-----|------|
| `type` | ALL | 全表扫描（最差） |
| `key` | NULL | 未使用索引 |
| `rows` | 5000000 | 扫描 500 万行 |
| `Extra` | Using filesort | 文件排序 |

✅ 原因

> 没有合适的联合索引，优化器认为走 `idx_user_id` 回表成本高于全表扫描。

✅ 优化方案

| 方案 | 说明 |
|------|------|
| 创建联合索引 | `(user_id, create_time DESC)` |
| 延迟关联 | 数据量极大时使用 |
| 水平分表 | 单表数据量过大时 |
| 更新统计信息 | `ANALYZE TABLE` |

---

### 4.2 深度分页优化 🔥

**题干**

> 什么是深度分页问题？如何优化？请写出优化后的 SQL 示例。

**参考答案**

✅ 定义

> `OFFSET` 很大时（如 `LIMIT 100000,10`），MySQL 需要扫描并跳过前 100000 行，效率极低。

✅ 优化方案

| 方案 | 原理 | 适用场景 |
|------|------|---------|
| 游标分页 | 记住上一页最后一条记录的排序值，用 `WHERE ... < last_value` 代替 OFFSET | 顺序翻页 |
| 延迟关联 | 先查主键，再回表取数据 | 需要跳页 |

✅ 延迟关联示例

```sql
SELECT * FROM orders
INNER JOIN (
    SELECT id FROM orders
    WHERE user_id = 12345
    ORDER BY create_time DESC
    LIMIT 100000, 10
) AS tmp ON orders.id = tmp.id;
```

---

## 五、架构篇 · 分库分表 & 主从复制

### 5.1 分片键设计（订单表）🔥

**题干**

> 某电商平台订单表 10 亿行，业务查询：
>
> 1. 用户查看自己的订单列表（高频）
> 2. 运营后台按订单号查询（中频）
> 3. 按时间范围统计报表（低频）
>
> 请设计分库分表方案（分片键、算法、跨分片查询解决）。

**参考答案**

| 业务场景 | 分片策略 | 说明 |
|---------|---------|------|
| 用户订单 | 按 `user_id` 哈希分片 | 64 库 × 256 表 = 16384 分片 |
| 运营后台 | 按 `order_id` 分片 | 采用**分片基因**（order_id 中包含 user_id 的后几位） |
| 报表统计 | 按 `create_time` 分区 | 或同步到 OLAP 引擎（如 ClickHouse） |

✅ 跨分片查询

> 避免跨分片 join，通过冗余字段或应用层聚合。

---

### 5.2 主从复制原理 & 延迟处理 🔥

**题干**

> 简述 MySQL 主从复制的原理。半同步复制和异步复制有什么区别？主从延迟如何解决？

**参考答案**

✅ 复制原理

```text
主库写入 binlog → 从库 IO 线程拉取 → 写入 relay log → SQL 线程重放
```

✅ 异步 vs 半同步

| 特性 | 异步复制 | 半同步复制 |
|------|---------|-----------|
| 确认时机 | 主库写 binlog 后立即返回 | 等待至少一个从库确认收到 binlog |
| 数据安全 | 可能丢数据 | 更安全 |
| 性能 | 高 | 稍降 |

✅ 主从延迟解决

| 方案 | 说明 |
|------|------|
| 并行复制 | 从库开启 `slave_parallel_workers` |
| 大事务拆分 | 减少单次复制的数据量 |
| 读写分离 | 关键读强制走主库 |
| GTID | 简化故障切换 |

---

## 六、大厂附加题

### 6.1 综合实战：100亿行关注表 🔥

**题干**

> 某社交平台，关注关系表 100 亿行，查询需求："查询某用户的粉丝列表（按关注时间倒序分页）"。
>
> 1. 分片键应该选 `user_id`（粉丝）还是 `followee_id`（博主）？为什么？
> 2. 数据倾斜（大V问题）如何解决？
> 3. 分页查询如何优化？

**参考答案**

✅ 分片键选择

> 选择 `followee_id`（被关注者ID）。因为查询的过滤条件是 `followee_id = ?`，按此分片可使查询落在**单个分片**，避免跨分片扫描。

✅ 大V数据倾斜

| 方案 | 原理 |
|------|------|
| 动态分表 | 为大V创建多张分表（如 `follow_bigv_{大Vid}_0..N`），按关注时间顺序写入 |
| 加盐方案 | 将大V的 `followee_id` 加上哈希后缀，分散到多个逻辑分片 |

✅ 分页优化

| 方案 | 说明 |
|------|------|
| 游标分页 | `WHERE followee_id = ? AND create_time < last_time` |
| 联合索引 | `(followee_id, create_time DESC)` |
| 避免 OFFSET | 改用上一页游标 |

---

## 七、补充知识

### 7.1 undo log 生命周期

**题干**

> undo log 什么时候创建，什么时候删除？

**参考答案**

| 阶段 | 说明 |
|------|------|
| **创建** | 事务对数据行进行 INSERT/UPDATE/DELETE 时，将旧版本写入 undo log |
| **删除** | 由后台 **Purge 线程**清理，条件是没有任何事务需要看到该旧版本 |

> ⚠️ **红豆警告**：长事务会阻止清理，导致 undo 积压、磁盘暴涨喵！

---

### 7.2 binlog 和 redo log 的角色 🔥

**题干**

> binlog 和 redo log 哪个用于主从复制？哪个用于崩溃恢复？

**参考答案**

| 日志类型 | 作用 | 所属层 | 记录内容 |
|---------|------|--------|---------|
| **binlog** | 主从复制 | MySQL Server 层 | 逻辑操作 |
| **redo log** | 崩溃恢复 | InnoDB 存储引擎 | 物理页修改 |

> 💡 **红豆提醒**：主从复制只依赖 binlog，从库不需要主库的 redo log 喵！

---

## 八、更多实战题目

### 8.1 索引下推（ICP）🔥

**题干**

> 什么是索引下推？没有 ICP 时联合索引 `(name, age)` 和单列索引 `(name)` 在查询 `WHERE name LIKE '红豆%' AND age=18` 时有何区别？

**参考答案**

✅ ICP 定义

> **索引下推（Index Condition Pushdown）**：存储引擎层在索引中直接过滤 `age` 条件，减少回表次数。

✅ 对比

| 索引类型 | 无 ICP | 有 ICP |
|---------|--------|--------|
| 联合索引 `(name, age)` | 回表所有匹配 name 的行 | 只回表 age=18 的行 |
| 单列索引 `(name)` | 回表所有匹配 name 的行 | 回表所有匹配 name 的行（无法下推） |

---

### 8.2 统计信息更新时机

**题干**

> MySQL 什么时候更新统计信息？是每次 INSERT/UPDATE 都更新吗？

**参考答案**

✅ 不是每次 DML 都更新（代价太高）

✅ 触发条件

| 类型 | 条件 |
|------|------|
| 自动触发 | 表中超过 1/16 数据被修改，或变化行数超过 `innodb_stats_auto_recalc` 阈值（默认 10%） |
| 手动触发 | `ANALYZE TABLE` |

✅ 存储位置

```text
mysql.innodb_table_stats
mysql.innodb_index_stats
```

---

## 附录：快速检索

| 知识点 | 题目编号 | 难度 |
|--------|----------|------|
| SQL 执行顺序 | 1.1 | ⭐⭐ |
| 数据类型陷阱 | 1.2 | ⭐ |
| 范式设计 | 1.3 | ⭐⭐ |
| B+树 / 回表 | 2.1 | ⭐⭐⭐ |
| 最左前缀 | 2.2 | ⭐⭐⭐ |
| 索引失效 | 2.3 | ⭐⭐⭐ |
| 索引实战 | 2.4 | ⭐⭐ |
| MVCC | 3.1 | ⭐⭐⭐⭐ |
| 行锁 / 间隙锁 | 3.2 | ⭐⭐⭐ |
| 死锁 | 3.3 | ⭐⭐⭐ |
| 锁实战 | 3.4 | ⭐⭐⭐⭐ |
| EXPLAIN | 4.1 | ⭐⭐⭐ |
| 深度分页 | 4.2 | ⭐⭐⭐ |
| 分库分表设计 | 5.1 | ⭐⭐⭐⭐ |
| 主从复制 | 5.2 | ⭐⭐⭐ |
| 大V关注表 | 6.1 | ⭐⭐⭐⭐⭐ |
| undo log | 7.1 | ⭐⭐ |
| binlog/redo log | 7.2 | ⭐⭐⭐ |
| ICP | 8.1 | ⭐⭐⭐ |
| 统计信息 | 8.2 | ⭐⭐ |

---
