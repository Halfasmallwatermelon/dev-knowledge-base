# MongoDB 技术文档

本文档面向中高级开发者与运维工程师，系统介绍 MongoDB 的核心机制、部署架构、CRUD 操作、性能优化、数据建模及生产实践。所有代码示例均基于现代 `mongosh`（MongoDB Shell）语法，可直接在交互式环境中运行。

---

## 1. 概述

### 1.1 什么是 MongoDB？
MongoDB 是一款开源的、面向文档的 NoSQL 数据库，采用 BSON（Binary JSON）格式存储数据。它支持动态 Schema、水平扩展、丰富的查询语言、聚合框架以及多文档事务，广泛应用于现代云原生与微服务架构中。

### 1.2 为什么选择 MongoDB？
| 优势 | 说明 |
|------|------|
| 灵活 Schema | 无需预定义表结构，字段可动态增减 |
| 高性能读写 | 内存映射文件 + WiredTiger 存储引擎，支持并发写入 |
| 水平扩展 | 原生分片架构，自动数据均衡 |
| 开发者友好 | 类 JSON 数据模型，与 JavaScript/Python/Java 等生态无缝对接 |
| 功能丰富 | 全文检索、地理空间索引、时间序列集合、变更流等 |

### 1.3 适用场景
- 内容管理系统（CMS）与博客平台
- 用户画像与配置中心
- 实时分析、日志采集与 IoT 数据
- 商品目录与电商库存
- 移动应用后端（快速迭代 Schema）

```javascript
// 连接测试与版本查看
const serverStatus = db.adminCommand({ serverStatus: 1 });
print(`MongoDB 版本: ${serverStatus.version}`);
print(`存储引擎: ${serverStatus.storageEngine.name}`);
print(`当前数据库: ${db.getName()}`);
```

---

## 2. 核心概念

### 2.1 Document（文档）
文档是 MongoDB 的基本存储单元，由键值对组成，类似 JSON。每个文档必须包含唯一的 `_id` 字段。

### 2.2 Collection（集合）
集合是一组文档的容器，类似关系型数据库中的“表”。集合不强制 Schema，同一集合内文档字段可以不同。

### 2.3 BSON
BSON 是 Binary-encoded Serialization Format，支持日期、二进制数据、正则、Decimal128 等 JSON 原生不支持的类型。

### 2.4 ObjectId
`ObjectId` 是默认的 `_id` 生成器，12 字节：
- 前 4 字节：时间戳（秒级）
- 中间 5 字节：机器标识 + 进程 ID
- 后 3 字节：自增计数器

### 2.5 数据库与集合操作

```javascript
// 切换/创建数据库（隐式创建）
use myapp

// 列出所有集合
show collections

// 创建集合（可指定约束）
db.createCollection("orders", {
  validator: { $jsonSchema: { required: ["orderId", "amount"] } },
  timeseries: { timeField: "createdAt", metaField: "deviceId" } // MongoDB 5.0+ 时序集合
})

// 删除集合
db.orders.drop()

// 查看集合统计信息
db.orders.stats()

// 重命名集合
db.old_orders.renameCollection("orders_v2")
```

---

## 3. 安装与部署

### 3.1 单节点部署
适用于开发测试或轻量生产环境。

```bash
# Linux 启动命令（默认端口 27017，数据目录 /data/db）
mongod --dbpath /data/db --port 27017 --bind_ip 127.0.0.1 --journal --wiredTigerCacheSizeGB 1

# Windows 可使用服务注册
mongod --install --dbpath C:\mongodb\data --serviceName "MongoDB"
```

### 3.2 副本集（Replica Set）
提供高可用与自动故障转移。推荐至少 3 个节点（含仲裁节点）。

```javascript
// 节点 A/B/C 分别以 --replSet rs0 启动后，在任一节点执行：
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "node-a:27017", priority: 2 },
    { _id: 1, host: "node-b:27017", priority: 1 },
    { _id: 2, host: "node-c:27017", priority: 0, arbiterOnly: true }
  ]
})

// 查看状态
rs.status()
rs.conf()
rs.printReplicationInfo()
```

### 3.3 分片集群（Sharded Cluster）
适用于海量数据与高吞吐场景。架构包含：`mongos` 路由 → `config server`（3节点）→ `shard`（副本集）。

