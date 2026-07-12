# GraphQL 指南

## 什么是 GraphQL？

GraphQL 是一种用于 API 的查询语言，让客户端可以精确请求所需数据。

## 与 REST 对比

| 特性 | REST | GraphQL |
|------|------|---------|
| 端点 | 多个 | 单一 |
| 数据获取 | 过度/不足 | 精确 |
| 版本控制 | URL 版本 | Schema 演进 |

## Schema 定义

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  users: [User!]!
  user(id: ID!): User
  posts: [Post!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
  createPost(title: String!, content: String!, authorId: ID!): Post!
}
```

## Apollo Server

```javascript
const { ApolloServer, gql } = require('apollo-server')

const typeDefs = gql`
  type Query {
    hello: String
  }
`

const resolvers = {
  Query: {
    hello: () => 'Hello world!'
  }
}

const server = new ApolloServer({ typeDefs, resolvers })
server.listen().then(({ url }) => console.log(`Server ready at ${url}`))
```

## Python (Strawberry)

```python
import strawberry
from typing import List

@strawberry.type
class User:
    id: int
    name: str
    email: str

@strawberry.type
class Query:
    @strawberry.field
    def users(self) -> List[User]:
        return [User(id=1, name="John", email="john@example.com")]

schema = strawberry.Schema(query=Query)
```
