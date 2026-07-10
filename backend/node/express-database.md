# Express 数据库集成

## MongoDB (Mongoose)

```javascript
const mongoose = require('mongoose')

// 连接
mongoose.connect('mongodb://localhost:27017/myapp')

// Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  age: { type: Number, min: 0 },
  createdAt: { type: Date, default: Date.now }
})

// Model
const User = mongoose.model('User', userSchema)

// CRUD
// 创建
const user = new User({ name: 'John', email: 'john@example.com' })
await user.save()

// 查询
const users = await User.find({ age: { $gte: 18 } })

// 更新
await User.findByIdAndUpdate(id, { name: 'Jane' })

// 删除
await User.findByIdAndDelete(id)
```

## PostgreSQL (pg)

```javascript
const { Pool } = require('pg')

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'myapp',
  user: 'postgres',
  password: 'password'
})

// 查询
async function getUsers() {
  const result = await pool.query('SELECT * FROM users')
  return result.rows
}

// 参数化查询
async function getUser(id) {
  const result = await pool.query('SELECT * FROM users WHERE id = $1', [id])
  return result.rows[0]
}

// 事务
async function transfer(fromId, toId, amount) {
  const client = await pool.connect()
  try {
    await client.query('BEGIN')
    await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, fromId])
    await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, toId])
    await client.query('COMMIT')
  } catch (e) {
    await client.query('ROLLBACK')
    throw e
  } finally {
    client.release()
  }
}
```

## Redis

```javascript
const redis = require('redis')
const client = redis.createClient()

// 连接
client.on('connect', () => {
  console.log('Redis connected')
})

// 缓存
async function getCached(key) {
  return new Promise((resolve, reject) => {
    client.get(key, (err, data) => {
      if (err) reject(err)
      resolve(data ? JSON.parse(data) : null)
    })
  })
}

async function setCache(key, value, ttl = 3600) {
  return new Promise((resolve, reject) => {
    client.setex(key, ttl, JSON.stringify(value), (err) => {
      if (err) reject(err)
      resolve()
    })
  })
}
```