```javascript
// 启用分片
sh.enableSharding("ecommerce")

// 选择分片键并创建索引
sh.shardCollection("ecommerce.products", { categoryId: 1 })

// 手动添加分片
sh.addShard("shard1/10.0.0.1:27017,10.0.0.2:27017")
sh.addShard("shard2/10.0.0.3:27017,10.0.0.4:27017")

// 查看分片状态
sh.status()

// 对已有集合启用哈希分片（均匀分布）
sh.shardCollection("ecommerce.logs", { _id: "hashed" })
```

> 💡 **提示**：生产环境建议分片键具有高基数且查询常用，避免使用递增字段作为分片键导致热点。

---

## 4. CRUD 操作详解

### 4.1 插入（Insert）

```javascript
// 单条插入
const insertResult = db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  age: 28,
  tags: ["admin", "developer"],
  profile: { city: "Shanghai", joinDate: new Date("2023-01-15") }
});
print(`插入 ID: ${insertResult.insertedId}`);

// 批量插入
db.users.insertMany([
  { name: "Bob", age: 32, tags: ["user"] },
  { name: "Carol", age: 25, tags: ["user", "beta"] }
]);

// Upsert（不存在则插入，存在则更新）
db.users.updateOne(
  { email: "new@example.com" },
  { $setOnInsert: { name: "Dave", age: 30 } },
  { upsert: true }
);
```

### 4.2 查询（Find）

```javascript
// 基础查询与投影
db.users.find({ age: { $gte: 25 } }, { name: 1, age: 1, _id: 0 }).sort({ age: -1 }).limit(10);

// 逻辑运算符
db.users.find({
  $or: [{ age: { $lt: 20 } }, { tags: "beta" }],
  name: { $regex: "^A" }
});

// 数组查询
db.users.find({ tags: { $all: ["user", "admin"] } });
db.users.find({ "profile.city": "Shanghai" });

// 分页查询（游标方式，适合大数据量）
let cursor = db.users.find().skip(100).limit(50);
while (cursor.hasNext()) printjson(cursor.next());

// 统计与去重
db.users.countDocuments({ age: { $gte: 25 } });
db.users.distinct("tags");
```

### 4.3 更新（Update）

```javascript
// 安全更新（仅匹配第一条）
db.users.updateOne(
  { name: "Alice" },
  {
    $set: { age: 29 },
    $inc: { loginCount: 1 },
    $push: { tags: "senior" },
    $currentDate: { lastLogin: true }
  }
);

// 批量更新
db.users.updateMany(
  { age: { $lt: 20 } },
  { $set: { category: "junior" } }
);

// 移除字段
db.users.updateOne({ name: "Bob" }, { $unset: { tags: "" } });

// 数组位置操作符
db.users.updateOne(
  { name: "Alice" },
  { $set: { "tags.$[elem]": "moderator" } }
).options = { arrayFilters: [{ "elem": { $eq: "admin" } }] };
// 注意：arrayFilters 需在 updateOne 第二个参数对象外单独设置（见下方完整写法）

// 完整带过滤数组更新
db.users.updateOne(
  { name: "Alice" },
  { $set: { "tags.$[elem]": "moderator" } },
  { arrayFilters: [{ "elem": { $eq: "admin" } }] }
);
```

### 4.4 删除（Delete）

```javascript
db.users.deleteOne({ name: "Bob" });
db.users.deleteMany({ age: { $lt: 18 } });

// 软删除标记（推荐生产使用）
db.users.updateOne({ _id: userId }, { $set: { deletedAt: new Date() } });
db.users.find({ deletedAt: { $exists: true } });
```

---

## 5. 索引与性能优化

### 5.1 索引类型

| 类型 | 用途 |
|------|------|
| 单字段 B-Tree | 最常见，等值/范围查询 |
| 复合索引 | 多字段排序与查询，遵循最左前缀原则 |
| 稀疏索引 | 跳过不含索引字段的文档 |
| 文本索引 | 全文检索 |
| 地理空间索引 | `$near`, `$geoWithin` |
| TTL 索引 | 自动过期删除 |
| 哈希索引 | 分片键均匀分布 |
| 部分索引 | `partialFilterExpression` 过滤条件 |

### 5.2 创建与管理

```javascript
// 单字段索引
db.users.createIndex({ email: 1 }, { unique: true });

// 复合索引
db.orders.createIndex({ customerId: 1, createdAt: -1 });

// 稀疏唯一索引
db.profiles.createIndex({ phone: 1 }, { sparse: true, unique: true });

// TTL 索引（30 分钟后自动删除）
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 30 });

// 部分索引
db.logs.createIndex({ level: 1 }, { partialFilterExpression: { level: "ERROR" } });

// 查看索引
db.users.getIndexes();
db.users.dropIndex("email_1");
```

