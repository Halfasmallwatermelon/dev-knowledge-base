# Express.js 概述

## 什么是 Express？

Express 是 Node.js 最流行的 Web 框架，简洁灵活。

## 快速开始

```javascript
const express = require('express')
const app = express()

app.get('/', (req, res) => {
  res.json({ message: 'Hello World' })
})

app.listen(3000, () => {
  console.log('Server running on port 3000')
})
```

## 路由

```javascript
// GET
app.get('/users', (req, res) => {
  res.json(users)
})

// POST
app.post('/users', (req, res) => {
  const user = req.body
  users.push(user)
  res.status(201).json(user)
})

// 路由参数
app.get('/users/:id', (req, res) => {
  const user = users.find(u => u.id === req.params.id)
  res.json(user)
})
```

## 中间件

```javascript
// 日志中间件
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`)
  next()
})

// 解析 JSON
app.use(express.json())

// 错误处理
app.use((err, req, res, next) => {
  console.error(err.stack)
  res.status(500).json({ error: 'Something went wrong!' })
})
```

## 路由模块化

```javascript
// routes/users.js
const express = require('express')
const router = express.Router()

router.get('/', (req, res) => {
  res.json(users)
})

router.post('/', (req, res) => {
  res.status(201).json(req.body)
})

module.exports = router

// app.js
const userRoutes = require('./routes/users')
app.use('/api/users', userRoutes)
```

## 静态文件

```javascript
app.use(express.static('public'))
// 访问 public/index.html: http://localhost:3000/index.html
```

## 模板引擎

```javascript
app.set('view engine', 'ejs')

app.get('/profile', (req, res) => {
  res.render('profile', { name: 'John' })
})
```