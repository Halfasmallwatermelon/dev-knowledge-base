# WebSocket 详解

## 1. 概述
WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议。它通过一次 HTTP/HTTPS 握手建立连接后，客户端和服务器可以随时相互发送数据，无需重复请求响应流程。适用于即时通讯、实时数据推送、在线游戏、协同编辑等低延迟场景。

```javascript
// 概念示例：WebSocket 通信模型
// 客户端 <-> 服务器（持久连接，双向实时通信）
const ws = new WebSocket('wss://example.com/socket');
ws.onopen = () => console.log('连接已建立');
ws.onmessage = (event) => console.log('收到:', event.data);
```

---

## 2. 核心概念
- **连接生命周期**：握手 → `OPEN` → 数据传输 → 关闭
- **帧（Frame）**：协议最小传输单元，包含操作码（文本/二进制/控制帧）、掩码、负载长度与数据
- **状态码**：`CONNECTING(0)`, `OPEN(1)`, `CLOSING(2)`, `CLOSED(3)`
- **控制帧**：`PING`/`PONG`（保活）、`CLOSE`（断开）、`CONTINUATION`（分片）

```javascript
// 监听连接状态变化
ws.addEventListener('open', () => console.log('状态: OPEN'));
ws.addEventListener('close', (e) => console.log(`断开: code=${e.code}, reason=${e.reason}`));
ws.addEventListener('error', (e) => console.error('连接错误'));
```

---

## 3. 协议原理
WebSocket 依赖 HTTP 升级机制完成握手：
1. 客户端发送带 `Upgrade: websocket` 和 `Sec-WebSocket-Key` 的请求
2. 服务器验证后返回 `101 Switching Protocols` 及计算好的 `Sec-WebSocket-Accept`
3. 后续数据以自定义帧格式传输，不再遵循 HTTP 报文结构

```http
# 客户端握手请求
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# 服务端握手响应
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

---

## 4. 与 HTTP 对比
| 特性          | HTTP/HTTPS              | WebSocket               |
|---------------|-------------------------|-------------------------|
| 通信模式      | 半双工，请求-响应       | 全双工，双向实时        |
| 连接维持      | 短连接（默认）或 Keep-Alive | 长连接，持久化          |
| 头部开销      | 每次请求携带完整 Header | 握手后仅 2~14 字节帧头  |
| 状态管理      | 无状态                  | 有状态（会话上下文）    |
| 适用场景      | 常规 API、页面加载      | 实时交互、高频推送      |

```javascript
// HTTP 轮询 vs WebSocket
// HTTP 轮询：客户端定时请求，服务器被动响应
setInterval(() => fetch('/api/data').then(r => r.json()), 3000);

// WebSocket：服务器主动推送，延迟极低
ws.send(JSON.stringify({ action: 'subscribe', channel: 'updates' }));
// 服务器收到后可随时调用 ws.send() 推送
```

---

## 5. Node.js 实现
使用 `ws` 库（目前最流行的 Node.js WebSocket 实现）搭建基础服务器。

```javascript
const { WebSocketServer } = require('ws');

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('新客户端连接:', req.url);
  
  // 接收消息
  ws.on('message', (data) => {
    const msg = JSON.parse(data.toString());
    console.log('收到:', msg);
    ws.send(JSON.stringify({ echo: msg }));
  });

  // 断开处理
  ws.on('close', (code, reason) => {
    console.log('客户端断开:', code, reason.toString());
  });

  // 发送欢迎消息
  ws.send(JSON.stringify({ type: 'welcome', id: Date.now() }));
});

console.log('WebSocket 服务器运行在 ws://localhost:8080');
```

---

## 6. 前端 JavaScript 客户端
原生浏览器 API 即可使用，支持 `ws://` 和 `wss://`。

```javascript
const socket = new WebSocket('wss://localhost:8080');

socket.onopen = () => {
  console.log('已连接');
  socket.send(JSON.stringify({ action: 'login', user: 'alice' }));
};

socket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'welcome') {
    console.log('欢迎消息:', data.id);
  }
};

socket.onerror = (err) => console.error('WebSocket 错误:', err);

socket.onclose = (e) => {
  console.log(`连接关闭: ${e.code} ${e.reason}`);
  // 可在此触发重连逻辑
};
```

---

## 7. 心跳机制
防止防火墙/NAT 超时断开连接，并检测僵尸连接。推荐使用协议内置的 `PING`/`PONG` 控制帧。

```javascript
// 服务端心跳配置
const HEARTBEAT_INTERVAL = 30000;
const HEARTBEAT_TIMEOUT = 10000;

function startHeartbeat(ws) {
  let pongReceived = true;
  
  ws.on('pong', () => { pongReceived = true; });
  
  const interval = setInterval(() => {
    if (!pongReceived) {
      console.log('客户端无响应，强制断开');
      clearInterval(interval);
      ws.terminate();
      return;
    }
    pongReceived = false;
    ws.ping(); // 发送 PING 控制帧
  }, HEARTBEAT_INTERVAL);

  ws.on('close', () => clearInterval(interval));
}

// 在 connection 事件中调用 startHeartbeat(ws);
```

---

## 8. 断线重连
网络波动会导致意外断开，客户端需实现指数退避重连。