### 5.3 Explain 分析与慢查询

```javascript
// 获取执行计划
db.users.find({ age: { $gte: 25 }, name: "Alice" }).explain("executionStats");

// 关键指标解读：
// totalDocsExamined: 扫描文档数（应接近返回数）
// nReturned: 返回文档数
// executionTimeMillis: 执行耗时
// winStage: IXSCAN（走索引）优于 COLLSCAN（全表扫描）

// 开启查询分析器（0=关, 1=慢查询>100ms, 2=全部）
db.setProfilingLevel(1, { slowOpThresholdMs: 50 });
db.system.profile.find().sort({ ts: -1 }).limit(5);

// 强制使用索引（调试用，生产慎用）
db.orders.find({ customerId: "C123" }).hint({ customerId: 1 });
```

### 5.4 优化建议
- 查询条件尽量覆盖索引前缀
- 使用投影减少网络传输
- 避免在索引字段上使用函数（如 `$where`、`$expr` 可能回退全扫）
- 大集合删除使用分批 `deleteMany` 配合 `limit`
- 定期运行 `collMod` 整理碎片：`db.runCommand({ compact: "users" })`

---

## 6. 聚合管道

聚合框架用于多阶段数据处理，支持 `$match`、`$group`、`$lookup`、`$unwind`、`$project`、`$sort`、`$limit` 等。

```javascript
db.orders.aggregate([
  // 阶段1：过滤
  { $match: { status: "paid", createdAt: { $gte: ISODate("2024-01-01") } } },
  
  // 阶段2：关联用户
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "userInfo"
    }
  },
  { $unwind: "$userInfo" },
  
  // 阶段3：展开订单明细
  { $unwind: "$items" },
  
  // 阶段4：计算单品金额
  { $addFields: { itemTotal: { $multiply: ["$items.price", "$items.qty"] } } },
  
  // 阶段5：按用户分组汇总
  {
    $group: {
      _id: "$userId",
      userName: { $first: "$userInfo.name" },
      orderCount: { $sum: 1 },
      totalAmount: { $sum: "$itemTotal" },
      avgPrice: { $avg: "$items.price" }
    }
  },
  
  // 阶段6：过滤与排序
  { $match: { totalAmount: { $gte: 500 } } },
  { $sort: { totalAmount: -1 } },
  { $limit: 20 },
  
  // 阶段7：重命名字段
  { $project: { _id: 0, userId: "$_id", userName: 1, orderCount: 1, totalAmount: 1, avgPrice: { $round: ["$avgPrice", 2] } } }
]);
```

> 💡 **注意**：管道默认最多处理 100MB 中间结果，可通过 `.allowDiskUse(true)` 启用磁盘溢出。

---

## 7. 数据建模

### 7.1 内嵌 vs 引用

| 模式 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 内嵌（Embedding） | 单次读取、无 JOIN、事务简单 | 文档大小限制 16MB、更新分散 | 一对一、一对少、读多写少 |
| 引用（Referencing） | 数据独立、易于扩展 | 需多次查询/$lookup、一致性复杂 | 一对多、多对多、数据量大 |

### 7.2 范式化与反范式化
- **反范式化**：冗余字段（如用户名、价格快照）提升读取性能，但需保证同步更新。
- **范式化**：保持数据一致性，适合频繁更新的核心实体。
- **混合策略**：主数据范式化，展示数据反范式化缓存。

### 7.3 数组操作

```javascript
// 向数组追加
db.cart.updateOne({ userId: "U001" }, { $push: { items: { sku: "S100", qty: 2 } } });

// 批量追加 + 限制长度
db.cart.updateOne({ userId: "U001" }, {
  $push: { logs: { $each: [{ t: new Date(), msg: "login" }], $slice: -100 } }
});

// 条件删除数组元素
db.cart.updateOne({ userId: "U001" }, { $pull: { items: { sku: "S100" } } });

// 定位更新数组内第一个匹配项
db.cart.updateOne(
  { userId: "U001", "items.sku": "S100" },
  { $inc: { "items.$.qty": 1 } }
);

// 更新所有数组元素
db.cart.updateOne(
  { userId: "U001" },
  { $inc: { "items.$[].qty": 1 } }
);
```

