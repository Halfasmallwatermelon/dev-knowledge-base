# GraphQL 指南

## 1. 概述

GraphQL 是由 Facebook 于 2015 年开源的 API 查询语言与运行时，旨在解决传统 REST API 在数据获取灵活性、版本管理与网络效率方面的痛点。它允许客户端精确声明所需的数据结构，服务端通过强类型 Schema 进行响应。与 REST 的多端点、固定响应格式不同，GraphQL 采用单一端点，通过声明式查询实现按需获取、批量请求与实时订阅，显著降低前后端耦合度与无效数据传输。

## 2. 核心概念

- **Schema（模式）**：GraphQL 的契约层，定义所有可用类型、字段及操作接口。
- **类型系统**：包含标量类型（`String`, `Int`, `Float`, `Boolean`, `ID`）、对象类型、接口、联合类型、枚举与输入类型。
- **Resolvers（解析器）**：将查询/变更请求映射到具体数据源的异步函数，负责数据获取、转换与权限校验。
- **Queries（查询）**：只读操作，支持嵌套、变量、片段与别名，返回结构严格匹配请求。
- **Mutations（变更）**：写操作，用于创建、更新或删除数据，默认串行执行以保证状态一致性。
- **Subscriptions（订阅）**：基于 WebSocket 的实时推送机制，适用于事件流或即时通知场景。
- **Introspection（自省）**：允许客户端动态查询 Schema 元数据，是 GraphiQL、类型提示与代码生成的基础。

## 3. Schema 定义

GraphQL 使用 SDL（Schema Definition Language）描述模式。以下为核心语法示例：

```graphql
# 查询入口
type Query {
  user(id: ID!): User
  users(limit: Int = 10, offset: Int = 0): [User!]!
}

# 变更入口
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User
  deleteUser(id: ID!): Boolean!
}

# 对象类型
type User {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
  posts: [Post!]!
}

# 枚举类型
enum UserRole {
  ADMIN
  USER
  GUEST
}

# 输入类型（仅用于 Mutation 参数）
input CreateUserInput {
  name: String!
  email: String!
  role: UserRole
}

# 接口与联合类型
interface Node {
  id: ID!
}

type Post implements Node {
  id: ID!
  title: String!
  content: String
  author: User!
}

union SearchResult = User | Post
```

**语法说明**：`!` 表示字段不可为空，`[]` 表示列表，输入类型必须使用 `input` 关键字。

## 4. 查询与变更

### 查询示例

```graphql
query GetUserPosts($userId: ID!, $limit: Int = 5) {
  user(id: $userId) {
    name
    email
    posts(limit: $limit) {
      title
      createdAt
    }
  }
}
```

- 支持变量绑定、字段别名（`myUser: user(id: "1")`）与片段复用（`fragment UserInfo on User { ... }`）。
- 响应结构严格遵循请求树，未请求的字段不会出现在结果中。

### 变更示例

```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    author { name }
  }
}
```

- 变更操作按提交顺序串行执行，避免并发写入冲突。
- 推荐返回完整资源对象以便客户端直接更新缓存。

### 订阅示例

```graphql
subscription OnNewPost($userId: ID!) {
  postCreated(userId: $userId) {
    id
    title
  }
}
```

需配合 WebSocket 协议与事件总线（如 Redis Pub/Sub）实现实时推送。

## 5. Apollo Server 实战

基于现代 v4 版本的快速搭建示例：

```javascript
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

const typeDefs = `#graphql
  type Query {
    hello(name: String): String
  }

  type Mutation {
    increment(count: Int!): Int!
  }
`;

let counter = 0;

const resolvers = {
  Query: {
    hello: (_, { name }) => `Hello, ${name || 'World'}!`,
  },
  Mutation: {
    increment: () => ++counter,
  },
};

async function startServer() {
  const server = new ApolloServer({ typeDefs, resolvers });
  const { url } = await startStandaloneServer(server, {
    listen: { port: 4000 },
  });
  console.log(`🚀 GraphQL Server ready at ${url}`);
}

startServer().catch(console.error);
```

