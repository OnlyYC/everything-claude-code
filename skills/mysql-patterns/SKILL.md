---
name: mysql-patterns
description: MySQL 数据库模式：查询优化、Schema 设计、索引策略、安全配置、分表分库、读写分离、性能调优。基于国内 MySQL 最佳实践。
version: 1.1.0
tech_stack: [MySQL 8.0, MyBatis-Plus, ShardingSphere]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills: [springboot-patterns, backend-patterns, springboot-verification]
---

# MySQL 模式

MySQL 最佳实践快速参考，涵盖分表分库、读写分离、性能调优等高级主题。

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
-- 输出：Query OK, 0 rows affected (0.02 sec)
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

## MyBatis-Plus 查询模式

### 基础查询
```java
// 等值查询
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getStatus, 1)
       .eq(User::getDeleted, 0);
List<User> users = userMapper.selectList(wrapper);

// 模糊查询
wrapper.like(User::getName, keyword)
       .or()
       .likeRight(User::getEmail, "@example.com");

// 范围查询
wrapper.ge(User::getCreateTime, startDate)
       .le(User::getCreateTime, endDate);

// IN 查询
wrapper.in(User::getId, userIds)
       .notIn(User::getStatus, deletedStatuses);

// 排序
wrapper.orderByDesc(User::getCreateTime)
       .orderByAsc(User::getName);
```

### 条件组装
```java
// 动态条件 - 只在参数非空时添加条件
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(StringUtils.isNotBlank(name), User::getName, name)
       .eq(status != null, User::getStatus, status)
       .ge(minAge != null, User::getAge, minAge)
       .le(maxAge != null, User::getAge, maxAge)
       .like(StringUtils.isNotBlank(keyword), User::getName, keyword);
```

### 分页查询
```java
// 基础分页
Page<User> page = userMapper.selectPage(
    new Page<>(currentPage, pageSize),
    wrapper
);

// 自定义排序分页
Page<User> page = userMapper.selectPage(
    new Page<>(currentPage, pageSize, false), // false = 不查询总数
    wrapper.orderByDesc(User::getCreateTime)
);
```

### 聚合查询
```java
// 分组统计
QueryWrapper<User> wrapper = Wrappers.query();
wrapper.select("status, count(*) as count")
       .groupBy("status");
List<Map<String, Object>> result = userMapper.selectMaps(wrapper);

// 条件统计
wrapper.select("count(*) as total, sum(amount) as total_amount")
       .ge("create_time", startTime);
```

### 批量操作
```java
// 批量插入
List<User> users = Arrays.asList(user1, user2, user3);
userService.saveBatch(users, 100); // 每批100条

// 批量更新
List<Long> ids = Arrays.asList(1L, 2L, 3L);
userMapper.update(null,
    Wrappers.lambdaUpdate()
        .set(User::getStatus, 1)
        .in(User::getId, ids)
);
```

### 关联查询
```java
// 使用 @TableField(exist = false) 标记非数据库字段
@TableName("user")
public class User {
    @TableId
    private Long id;

    @TableField("dept_id")
    private Long deptId;

    @TableField(exist = false)  // 不查询此字段
    private String deptName;
}

// 手动组装关联数据
List<User> users = userMapper.selectList(wrapper);
Set<Long> deptIds = users.stream()
    .map(User::getDeptId)
    .collect(Collectors.toSet());
Map<Long, Dept> deptMap = deptService.listByIds(deptIds).stream()
    .collect(Collectors.toMap(Dept::getId, Function.identity()));
users.forEach(u -> u.setDeptName(deptMap.get(u.getDeptId()).getName()));
```

## 相关

- Skill：`java-coding-standards` - Java 编码规范
- Skill：`springboot-patterns` - Spring Boot 开发模式
- Skill：`springboot-tdd` - 测试驱动开发

---

## 分表分库

### 分表策略

**水平分表（按范围）：**
```sql
-- 按时间范围分表
CREATE TABLE orders_2024_01 LIKE orders;
CREATE TABLE orders_2024_02 LIKE orders;
CREATE TABLE orders_2024_03 LIKE orders;

-- 按ID范围分表
CREATE TABLE users_0 LIKE users;
CREATE TABLE users_1 LIKE users;
CREATE TABLE users_2 LIKE users;
CREATE TABLE users_3 LIKE users;
```

**水平分表（按哈希）：**
```java
// 应用层哈希分表
public String getTableSuffix(Long userId, int tableCount) {
    return "_" + (userId % tableCount);
}

// 使用方式
String suffix = getTableSuffix(userId, 4);
String tableName = "user_logs" + suffix;
```

**垂直分表：**
```sql
-- 热点字段表（频繁访问）
CREATE TABLE user_core (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    status TINYINT,
    created_at DATETIME,
    INDEX idx_email (email),
    INDEX idx_status (status)
);

-- 冷数据表（不常访问）
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY,
    nickname VARCHAR(50),
    avatar VARCHAR(255),
    bio TEXT,
    birthday DATE,
    updated_at DATETIME
);
```