### 7.4 文档大小限制
单个 BSON 文档最大 **16MB**。若业务需要更大对象，应拆分为子集合或存储至文件系统/OSS。

---

## 8. 事务与一致性

MongoDB 从 4.0 起支持副本集多文档事务，4.2+ 支持分片集群事务。底层基于 MVCC 与两阶段提交。

### 8.1 多文档事务

```javascript
// 创建会话
const session = db.getMongo().startSession();

try {
  session.withTransaction(async () => {
    const buyer = session.getDatabase("bank").accounts;
    const seller = session.getDatabase("bank").accounts;

    buyer.updateOne({ id: "A" }, { $inc: { balance: -100 } });
    seller.updateOne({ id: "B" }, { $inc: { balance: 100 } });
    
    // 可在此调用其他集合操作
  }, {
    readPreference: "primary",
    readConcern: { level: "local" },
    writeConcern: { w: "majority" }
  });
  print("事务提交成功");
} catch (e) {
  print(`事务失败: ${e.message}`);
} finally {
  session.endSession();
}
```

### 8.2 读写关注（Read/Write Concern）

| Write Concern | 含义 |
|---------------|------|
| `{ w: 1 }` | 单节点确认写入 |
| `{ w: "majority" }` | 多数副本确认，防主从切换丢数据 |
| `{ w: 0 }` | 异步写入，最快但不可靠 |

| Read Concern | 含义 |
|--------------|------|
| `local` | 默认，读取最新已持久化数据 |
| `majority` | 等待多数副本确认，强一致 |
| `available` | 允许读取可能丢失的数据 |

### 8.3 读偏好（Read Preference）

```javascript
// 设置会话读偏好
session.withTransaction(() => {
  return db.orders.find({}).readPref("secondaryPreferred").toArray();
});

// 常见模式：primary（写/强读）、secondaryPreferred（读负载分流）、nearest（低延迟）
```

> ⚠️ 事务内必须使用 `primary` 读偏好，且不支持跨数据库集合的只读查询参与事务提交决策。

---

## 9. 安全与认证

### 9.1 用户管理与角色

```javascript
// 切换到 admin 库创建管理员
use admin
db.createUser({
  user: "root",
  pwd: "StrongP@ssw0rd",
  roles: [{ role: "root", db: "admin" }]
});

// 创建业务用户
use myapp
db.createUser({
  user: "appReader",
  pwd: "readerPass",
  roles: [
    { role: "read", db: "myapp" },
    { role: "readWrite", db: "myapp", collection: "public_logs" }
  ]
});

// 修改密码与删除用户
db.changeUserPassword("appReader", "newPass");
db.dropUser("appReader");

// 查看用户
db.getUsers();
```

### 9.2 内置角色速查
- `read` / `readWrite`：库级只读/读写
- `dbAdmin`：集合管理、索引创建
- `clusterMonitor`：监控权限
- `backup` / `restore`：备份恢复专用
- `readAnyDatabase` / `writeAnyDatabase`：跨库权限（谨慎授予）

### 9.3 TLS/SSL 加密配置
在 `mongod.conf` 中配置：

```yaml
net:
  port: 27017
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/ssl/server.pem
    CAFile: /etc/mongodb/ssl/ca.pem
    allowConnectionsWithoutTLS: false
```

客户端连接：
```javascript
// mongosh 启动时指定
// mongosh "mongodb://user:pass@host:27017/?tls=true&tlsCAFile=ca.pem"

// 或在代码中
const client = new MongoClient(uri, { tls: true, tlsCAFile: "/path/to/ca.pem" });
```

### 9.4 审计日志
```yaml
security:
  authorization: enabled
auditLog:
  destination: file
  path: /var/log/mongodb/audit.log
  format: BSON
  filter: '{ atype: { $in: ["authSuccess", "authFailure", "command"] } }'
```

> 🔒 **生产强制**：始终启用 `authorization: enabled`，禁止绑定 `0.0.0.0`，使用 VPC/防火墙隔离。

---

## 10. 最佳实践

### 10.1 生产环境配置建议
- 启用 Journaling：`storage.journal.enabled: true`
- 合理设置缓存：`storage.wiredTiger.engineConfig.cacheSizeGB` ≈ 系统内存的 50%~70%
- 关闭不必要的网络暴露：`net.bindIp: 127.0.0.1,10.0.0.0/24`
- 副本集至少 3 节点，分片集群 Config Server 固定 3 节点
- 定期升级 Patch 版本，避免大版本跳跃

