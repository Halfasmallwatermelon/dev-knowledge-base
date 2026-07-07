---
name: backend-knowledge
description: 获取、讲解与沉淀后端开发知识（REST/GraphQL、认证、分层架构、中间件、消息队列、微服务）。用户学习后端、写 backend/ 笔记、问 API/鉴权/事务/并发时使用。
---

# 后端知识技能

## 触发后动作

1. Read `backend/` 已有笔记
2. 遵循 `.cursor/rules/backend-learning.mdc`
3. 落盘时用 `write-learning-note` 模板

## 讲解顺序

业务场景 → API 契约 → 分层实现 → 错误/日志 → 安全与可靠性

## 必含元素（按主题选用）

- **HTTP 语义**：方法、状态码、Header、幂等
- **分层**：Controller / Service / Repository 职责
- **认证授权**：Session vs JWT、RBAC
- **可靠性**：超时、重试、熔断、限流
- **数据一致性**：本地事务 vs 分布式事务/Saga

## API 文档片段

```json
// 请求
POST /api/v1/resources
{ "name": "example" }

// 成功 201
{ "id": "uuid", "name": "example" }

// 错误 400
{ "code": "VALIDATION_ERROR", "message": "...", "details": [] }
```

## 权威资源

详见 [resources.md](resources.md)

## 练习粒度

CRUD → 鉴权 → 分页 → 缓存 → 异步任务，每步可 curl/Postman 验证
