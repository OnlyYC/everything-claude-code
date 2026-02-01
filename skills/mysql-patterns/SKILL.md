---
name: mysql-patterns
description: MySQL 数据库模式：查询优化、Schema 设计、索引策略、安全配置。基于国内 MySQL 最佳实践。
---

# MySQL 模式

MySQL 最佳实践快速参考。详细指南请使用数据库审查相关工具。

## 何时启用

- 编写 SQL 查询或数据库变更脚本
- 设计数据库表结构
- 排查慢查询
- 配置连接池
- 数据库性能优化

## 快速参考

### 索引速查表

| 查询模式 | 索引类型 | 示例 |
|---------|---------|------|
| `WHERE col = value` | B-Tree（默认） | `CREATE INDEX idx ON t(col)` |
| `WHERE col > value` | B-Tree | `CREATE INDEX idx ON t(col)` |
| `WHERE a = x AND b > y` | 联合索引 | `CREATE INDEX idx ON t(a, b)` |
| `LIKE 'prefix%'` | 前缀索引 | `CREATE INDEX idx ON t(col(10))` |
| `WHERE col IN (list)` | B-Tree | `CREATE INDEX idx ON t(col)` |
| 全文搜索 | FULLTEXT | `CREATE FULLTEXT INDEX idx ON t(col)` |
| 空间数据 | SPATIAL | `CREATE SPATIAL INDEX idx ON t(geo)` |

### 数据类型快速参考

| 使用场景 | 正确类型 | 避免 |
|---------|---------|------|
| 主键 ID | `BIGINT UNSIGNED` | `INT`、UUID |
| 字符串 | `VARCHAR` | `CHAR`（除非定长） |
| 时间戳 | `DATETIME` 或 `TIMESTAMP` | 存储时区 |
| 金额 | `DECIMAL(19,4)` | `FLOAT`、`DOUBLE` |
| 布尔值 | `TINYINT(1)` | `CHAR(1)` |
| JSON | `JSON` | `TEXT`（MySQL 5.7+） |
| 枚举 | `ENUM` 或 `TINYINT` | `VARCHAR` |

### 常见模式

**联合索引顺序：**
```sql
-- 等值字段在前，范围字段在后
CREATE INDEX idx_order_status_time ON orders(status, created_at);
-- 适用于：WHERE status = 1 AND created_at > '2024-01-01'
```

**覆盖索引：**
```sql
-- 避免回表查询
CREATE INDEX idx_user_cover ON users(email, name, created_at);
-- 查询 SELECT email, name, created_at FROM users WHERE email = ?
-- 不需要回表，直接从索引获取数据
```

**前缀索引：**
```sql
-- 对长字符串使用前缀索引
CREATE INDEX idx_content_prefix ON articles(content(100));
-- 更小的索引，适用于 LIKE 'prefix%' 查询
```

**UPSERT（ON DUPLICATE KEY UPDATE）：**
```sql
INSERT INTO user_settings (user_id, setting_key, setting_value)
VALUES (123, 'theme', 'dark')
ON DUPLICATE KEY UPDATE setting_value = VALUES(setting_value), updated_at = NOW();
```

**游标分页（延迟关联）：**
```sql
-- O(1) vs OFFSET 是 O(n)
SELECT * FROM orders
WHERE id > #{lastId}
ORDER BY id
LIMIT 20;
```

**悲观锁（SELECT FOR UPDATE）：**
```sql
-- 队列处理，使用 SKIP LOCKED 避免等待
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

**批量插入：**
```sql
-- 使用批量插入代替单条插入
INSERT INTO orders (user_id, amount, status) VALUES
    (1, 100.00, 'pending'),
    (2, 200.00, 'pending'),
    (3, 150.00, 'pending');
```

### 反模式检测

```sql
-- 查找未建索引的外键
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    CONSTRAINT_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME IS NOT NULL
  AND TABLE_SCHEMA = 'your_db'
  AND CONSTRAINT_NAME NOT IN (
      SELECT INDEX_NAME
      FROM INFORMATION_SCHEMA.STATISTICS
      WHERE TABLE_SCHEMA = 'your_db'
  );

-- 查找慢查询（需要开启慢查询日志）
-- 或使用 EXPLAIN 分析查询计划
EXPLAIN SELECT * FROM orders WHERE status = 1;

-- 检查表碎片
SELECT
    TABLE_NAME,
    DATA_LENGTH / 1024 / 1024 AS data_mb,
    INDEX_LENGTH / 1024 / 1024 AS index_mb,
    DATA_FREE / 1024 / 1024 AS fragment_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_db'
  AND DATA_FREE > 0;
```

### 配置模板（my.cnf）

```ini
[mysqld]
# 基础配置
port = 3306
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# 连接配置
max_connections = 500
max_connect_errors = 1000

# InnoDB 配置
innodb_buffer_pool_size = 2G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT

# 查询缓存（MySQL 5.7 及以下）
query_cache_size = 0
query_cache_type = 0

# 慢查询日志
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# 二进制日志
log_bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 100M

# 超时配置
wait_timeout = 600
interactive_timeout = 600
innodb_lock_wait_timeout = 50

# 安全配置
skip-name-resolve
local_infile = 0
```

### MyBatis-Plus 最佳实践

```java
// 使用 LambdaQueryWrapper 避免硬编码字段名
LambdaQueryWrapper<MarketEntity> wrapper = Wrappers.lambdaQuery();
wrapper.eq(MarketEntity::getStatus, MarketStatus.ACTIVE)
       .gt(MarketEntity::getVolume, 1000)
       .orderByDesc(MarketEntity::getCreateTime);

// 条件构造器
LambdaQueryWrapper<MarketEntity> wrapper = Wrappers.lambdaQuery();
wrapper.eq(MarketEntity::getDeleted, 0)
       .and(w -> w.eq(MarketEntity::getStatus, MarketStatus.ACTIVE)
                  .or()
                  .eq(MarketEntity::getStatus, MarketStatus.PENDING));
```

## 相关

- Skill：`java-coding-standards` - Java 编码规范
- Skill：`springboot-patterns` - Spring Boot 开发模式
- Skill：`springboot-tdd` - 测试驱动开发

---

*基于国内 MySQL + MyBatis-Plus 技术栈最佳实践*