### ShardingSphere 配置

```yaml
# application-sharding.yml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/db0
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/db1

    rules:
      sharding:
        tables:
          # 订单表分库分表
          t_order:
            actual-data-nodes: ds$->{0..1}.t_order_$->{0..1}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table_mod

        sharding-algorithms:
          db_mod:
            type: MOD
            props:
              sharding-count: 2
          table_mod:
            type: MOD
            props:
              sharding-count: 2

    props:
      sql-show: true
```

**分片算法示例：**
```java
// 自定义分片算法
public class UserIdShardingAlgorithm implements PreciseShardingAlgorithm<Long> {

    @Override
    public String doSharding(Collection<String> availableTargetNames,
                           PreciseShardingValue<Long> shardingValue) {
        Long userId = shardingValue.getValue();
        Long dbIndex = userId % 2;
        for (String targetName : availableTargetNames) {
            if (targetName.endsWith(String.valueOf(dbIndex))) {
                return targetName;
            }
        }
        throw new IllegalArgumentException("无可用数据源");
    }
}

// 时间范围分片算法
public class OrderTimeShardingAlgorithm implements RangeShardingAlgorithm<Date> {

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames,
                                        RangeShardingValue<Date> shardingValue) {
        Collection<String> result = new ArrayList<>();
        Range<Date> range = shardingValue.getValueRange();

        // 根据时间范围选择对应的分表
        Calendar start = Calendar.getInstance();
        start.setTime(range.lowerEndpoint());

        Calendar end = Calendar.getInstance();
        end.setTime(range.upperEndpoint());

        while (start.before(end) || start.equals(end)) {
            String suffix = "_" + start.get(Calendar.YEAR) +
                           "_" + String.format("%02d", start.get(Calendar.MONTH) + 1);
            result.add(shardingValue.getLogicTableName() + suffix);
            start.add(Calendar.MONTH, 1);
        }

        return result;
    }
}
```

---

## 读写分离

### MySQL 主从复制配置

**主库配置（my.cnf）：**
```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
binlog_do_db = your_database
expire_logs_days = 7
max_binlog_size = 100M

# GTID 模式（推荐）
gtid_mode = ON
enforce_gtid_consistency = ON
```

**从库配置（my.cnf）：**
```ini
[mysqld]
server-id = 2
relay-log = mysql-relay-bin
read_only = 1
super_read_only = 1

# GTID 模式
gtid_mode = ON
enforce_gtid_consistency = ON
```

**配置主从复制：**
```sql
-- 在主库创建复制用户
CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;

-- 在从库配置主库信息
CHANGE MASTER TO
  MASTER_HOST='master_host_ip',
  MASTER_USER='replication',
  MASTER_PASSWORD='password',
  MASTER_PORT=3306,
  MASTER_AUTO_POSITION=1;  -- GTID 模式

-- 启动复制
START SLAVE;

-- 查看复制状态
SHOW SLAVE STATUS\G
```

### 应用层读写分离

**ShardingSphere 读写分离：**
```yaml
spring:
  shardingsphere:
    datasource:
      names: master,slave0,slave1
      master:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://master:3306/db
      slave0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave0:3306/db
      slave1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave1:3306/db

    rules:
      readwrite-splitting:
        data-sources:
          readwrite_ds:
            static-strategy:
              write-data-source-name: master
              read-data-source-names: slave0,slave1
            load-balancer-name: round_robin

        load-balancers:
          round_robin:
            type: ROUND_ROBIN
```

**MyBatis-Plus 读写分离：**
```java
// 自定义读写分离数据源
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource dynamicDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("slave", slaveDataSource());

        DynamicRoutingDataSource dataSource = new DynamicRoutingDataSource();
        dataSource.setTargetDataSources(targetDataSources);
        dataSource.setDefaultTargetDataSource(masterDataSource());

        return dataSource;
    }
}

// 使用注解切换数据源
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnly {
}

// AOP 切面
@Aspect
@Component
public class DataSourceAspect {

    @Before("@annotation(readOnly)")
    public void setReadDataSource(ReadOnly readOnly) {
        DynamicRoutingDataSource.setDataSource("slave");
    }

    @After("@annotation(readOnly)")
    public void clearDataSource() {
        DynamicRoutingDataSource.clearDataSource();
    }
}
```

**强制读主库：**
```java
// 对于需要强一致性的场景，强制读主库
@Service
public class OrderService {

    @Transactional
    public Order createOrder(OrderDTO dto) {
        Order order = orderMapper.insert(dto);

        // 创建订单后立即查询，需要读主库
        Order result = orderMapper.selectById(order.getId());

        return result;
    }
}
```

---

## 性能调优

### SQL 优化技巧

**避免 SELECT \*：**
```sql
-- ❌ 避免
SELECT * FROM orders WHERE user_id = ?;

-- ✅ 推荐
SELECT id, amount, status FROM orders WHERE user_id = ?;
```

**使用覆盖索引：**
```sql
-- 创建覆盖索引
CREATE INDEX idx_user_status_amount ON orders(user_id, status, amount);

-- 查询只使用索引，不需要回表
SELECT status, amount FROM orders WHERE user_id = ?;
```

