# 微服务架构指南

## 什么是微服务？

微服务是一种架构风格，将应用程序构建为一组小型、独立的服务。

## 核心原则

1. **单一职责**：每个服务只做一件事
2. **独立部署**：可以独立部署和扩展
3. **去中心化治理**：每个服务可以选择自己的技术栈
4. **容错设计**：服务故障不影响其他服务

## 服务通信

### 同步通信（HTTP/REST）

```python
# 服务A调用服务B
import requests

response = requests.get('http://service-b/api/users/1')
user = response.json()
```

### 异步通信（消息队列）

```python
# 生产者
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='task_queue')
channel.basic_publish(exchange='', routing_key='task_queue', body='Hello')

# 消费者
def callback(ch, method, properties, body):
    print(f"Received: {body}")

channel.basic_consume(queue='task_queue', on_message_callback=callback)
channel.start_consuming()
```

## 服务发现

```yaml
# Docker Compose
services:
  user-service:
    image: user-service
    networks:
      - app-network
  
  order-service:
    image: order-service
    environment:
      - USER_SERVICE_URL=http://user-service:3000
    networks:
      - app-network

networks:
  app-network:
```

## API 网关

```python
# FastAPI 作为网关
from fastapi import FastAPI
import requests

app = FastAPI()

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    response = requests.get(f"http://user-service:3000/users/{user_id}")
    return response.json()

@app.get("/api/orders/{order_id}")
async def get_order(order_id: int):
    response = requests.get(f"http://order-service:3001/orders/{order_id}")
    return response.json()
```

## 配置管理

```yaml
# 共享配置
shared_config:
  database:
    host: db-service
    port: 5432
  redis:
    host: redis-service
    port: 6379

# 服务配置
user_service:
  port: 3000
  database: *shared_config
```

## 服务治理

### 熔断器

```python
import circuitbreaker

@circuitbreaker.circuit(failure_threshold=5, recovery_timeout=60)
def call_external_service():
    response = requests.get('http://external-service/api')
    return response.json()
```

### 限流

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter

@app.get("/api/data")
@limiter.limit("100/minute")
async def get_data():
    return {"data": "value"}
```

### 负载均衡

```nginx
# Nginx 负载均衡
upstream backend {
    server backend1:3000
    server backend2:3000
    server backend3:3000
}

server {
    listen 80
    
    location /api/ {
        proxy_pass http://backend
    }
}
```

## 部署策略

### 容器化

```dockerfile
# 用户服务
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    spec:
      containers:
      - name: user-service
        image: user-service:latest
        ports:
        - containerPort: 3000
```

## 监控和日志

```python
# 结构化日志
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'service': 'user-service'
        })

logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
```