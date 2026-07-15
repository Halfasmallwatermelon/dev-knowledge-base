# MongoDB 技术文档

## 一、概述 (Overview)
MongoDB 是一款面向文档的分布式 NoSQL 数据库，采用灵活的模式设计，支持高并发读写、水平扩展与内置高可用机制。其核心优势包括：Schema-Free 数据模型、原生 JSON 查询语言、强大的聚合能力、自动故障转移与分片路由。

```javascript
// 验证连接并获取版本信息
db.adminCommand({ buildInfo: 1 })
```

## 二、架构设计 (Architecture)
MongoDB 采用分层架构：客户端通过驱动程序或 `mongosh` 连接至 `mongod`（单节点）或 `mongos`（分片路由）。数据经由存储引擎（默认 WiredTiger）持久化至磁盘，内存缓存（WiredTiger Cache）加速热点数据访问。网络层支持 TCP/IP 与 TLS 加密。

```yaml
# mongod.conf 基础架构配置示例
storage:
  dbPath: /var/lib/mongodb
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.1.100
```

## 三、核心概念 (Core Concepts)
- **Document（文档）**：基本数据单元，以 BSON 格式存储，形如 JSON。必须包含唯一 `_id` 字段（默认 ObjectId）。
- **Collection（集合）**：文档的物理容器，无强制 Schema，允许异构结构。
- **BSON（Binary JSON）**：扩展 JSON 的二进制编码格式，支持 Date、Binary、Decimal128、ObjectId 等类型，提升解析效率与空间利用率。

```javascript
// 文档插入与 BSON 类型体现
db.users.insertOne({
  _id: ObjectId("507f1f77bcf86cd799439011"),
  name: "张三",
  age: NumberInt(28),
  createdAt: new Date(),
  tags: ["admin", "developer"],
  metadata: BinData(0, "SGVsbG8gV29ybGQ=")
})
```

## 四、安装部署 (Installation)
### 1. Standalone（单节点）
适用于开发测试或低负载场景。直接启动单一 `mongod` 进程。
```bash
mongod --dbpath /data/db --port 27017 --logpath /var/log/mongod.log
```

### 2. Replica Set（副本集）
至少 3 个节点（含 1 主 2 从或 1 仲裁），提供自动故障切换。
```javascript
// 初始化副本集
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "node1:27017" },
    { _id: 1, host: "node2:27017" },
    { _id: 2, host: "node3:27017" }
  ]
})
```

### 3. Sharded Cluster（分片集群）
由 `mongos` 路由、`configsvr` 配置节点、`shards` 数据节点组成。
```bash
# 启动配置服务器（至少3个）
mongod --configsvr --replSet cfgRepl --port 27019 --dbpath /data/configdb

# 启动分片节点
mongod --shardsvr --replSet shard0Repl --port 27018 --dbpath /data/shard0

# 启动路由服务
mongos --configdb cfgRepl/node1:27019,node2:27019,node3:27019 --port 27017
```

## 五、CRUD 操作与高级示例 (CRUD Operations)
支持原子级更新、批量写入、条件投影与聚合表达式更新。

```javascript
// 基础 CRUD
db.orders.insertOne({ item: "keyboard", qty: 15, status: "A" })
db.orders.find({ status: "A" }).limit(5)
db.orders.updateOne({ item: "keyboard" }, { $set: { qty: 20 } })
db.orders.deleteMany({ status: "D" })

// 高级：Upsert + 聚合表达式更新
db.products.updateMany(
  { category: "electronics" },
  [
    { $set: { price: { $multiply: ["$price", 1.1] }, updated_at: new Date() } }
  ],
  { upsert: true }
)

// 高级：Bulk Write 批量操作
const bulk = db.sessions.initializeUnorderedBulkOp()
bulk.insert({ user: "u1", action: "login", ts: new Date() })
bulk.update({ user: "u2" }, { $inc: { score: 5 } })
bulk.execute()
```

## 六、索引机制 (Indexing)
支持单字段、复合、多键、文本、地理空间、哈希索引。覆盖查询（Covered Query）可完全利用索引返回结果，避免回表。

```javascript
// 创建复合索引与文本索引
db.logs.createIndex({ timestamp: -1, level: 1 })
db.articles.createIndex({ title: "text", content: "text" })

// 覆盖查询示例（仅返回索引中存在的字段）
db.inventory.createIndex({ item: 1, status: 1, qty: 1 })
db.inventory.find(
  { status: "A" },
  { item: 1, qty: 1, _id: 0 }
).explain("executionStats").executionStats.executionStages.stage // 应为 "FETCH" 无 "COLLSCAN"

// 哈希索引（适合分片键）
db.sharded_collection.createIndex({ shardKey: "hashed" })
```

## 七、聚合管道 (Aggregation Pipeline)
基于阶段的流式处理模型，支持 `$match`、`$group`、`$lookup`、`$unwind` 等 50+ 阶段。

```javascript
db.sales.aggregate([
  { $match: { date: { $gte: ISODate("2024-01-01") } } },
  { $unwind: "$items" },
  { $group: {
      _id: { region: "$region", product: "$items.product" },
      totalQty: { $sum: "$items.qty" },
      avgPrice: { $avg: "$items.price" }
    }
  },
  { $sort: { totalQty: -1 } },
  { $limit: 10 },
  { $project: {
      _id: 0,
      region: "$_id.region",
      product: "$_id.product",
      totalQty: 1,
      avgPrice: { $round: ["$avgPrice", 2] }
    }
  }
])
```

