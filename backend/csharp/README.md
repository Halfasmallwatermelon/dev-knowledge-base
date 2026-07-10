# C# / .NET 后端

> 关键词：C#、.NET、ASP.NET Core | 前置知识：HTTP 基础、面向对象入门 | 难度：入门

## 概述

**C#**（读作「C Sharp」）是微软设计的编程语言；**.NET** 是运行 C# 程序的平台（类似 Java 需要 JVM）。**ASP.NET Core** 是在 .NET 上构建 Web API 的框架——前端发 HTTP 请求，它接收、处理、返回 JSON。

可以把整条链路想成一家餐厅：

| 角色 | 对应技术 | 做什么 |
|------|----------|--------|
| 顾客 | 前端 / 手机 App | 点菜（发请求） |
| 前台 | Controller / Minimal API | 接单、核对、上菜 |
| 厨房 | Service 业务层 | 按规则做菜 |
| 仓库 | Repository / EF Core | 取食材、存库存 |
| 传菜通道 | 中间件管道 | 安检、计时、权限检查 |

本目录按「先跑起来 → 再懂原理 → 再写完整 API → 进阶能力」组织笔记，适合零基础按顺序阅读。

## 学习路线

### 第一阶段：跑起来与核心机制

| 顺序 | 文档 | 说明 |
|------|------|------|
| 1 | [aspnet-core-overview.md](aspnet-core-overview.md) | 框架是什么、项目长什么样、请求怎么进来 |
| 2 | [configuration-and-logging.md](configuration-and-logging.md) | 配置文件、环境变量、ILogger 日志 |
| 3 | [dependency-injection.md](dependency-injection.md) | 依赖注入：框架帮你「组装」对象 |
| 4 | [middleware-pipeline.md](middleware-pipeline.md) | 中间件：请求经过的一道道「关卡」 |

### 第二阶段：完整 API

| 顺序 | 文档 | 说明 |
|------|------|------|
| 5 | [api-development.md](api-development.md) | 写 REST API：路由、分层、错误响应 |
| 6 | [validation-and-pagination.md](validation-and-pagination.md) | 输入校验、分页 Query、ProblemDetails |
| 7 | [entity-framework-core.md](entity-framework-core.md) | 用 C# 操作数据库、迁移、Repository |
| 8 | [authentication-jwt.md](authentication-jwt.md) | 登录与 JWT：谁可以访问哪些接口 |

### 第三阶段：进阶与质量

| 顺序 | 文档 | 说明 |
|------|------|------|
| 9 | [caching.md](caching.md) | 内存缓存与 Redis，减轻数据库压力 |
| 10 | [background-services.md](background-services.md) | 后台任务、Channel 队列、定时清理 |
| 11 | [integration-testing.md](integration-testing.md) | WebApplicationFactory 集成测试 |

## 环境准备

**第 1 步：安装 .NET SDK**

从 [.NET 下载页](https://dotnet.microsoft.com/download) 安装 **LTS（长期支持）** 版本（如 .NET 8）。SDK 里包含 `dotnet` 命令行工具。

**第 2 步：检查是否安装成功**

```powershell
# 在 PowerShell 或 CMD 中执行；应输出版本号，如 8.0.xxx
dotnet --version
```

**第 3 步：创建并运行第一个 API**

```powershell
# 创建 Web API 项目；--use-minimal-apis 表示用轻量写法
dotnet new webapi -n MyApi -o MyApi --use-minimal-apis

# 进入项目目录
cd MyApi

# 启动开发服务器；终端会显示监听地址，如 http://localhost:5xxx
dotnet run
```

浏览器访问终端里显示的地址 + `/swagger`，能看到接口文档页面即表示环境正常。

## 动手练习（递进）

每完成一步，用浏览器、`curl` 或 Swagger 验证；对应文档见上表。

| 步骤 | 练习内容 | 主要文档 |
|------|----------|----------|
| 1 | CRUD API + SQLite | `api-development.md`、`entity-framework-core.md` |
| 2 | 登录 JWT，保护写接口 | `authentication-jwt.md` |
| 3 | 分页 + FluentValidation | `validation-and-pagination.md` |
| 4 | IMemoryCache / Redis | `caching.md` |
| 5 | 下单后后台发邮件 | `background-services.md` |
| 6 | xUnit 集成测试覆盖 CRUD | `integration-testing.md` |

## 权威资源

- [ASP.NET Core 官方文档](https://learn.microsoft.com/aspnet/core/)
- [Entity Framework Core 文档](https://learn.microsoft.com/ef/core/)
- [.NET API 浏览器](https://learn.microsoft.com/dotnet/api/)
- [C# 语言文档](https://learn.microsoft.com/dotnet/csharp/)

## 延伸阅读

- 上级目录：[../README.md](../README.md)（后端通用主题）
- 跨目录：`database/`（SQL、索引）、`deployment/`（Docker 部署）、`testing/`（测试金字塔）