**优化子查询：**
```sql
-- ❌ 子查询
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE amount > 1000);

-- ✅ JOIN
SELECT u.* FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.amount > 1000;
```

**批量操作：**
```java
// ✅ 使用批量插入
@Service
public class BatchInsertService {

    public void batchInsert(List<Order> orders) {
        // 分批插入，每批 1000 条
        int batchSize = 1000;
        for (int i = 0; i < orders.size(); i += batchSize) {
            int end = Math.min(i + batchSize, orders.size());
            List<Order> batch = orders.subList(i, end);
            orderMapper.insertBatch(batch);
        }
    }
}
```

### 索引优化

**联合索引顺序：**
```sql
-- 等值条件在前，范围条件在后
CREATE INDEX idx_a_b_c ON table(a, b, c);

-- 适用查询
WHERE a = 1 AND b = 2 AND c > 3; -- ✅ 完全使用
WHERE a = 1 AND b > 2 AND c = 3;  -- ❌ c 索引失效
```

**索引下推（ICP）：**
```sql
-- MySQL 5.6+ 自动使用索引下推
CREATE INDEX idx_name_age ON users(name, age);

-- 查询在存储引擎层过滤
SELECT * FROM users WHERE name LIKE 'Zhang%' AND age > 20;
```

### 连接池优化

**HikariCP 配置：**
```yaml
spring:
  datasource:
    hikari:
      # 连接池大小 = CPU 核心数 * 2 + 磁盘数
      maximum-pool-size: 20
      minimum-idle: 10

      # 连接超时
      connection-timeout: 30000
      validation-timeout: 5000

      # 空闲连接存活时间
      max-lifetime: 1200000
      idle-timeout: 600000

      # 连接测试
      connection-test-query: SELECT 1
```

**连接池监控：**
```java
@Component
public class HikariMonitor {

    @Autowired
    private DataSource dataSource;

    @Scheduled(fixedRate = 60000)
    public void monitor() {
        if (dataSource instanceof HikariDataSource) {
            HikariPoolMXBean pool = ((HikariDataSource) dataSource).getHikariPoolMXBean();

            log.info("活跃连接: {}, 空闲连接: {}, 总连接: {}, 等待线程: {}",
                pool.getActiveConnections(),
                pool.getIdleConnections(),
                pool.getTotalConnections(),
                pool.getThreadsAwaitingConnection()
            );
        }
    }
}
```

### InnoDB 优化

```ini
# 缓冲池大小（物理内存的 50-70%）
innodb_buffer_pool_size = 4G

# 多个缓冲池实例（CPU 核心数）
innodb_buffer_pool_instances = 8

# 刷新策略
innodb_flush_log_at_trx_commit = 2  # 性能模式
innodb_flush_method = O_DIRECT

# 日志文件大小
innodb_log_file_size = 512M
innodb_log_buffer_size = 16M

# IO 容量
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# 脏页刷新
innodb_max_dirty_pages_pct = 75
innodb_max_dirty_pages_pct_lwm = 50
```

### 慢查询分析

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- 分析慢查询
SELECT * FROM mysql.slow_log
WHERE query_time > 1
ORDER BY query_time DESC
LIMIT 10;

-- 使用 EXPLAIN 分析
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- 使用 EXPLAIN ANALYZE（MySQL 8.0+）
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

**EXPLAIN 关键指标：**
- `type`：访问类型（system > const > eq_ref > ref > range > index > ALL）
- `key`：实际使用的索引
- `rows`：扫描的行数
- `Extra`：额外信息（Using index = 覆盖索引，Using filesort = 文件排序）

---

## 高可用架构

### 主主复制

```ini
# 主库 1
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-do-db = your_database
auto-increment-increment = 2
auto-increment-offset = 1

# 主库 2
[mysqld]
server-id = 2
log-bin = mysql-bin
binlog-do-db = your_database
auto-increment-increment = 2
auto-increment-offset = 2
```

### 故障转移

```java
// 使用 VIP 或 DNS 实现自动故障转移
@Configuration
public class FailoverConfig {

    @Bean
    public DataSource failoverDataSource() {
        String masterUrl = System.getProperty("db.master.url");
        String slaveUrl = System.getProperty("db.slave.url");

        // 检查主库可用性
        if (isDatabaseAvailable(masterUrl)) {
            return createDataSource(masterUrl);
        } else {
            log.warn("主库不可用，切换到从库");
            return createDataSource(slaveUrl);
        }
    }

    private boolean isDatabaseAvailable(String url) {
        try (Connection conn = DriverManager.getConnection(url)) {
            return conn.isValid(5);
        } catch (SQLException e) {
            return false;
        }
    }
}
```

---

**记住**：数据库是系统的核心，分表分库、读写分离需要在设计阶段就考虑好。性能优化是持续的过程，需要定期监控和调整。

*基于国内 MySQL 8.0 + MyBatis-Plus + ShardingSphere 技术栈最佳实践*
