# SQL 优化大全

## 索引优化

```sql
-- 创建索引
CREATE INDEX idx_users_email ON users(email);

-- 联合索引
CREATE INDEX idx_users_status_created ON users(status, created_at);

-- 覆盖索引
CREATE INDEX idx_users_cover ON users(id, name, email);
```

## 查询优化

```sql
-- 1. 避免 SELECT *
SELECT id, name, email FROM users WHERE id = 1;

-- 2. 分页优化
-- 深分页问题
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;  -- 慢
-- 游标分页
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;  -- 快

-- 3. JOIN 优化
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE u.status = 1;

-- 4. 子查询优化
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);  -- 慢
SELECT DISTINCT u.* FROM users u INNER JOIN orders o ON u.id = o.user_id;  -- 快

-- 5. 批量插入
INSERT INTO users (name, email) VALUES
('user1', 'user1@example.com'),
('user2', 'user2@example.com');
```

## EXPLAIN 分析

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- 重点关注
-- type: ALL(全表扫描) -> index -> range -> ref -> eq_ref -> const
-- key: 使用的索引
-- rows: 预估扫描行数
-- Extra: Using index(覆盖索引), Using filesort(需优化)
```

## 慢查询日志

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 2;

-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query%';
```

## 配置优化

```ini
# my.cnf
[mysqld]
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
max_connections = 200
query_cache_type = 1
query_cache_size = 64M
```