# MongoDB 使用指南

## 什么是 MongoDB？

MongoDB 是一个基于文档的 NoSQL 数据库，使用 JSON-like 的文档存储数据。

## 核心概念

| SQL 概念 | MongoDB 概念 |
|----------|--------------|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |

## 安装与启动

```bash
# Ubuntu
sudo apt-get install mongodb
sudo systemctl start mongodb

# Docker
docker run -d -p 27017:27017 mongo

# 连接
mongo mongodb://localhost:27017/mydb
```

## CRUD 操作

```javascript
// 插入
db.users.insertOne({ name: "John", age: 25 })
db.users.insertMany([{ name: "Jane", age: 30 }, { name: "Bob", age: 35 }])

// 查询
db.users.find({ age: { $gte: 18 } })
db.users.findOne({ name: "John" })

// 更新
db.users.updateOne({ name: "John" }, { $set: { age: 26 } })

// 删除
db.users.deleteOne({ name: "John" })
```

## 索引

```javascript
db.users.createIndex({ email: 1 })
db.users.createIndex({ name: 1, age: -1 })
db.users.createIndex({ email: 1 }, { unique: true })
```

## Python (PyMongo)

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')
db = client.mydb

# 插入
db.users.insert_one({"name": "John", "age": 25})

# 查询
users = db.users.find({"age": {"$gte": 18}})
```
