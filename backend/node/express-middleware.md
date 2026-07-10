# Express 中间件详解

## 什么是中间件？

中间件是 Express 的核心概念，可以访问请求、响应和下一个中间件函数。

## 内置中间件

```javascript
const express = require('express')
const app = express()

// 解析 JSON
app.use(express.json())

// 解析 URL 编码
app.use(express.urlencoded({ extended: true }))

// 静态文件
app.use(express.static('public'))
```

## 自定义中间件

```javascript
// 日志中间件
function logger(req, res, next) {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`)
  next()
}

app.use(logger)

// 认证中间件
function auth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1]
  if (!token) {
    return res.status(401).json({ error: 'No token' })
  }
  
  try {
    const decoded = jwt.verify(token, SECRET_KEY)
    req.user = decoded
    next()
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' })
  }
}

// 路由保护
app.get('/api/profile', auth, (req, res) => {
  res.json({ user: req.user })
})
```

## 路由级中间件

```javascript
const router = express.Router()

// 路由级中间件
router.use((req, res, next) => {
  console.log('Router middleware')
  next()
})

router.get('/', (req, res) => {
  res.json({ message: 'Home' })
})

app.use('/api', router)
```

## 错误处理中间件

```javascript
// 错误处理中间件（4个参数）
app.use((err, req, res, next) => {
  console.error(err.stack)
  res.status(500).json({ error: 'Something went wrong!' })
})

// 异步错误处理
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next)
  }
}

app.get('/api/users', asyncHandler(async (req, res) => {
  const users = await User.findAll()
  res.json(users)
}))
```

## 第三方中间件

```javascript
// CORS
const cors = require('cors')
app.use(cors())

// Helmet（安全）
const helmet = require('helmet')
app.use(helmet())

// Morgan（日志）
const morgan = require('morgan')
app.use(morgan('combined'))

// Rate Limiting
const rateLimit = require('express-rate-limit')
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
})
app.use('/api/', limiter)
```