**关键特性**：

- 内置错误捕获与堆栈过滤（生产环境不暴露内部细节）
- 插件系统支持日志记录、速率限制、缓存控制
- 原生支持 Express/Fastify/Koa 等 Web 框架

## 6. 代码生成

前端通过工具自动生成类型安全的查询代码，消除手动维护接口类型的成本。推荐使用 **GraphQL Code Generator**。

`codegen.yml` 配置示例：

```yaml
schema: "http://localhost:4000/graphql"
documents: "src/**/*.graphql"
generates:
  src/types/generated.ts:
    plugins:
      - "typescript"
      - "typescript-operations"
      - "typescript-react-query"
  src/gql/:
    preset: "client"
    plugins: []
```

**生成收益**：

- 完整的 TypeScript 类型推导
- 自动匹配的 Hook/组件（如 `useGetUserPosts`）
- 编译时检查字段拼写与类型不匹配
- 支持 React Query、SWR、Vue Query 等主流数据获取库

## 7. 性能优化

| 问题 | 解决方案 |
|------|----------|
| **N+1 查询** | 使用 `DataLoader` 批量加载与缓存同批次请求，将多次 DB 调用合并为单次批量查询 |
| **查询过深/过宽** | 部署 `graphql-depth-limit` 与 `query-complexity` 插件，限制嵌套层级与字段权重 |
| **缓存缺失** | 结合 HTTP `ETag`/`Cache-Control`、客户端缓存（Apollo Client Cache）与持久化查询（PQ） |
| **大数据分页慢** | 采用游标分页（Cursor-based）替代偏移分页，避免 `OFFSET` 全表扫描 |
| **网络传输冗余** | 客户端按需请求字段，服务端对敏感/大对象实施懒加载或分片返回 |

## 8. 最佳实践

1. **契约驱动开发**：前后端基于 Schema 并行开发，利用 CI 流程验证变更兼容性。
2. **错误标准化**：统一错误结构 `{ code, message, details }`，区分业务校验失败与系统异常。
3. **平滑版本演进**：使用 `@deprecated(reason: "...")` 标记废弃字段，禁止直接删除强类型字段。
4. **安全加固**：启用 CORS 白名单、过滤用户输入、限制查询复杂度、对鉴权字段实施中间件拦截。
5. **文档自动化**：通过 Schema 注释或 `graphql-markdown` 生成结构化 API 文档，保持与代码同步。
6. **测试策略**：单元测试 Resolver 逻辑，集成测试验证端到端查询，使用 Mock Server 隔离外部依赖。

## 9. 常见问题

**Q：GraphQL 会取代 REST 吗？**

A：不会。两者定位互补：REST 适合资源导向的简单 CRUD 与标准 HTTP 缓存场景；GraphQL 适合复杂数据聚合、多端适配与高频迭代。实际项目中常混合使用（如 REST 处理文件上传/支付回调，GraphQL 处理核心业务数据）。

**Q：学习曲线是否陡峭？**

A：初期需掌握 Schema 设计与 Resolver 编写规范，但建立工程模板后，协作效率显著提升。社区提供大量脚手架、类型生成工具与调试面板，大幅降低上手门槛。

**Q：如何防止恶意查询耗尽服务器资源？**

A：部署查询复杂度分析插件、限制最大查询深度、启用速率限制（Rate Limiting），并对高频查询实施客户端缓存与持久化。

**Q：生产环境部署注意事项？**

A：需配置 WebSocket 代理支持订阅；CDN 仅缓存静态 Schema 或公开端点；启用健康检查 `/health`；关闭 GraphiQL/Playground 的交互式调试面板。

**Q：与 REST 相比的优缺点总结**

- ✅ 优势：按需获取、类型安全、强契约、减少往返次数、实时订阅原生支持
- ⚠️ 注意：缓存策略更复杂、学习成本略高、不适合纯静态资源分发、需规范 Resolver 设计避免性能陷阱
