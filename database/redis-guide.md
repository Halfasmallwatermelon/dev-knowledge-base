# Redis 使用指南

## 数据结构

```bash
# String
SET key value
GET key

# Hash
HSET user:1 name "John"
HGET user:1 name

# List
LPUSH queue task1
RPUSH queue task2

# Set
SADD tags "python" "redis"

# Sorted Set
ZADD leaderboard 100 "player1"
```

## 常见应用场景

- 缓存
- 分布式锁
- 计数器
- 排行榜