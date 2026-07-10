---
name: backend-knowledge
description: 获取、讲解与沉淀后端开发知识（REST/GraphQL、认证、分层架构、中间件、消息队列、微服务）。用户学习后端、写 backend/ 笔记、问 API/鉴权/事务/并发时使用。
---

# 后端知识技能

## 触发后动作

1. Read `backend/` 已有笔记
2. 遵循 `.cursor/rules/backend-learning.mdc`
3. 落盘时用 `write-learning-note` 模板

## 讲解顺序（零基础）

生活类比 → 业务场景 → API 契约 → 分层实现 → **带注释代码** → 逐步讲解 → 错误/日志 → 安全
## 必含元素（按主题选用）

- **HTTP 语义**：方法、状态码、Header、幂等
- **分层**：Controller / Service / Repository 职责
- **认证授权**：Session vs JWT、RBAC
- **可靠性**：超时、重试、熔断、限流
- **数据一致性**：本地事务 vs 分布式事务/Saga

## API 文档片段

```json
// POST /api/v1/resources — POST 表示「新建」；URL 是接口地址
// 请求体：要创建的资源数据
{ "name": "example" }  // name 字段：资源名称

// 成功响应 201 — 201 表示「已创建」
{ "id": "uuid", "name": "example" }  // id：系统分配的唯一编号

// 错误响应 400 — 400 表示「请求数据有问题」
{ "code": "VALIDATION_ERROR", "message": "...", "details": [] }
```

## 权威资源

详见 [resources.md](resources.md)

## 练习粒度

CRUD → 鉴权 → 分页 → 缓存 → 异步任务，每步可 curl/Postman 验证
