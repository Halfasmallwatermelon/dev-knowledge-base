# MySQL 性能优化

## 索引优化

```sql
-- 创建索引
ALTER TABLE users ADD INDEX idx_email (email);

-- 联合索引
ALTER TABLE users ADD INDEX idx_name_age (name, age);
```

## 查询优化

```sql
-- 避免 SELECT *
SELECT id, name, email FROM users WHERE id = 1;

-- 分页优化
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

## EXPLAIN 分析

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```