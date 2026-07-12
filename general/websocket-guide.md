# WebSocket 详解

## 什么是 WebSocket？

WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议。

## 与 HTTP 对比

| 特性 | HTTP | WebSocket |
|------|------|-----------|
| 通信方式 | 请求-响应 | 全双工 |
| 连接 | 短连接 | 长连接 |
| 服务推送 | 不支持 | 支持 |

## Node.js 实现

```javascript
const WebSocket = require('ws')
const wss = new WebSocket.Server({ port: 8080 })

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    ws.send(`Echo: ${message}`)
  })
})
```

## Python 实现

```python
import asyncio
import websockets

async def handler(websocket, path):
    async for message in websocket:
        await websocket.send(f"Echo: {message}")

async def main():
    async with websockets.serve(handler, "localhost", 8765):
        await asyncio.Future()

asyncio.run(main())
```

## 心跳机制

```javascript
setInterval(() => {
  ws.ping()
}, 30000)

ws.on('pong', () => { ws.isAlive = true })
```

## 断线重连

```javascript
function connect() {
  const ws = new WebSocket(url)
  ws.onclose = () => setTimeout(connect, 1000)
}
```
