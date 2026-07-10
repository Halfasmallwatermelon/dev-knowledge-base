---
name: database-knowledge
description: 获取、讲解与沉淀数据库知识（SQL、建模、索引、事务、MySQL/PostgreSQL/Redis）。用户学习数据库、写 database/ 笔记、问慢查询/索引/范式/NoSQL 选型时使用。
---

# 数据库知识技能

## 触发后动作

1. Read `database/` 已有笔记
2. 遵循 `.cursor/rules/database-learning.mdc`
3. 落盘时用 `write-learning-note` 模板

## 讲解顺序

业务模型 → Schema/SQL → 执行计划 → 优化 → 一致性方案

## 必含元素（按主题选用）

- **建模**：ER 图、范式与反范式权衡
- **索引**：B+Tree、联合索引最左前缀、覆盖索引
- **事务**：隔离级别与异常现象对照表
- **NoSQL**：Redis 数据结构、缓存穿透/击穿/雪崩

## SQL 案例格式

```sql
-- 问题
SELECT … WHERE …;

-- EXPLAIN 要点
-- type、key、rows、Extra

-- 优化后
CREATE INDEX …;
```

## 权威资源

详见 [resources.md](resources.md)

## 练习粒度

建表 → 写查询 → EXPLAIN → 加索引对比 → 模拟并发事务
