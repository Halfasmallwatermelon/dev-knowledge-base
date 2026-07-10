# CI/CD 指南

## 什么是 CI/CD？

- **CI（持续集成）**：代码合并后自动构建和测试
- **CD（持续部署）**：代码通过测试后自动部署

## GitHub Actions

### 基本流程

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm ci --only=production
            pm2 restart myapp
```

### Docker 构建

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/myapp:${{ github.ref_name }}
```

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  image: node:18
  script:
    - npm ci
    - npm test

build:
  stage: build
  image: docker:latest
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA

deploy:
  stage: deploy
  script:
    - ssh user@server "docker pull myapp:$CI_COMMIT_SHA && docker-compose up -d"
  only:
    - main
```

## 部署策略

### 蓝绿部署

```bash
# 当前版本（蓝）
docker run -d --name blue -p 80:3000 myapp:v1

# 新版本（绿）
docker run -d --name green -p 81:3000 myapp:v2

# 切换流量
nginx -s reload  # 切换到 green

# 删除旧版本
docker stop blue
docker rm blue
```

### 滚动更新

```yaml
# Kubernetes
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

## 环境管理

```yaml
# 环境变量
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: url
  - name: API_KEY
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: api-key
```