### 10.2 备份与恢复

```bash
# 逻辑备份（推荐）
mongodump --uri="mongodb://user:pass@host:27017/myapp" --gzip --out=/backup/daily/

# 增量备份（配合 oplog）
mongodump --oplog --snapshot --uri="..." --out=/backup/

# 恢复
mongorestore --uri="mongodb://user:pass@host:27017/" --gzip /backup/daily/myapp

# 物理备份（WiredTiger 文件级，速度快）
mongodump 不支持物理备份，需使用：
- Percona MongoDB Backup
- MongoDB Cloud Manager / Ops Manager
- 或停机复制 /data/db 目录
```

### 10.3 监控方案

```bash
# 实时统计
mongostat --host localhost --interval 1
mongotop --host localhost --interval 1

# 集合级锁与操作统计
db.serverStatus().opcounters
db.serverStatus().metrics.commands

# 推荐监控栈
# - Prometheus + mongodb_exporter + Grafana
# - MongoDB Atlas / Cloud Manager（商业）
# - ELK/Loki 采集 audit log 与 slow query
```

Grafana 核心面板指标：
- `ops`（QPS）、`connection.clients`、`wiredTiger.cache`、`oplatencies.total`、`replset.getLastError.wtime`

---

## 11. 常见问题与排错

### 11.1 连接失败
**错误**：`ECONNREFUSED 127.0.0.1:27017`
```bash
# 检查进程
ps aux | grep mongod
# 检查端口
ss -tlnp | grep 27017
# 确认 bindIp 与防火墙
sudo ufw allow 27017/tcp
```

### 11.2 WiredTiger 锁等待/超时
**错误**：`WiredTiger (8) call failed: File exists` 或长时间 `lock wait`
```javascript
// 查看当前长事务
db.currentOp({ "secs_running": { $gt: 60 } })

// 终止异常操作
db.killOp(<opid>)

// 调整事务超时
db.adminCommand({ setParameter: 1, transactionLifetimeLimitSeconds: 60 })
```

### 11.3 文档超过 16MB
**原因**：插入/更新后 BSON 大小超限。
**解决**：拆分文档、使用 GridFS、或检查 `$push` 数组是否无限增长。

```javascript
// 查看最大文档大小限制
db.adminCommand({ getParameter: 1, maxDocumentSize: 1 })
```

### 11.4 Oplog 过小导致副本集同步失败
```javascript
// 查看 oplog 窗口
rs.printReplicationInfo()
// 建议：主库写入量 × 故障恢复时间 × 1.5

// 扩容 oplog（需在 PRIMARY 执行）
use local
db.oplog.rs.stats().maxSize
db.adminCommand({ resizeOplog: 1, size: 10240 }) // MB
```

### 11.5 索引构建阻塞写入
MongoDB 4.2+ 默认支持后台索引构建（非阻塞），但仍会消耗 CPU/IO。
```javascript
// 检查索引构建状态
db.currentOp({ "msg": { $regex: "IndexBuilds" } })

// 暂停/恢复索引构建
db.adminCommand({ pauseIndexBuilds: 1 })
db.adminCommand({ resumeIndexBuilds: 1 })
```

### 11.6 诊断命令速查表

```javascript
// 服务器状态
db.serverStatus()
// 复制集状态
rs.status()
// 分片状态
sh.status()
// 锁信息
db.adminCommand({ currentLocks: 1 })
// 连接数
db.serverStatus().connections
// 内存使用
db.serverStatus().mem
```

---

## 附录：快速参考

| 操作 | mongosh 命令 |
|------|-------------|
| 创建索引 | `db.col.createIndex({ field: 1 })` |
| 查看执行计划 | `db.col.find(...).explain("executionStats")` |
| 启动事务 | `const s = db.getMongo().startSession()` |
| 备份 | `mongodump --uri="..." --gzip` |
| 开启分析器 | `db.setProfilingLevel(1)` |
| 查看用户 | `db.getUsers()` |
| 授权模式 | `security.authorization: enabled` |

> 📌 本文档基于 MongoDB 5.0/6.0/7.0 特性编写。部分语法在不同小版本间可能有细微差异，请以官方文档与 `mongosh --version` 输出为准。生产部署前请务必在测试环境充分验证。