```javascript
class ReconnectingSocket {
  constructor(url) {
    this.url = url;
    this.socket = null;
    this.reconnectDelay = 1000;
    this.maxDelay = 30000;
    this.connect();
  }

  connect() {
    this.socket = new WebSocket(this.url);
    
    this.socket.onopen = () => {
      console.log('连接成功');
      this.reconnectDelay = 1000; // 重置延迟
    };

    this.socket.onclose = (e) => {
      console.warn(`连接断开 (${e.code})，${this.reconnectDelay/1000}s 后重连...`);
      setTimeout(() => this.connect(), this.reconnectDelay);
      this.reconnectDelay = Math.min(this.reconnectDelay * 2, this.maxDelay);
    };

    this.socket.onmessage = (e) => {
      console.log('收到:', e.data);
    };
  }
}

// 使用: new ReconnectingSocket('wss://example.com/ws');
```

---

## 9. 消息广播
向所有已连接客户端推送消息。

```javascript
const clients = new Set();

wss.on('connection', (ws) => {
  clients.add(ws);
  ws.on('close', () => clients.delete(ws));
});

function broadcast(message) {
  const data = JSON.stringify(message);
  clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(data);
    }
  });
}

// 调用示例: broadcast({ type: 'system', text: '服务器维护中' });
```

---

## 10. 房间管理
支持用户加入/离开特定频道，实现定向推送。

```javascript
class RoomManager {
  constructor() {
    this.rooms = new Map(); // roomId -> Set<WebSocket>
  }

  join(ws, roomId) {
    if (!this.rooms.has(roomId)) this.rooms.set(roomId, new Set());
    this.rooms.get(roomId).add(ws);
    ws.roomId = roomId;
    console.log(`客户端加入房间: ${roomId}`);
  }

  leave(ws) {
    if (ws.roomId && this.rooms.has(ws.roomId)) {
      this.rooms.get(ws.roomId).delete(ws);
      if (this.rooms.get(ws.roomId).size === 0) {
        this.rooms.delete(ws.roomId);
      }
    }
  }

  sendToRoom(roomId, message) {
    const room = this.rooms.get(roomId);
    if (!room) return;
    const data = JSON.stringify(message);
    room.forEach(ws => {
      if (ws.readyState === WebSocket.OPEN) ws.send(data);
    });
  }
}

// 使用示例:
// const roomMgr = new RoomManager();
// roomMgr.join(ws, 'game_lobby');
// roomMgr.sendToRoom('game_lobby', { type: 'player_joined', user: 'bob' });
```

---

## 11. 安全性
生产环境必须启用 WSS（TLS），并实施认证、限流、源校验。

```javascript
const wss = new WebSocketServer({
  port: 443,
  verifyClient: ({ origin, req }) => {
    // 1. 校验 Origin 防 CSRF
    if (!['https://yourdomain.com'].includes(origin)) return false;
    // 2. 简单 Token 鉴权（实际建议放在业务层）
    const token = req.headers.cookie?.match(/token=([^;]+)/)?.[1];
    if (!token || !isValidToken(token)) return false;
    return true;
  }
});

// 消息内容安全过滤
ws.on('message', (raw) => {
  try {
    const msg = JSON.parse(raw.toString());
    if (msg.content && msg.content.length > 2000) throw new Error('Payload too long');
    // 转义 HTML/脚本注入风险
    msg.content = escapeHtml(msg.content);
    ws.send(JSON.stringify({ ok: true }));
  } catch (err) {
    ws.close(1008, 'Invalid message');
  }
});
```

---

## 12. 性能优化
- 启用帧压缩 (`permessage-deflate`)
- 限制单条消息大小
- 批量发送合并
- 连接池与负载均衡

```javascript
const wss = new WebSocketServer({
  port: 8080,
  compression: true,           // 启用 permessage-deflate
  maxPayload: 1024 * 1024,     // 限制 1MB
  clientTracking: true
});

// 批量发送优化（减少系统调用）
function batchSend(clients, messages) {
  const buffer = messages.map(m => JSON.stringify(m));
  clients.forEach(c => {
    if (c.readyState === c.OPEN) {
      // 实际项目中可使用 Buffer.concat 或 stream 提升吞吐
      buffer.forEach(b => c.send(b));
    }
  });
}
```

---

## 13. 最佳实践
1. **始终使用 WSS**：明文 WebSocket 易被中间人篡改
2. **设计应用层协议**：定义统一消息格式 `{ type, payload, seq, timestamp }`
3. **优雅降级**：浏览器不支持时自动切换为 SSE 或长轮询
4. **监控指标**：记录连接数、消息吞吐量、延迟、错误率
5. **服务治理**：集群部署时配合 Redis Pub/Sub 或消息队列实现跨节点广播

```javascript
// 统一消息格式示例
const sendMessage = (ws, type, payload) => {
  ws.send(JSON.stringify({
    type,
    payload,
    seq: ++seqCounter,
    ts: Date.now()
  }));
};
```

---

## 14. 常见问题
| 问题                      | 原因/解决方案                                                                 |
|---------------------------|------------------------------------------------------------------------------|
| 跨域失败 `Origin not allowed` | 服务端未配置 `verifyClient` 或 `origin` 白名单；开发环境可临时关闭严格校验 |
| 代理断开连接              | Nginx/Apache 需配置 `proxy_set_header Upgrade $http_upgrade;` 等参数        |
| 消息乱序/重复             | 网络重试导致；建议在消息体加唯一 `id` 或序列号做幂等处理                   |
| 大文件传输卡顿            | 避免直接传 Base64；改用分片上传或独立存储后传 URL                          |
| 浏览器标签休眠断连        | iOS Safari 会暂停后台 WebSocket；需结合 App 推送或 Service Worker 保活     |

```nginx
# Nginx WebSocket 代理配置示例
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400s;
}
```

---

*本文档基于 RFC 6455 规范及现代工程实践编写，代码示例可直接集成至生产项目。实际部署请结合业务规模进行压测与容量规划。*
