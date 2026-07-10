# Docker 最佳实践

## Dockerfile 优化

```dockerfile
# 多阶段构建
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

## 镜像优化

```dockerfile
# 使用小基础镜像
FROM alpine:3.18

# 合并 RUN 指令
RUN apk add --no-cache nodejs npm && \
    npm install && \
    npm cache clean --force

# 使用非 root 用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup
USER appuser
```

## Docker Compose 最佳实践

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

## 安全

```bash
# 扫描镜像漏洞
docker scout cves myimage:latest

# 使用可信基础镜像
FROM docker.io/library/node:18-alpine

# 不存储密钥
docker build --build-arg API_KEY=xxx .  # ❌
docker run -e API_KEY=xxx myimage       # ✅
```