## 八、副本集与分片集群 (Replica Set & Sharding)
- **副本集**：Primary 处理写，Secondaries 异步复制。读可配置 `secondaryPreferred`。
- **分片**：通过 Shard Key 将数据切分为 Chunk，Balancer 自动迁移。范围分片与哈希分片是主流策略。

```javascript
// 启用数据库分片与集合分片
sh.enableSharding("ecommerce")
sh.shardCollection("ecommerce.orders", { orderId: "hashed" })

// 查看分片状态与平衡器
sh.status()
db.settings.find({ _id: "balancer" })

// 副本集读写偏好设置
db.orders.find({}).readPref("secondaryPreferred")
```

## 九、多文档事务 (Transactions)
MongoDB 4.0+ 支持跨文档、跨集合 ACID 事务，需在副本集或分片集群上运行。默认使用快照读，隔离级别为 `snapshot`。

```javascript
const session = db.getMongo().startSession()
session.startTransaction({ readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } })

try {
  const accts = session.getDatabase("bank").accounts
  accts.updateOne({ _id: "A" }, { $inc: { balance: -100 } })
  accts.updateOne({ _id: "B" }, { $inc: { balance: 100 } })
  session.commitTransaction()
} catch (err) {
  session.abortTransaction()
  throw err
} finally {
  session.endSession()
}
```

## 十、性能优化指南 (Performance Optimization)
关键策略：合理索引、控制文档大小、使用 `explain` 分析执行计划、调整 WiredTiger 缓存、限制 `$where` 与正则全扫。

```javascript
// 开启慢查询日志（阈值 100ms）
db.setProfilingLevel(1, { slowms: 100 })

// 查看当前操作与锁等待
db.currentOp({ "secs_running": { $gt: 5 } })

// 集合统计与索引使用情况
db.collection.stats()
db.collection.aggregate([ { $indexStats: {} } ])

// 优化 WiredTiger 缓存比例（物理内存 50%~70%）
// 在 mongod.conf 中配置：
// wiredTiger.engineConfig.cacheSizeGB: 4
```

## 十一、安全配置 (Security Configuration)
启用身份认证、基于角色的访问控制（RBAC）、传输层加密（TLS/SSL）与字段级加密（企业版）。

```yaml
# 启用认证与 TLS
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile
net:
  ssl:
    mode: requireSSL
    PEMKeyFile: /etc/mongodb/server.pem
    CAFile: /etc/mongodb/ca.pem
```

```javascript
// 创建管理员与只读角色用户
db.createUser({
  user: "admin",
  pwd: "strongPassword123!",
  roles: [{ role: "root", db: "admin" }]
})
db.createUser({
  user: "reader",
  pwd: "readerPass!",
  roles: [{ role: "read", db: "appdb" }]
})
```

## 十二、客户端驱动示例 (Python/Node.js Driver Examples)
### Python (PyMongo)
```python
from pymongo import MongoClient
from bson.objectid import ObjectId

client = MongoClient("mongodb://admin:***@host:27017/?authSource=admin&replicaSet=rs0")
db = client["ecommerce"]
collection = db["orders"]

# 插入
res = collection.insert_one({"item": "mouse", "qty": 5})
print(res.inserted_id)

# 查询与聚合
pipeline = [{"$group": {"_id": "$status", "count": {"$sum": 1}}}]
for doc in collection.aggregate(pipeline):
    print(doc)
```

### Node.js (MongoDB Driver)
```javascript
const { MongoClient } = require('mongodb');
const uri = 'mongodb://admin:***@host:27017/?authSource=admin&replicaSet=rs0';
const client = new MongoClient(uri);

async function run() {
  await client.connect();
  const db = client.db('ecommerce');
  const coll = db.collection('orders');

  await coll.insertOne({ item: 'keyboard', qty: 10 });
  const results = await coll.find({ qty: { $gt: 5 } }).toArray();
  console.log(results);
  await client.close();
}
run().catch(console.error);
```

## 十三、常见故障排查 (Troubleshooting)
| 现象 | 诊断命令 | 解决方案 |
|------|----------|----------|
| 连接超时/拒绝 | `db.adminCommand({ serverStatus: 1 }).network` | 检查防火墙、bindIp、端口监听 |
| 复制延迟高 | `rs.printSecondaryReplicationInfo()` | 优化索引、增加 Secondary 资源、检查网络带宽 |
| 磁盘空间满 | `db.serverStatus().storageEngine` | 清理 oplog、启用 TTL、扩容或归档冷数据 |
| 锁等待严重 | `db.currentOp({ waitingForLock: true })` | 减少大事务、拆分批量写入、检查未命中索引的查询 |

```javascript
// 综合诊断脚本
db.serverStatus().opcounters
db.serverStatus().connections
db.serverStatus().wiredTiger.cache["modified bytes evicted"]
db.printReplicationInfo() // 查看 oplog 窗口
```

---
*本文档基于 MongoDB 5.0/6.0+ 语法编写，实际生产环境请结合版本特性与运维规范进行调整。*
