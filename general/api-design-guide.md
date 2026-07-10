# API 设计指南

## RESTful 设计原则

### URL 设计

```
GET    /api/users          # 获取用户列表
GET    /api/users/:id      # 获取单个用户
POST   /api/users          # 创建用户
PUT    /api/users/:id      # 更新用户
DELETE /api/users/:id      # 删除用户

# 嵌套资源
GET    /api/users/:id/orders    # 获取用户的订单
POST   /api/users/:id/orders    # 为用户创建订单
```

### 响应格式

```json
// 成功
{
  "success": true,
  "data": { ... }
}

// 列表
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100
  }
}

// 错误
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format"
  }
}
```

### 状态码

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 200 | OK | 成功 |
| 201 | Created | 创建成功 |
| 204 | No Content | 删除成功 |
| 400 | Bad Request | 参数错误 |
| 401 | Unauthorized | 未认证 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 500 | Internal Server Error | 服务器错误 |

## 版本控制

```yaml
# URL 版本
/api/v1/users

# Header 版本
Accept: application/vnd.api.v1+json

# 查询参数
/api/users?version=1
```

## 分页

```python
# Cursor 分页（推荐）
GET /api/messages?cursor=123&limit=20

# Offset 分页
GET /api/users?page=2&limit=10
```

## 过滤和排序

```python
# 过滤
GET /api/users?status=active&age_min=18

# 排序
GET /api/users?sort=-created_at,name
```