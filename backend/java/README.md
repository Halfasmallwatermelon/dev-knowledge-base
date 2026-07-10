# Java 后端

> 关键词：Java、JVM、Spring Boot | 前置知识：HTTP 基础、面向对象入门 | 难度：入门

## 概述

**Java** 是一种面向对象的编程语言，写好的代码在 **JVM**（Java Virtual Machine，Java 虚拟机）上运行——类似「同一份 bytecode 可以在不同操作系统跑」。企业里 Java 后端最常见组合是 **Spring Boot**：帮你快速搭建 REST API、连数据库、做鉴权。

餐厅类比（与 `backend/csharp/` 一致，便于对照学习）：

| 角色 | 对应技术 | 做什么 |
|------|----------|--------|
| 顾客 | 前端 / App | 发 HTTP 请求 |
| 前台 | `@RestController` | 接单、返回 JSON |
| 厨房 | `@Service` | 业务规则 |
| 仓库 | `@Repository` / JPA | 读写数据库 |
| 传菜通道 | Filter / Interceptor | 日志、鉴权、跨域 |

本目录以 **Spring Boot 3** 为主（使用 Jakarta EE 命名空间），零基础可按学习路线顺序阅读。

## 学习路线

### 第一阶段：跑起来与核心机制

| 顺序 | 文档 | 说明 |
|------|------|------|
| 1 | [spring-boot-overview.md](spring-boot-overview.md) | Spring Boot 是什么、项目结构、启动流程 |
| 2 | [configuration-and-logging.md](configuration-and-logging.md) | `application.yml`、Profile、SLF4J 日志 |
| 3 | [dependency-injection.md](dependency-injection.md) | IoC 容器、`@Autowired`、Bean 作用域 |
| 4 | [filters-and-interceptors.md](filters-and-interceptors.md) | Filter、Interceptor、CORS、请求链路 |

### 第二阶段：完整 API

| 顺序 | 文档 | 说明 |
|------|------|------|
| 5 | [api-development.md](api-development.md) | REST Controller、分层、统一异常 |
| 6 | [validation-and-pagination.md](validation-and-pagination.md) | Bean Validation、分页 `Pageable` |
| 7 | [spring-data-jpa.md](spring-data-jpa.md) | JPA 实体、Repository、迁移 Flyway |
| 8 | [authentication-jwt.md](authentication-jwt.md) | Spring Security + JWT |

### 第三阶段：进阶与质量

| 顺序 | 文档 | 说明 |
|------|------|------|
| 9 | [caching.md](caching.md) | `@Cacheable`、Redis |
| 10 | [async-and-scheduling.md](async-and-scheduling.md) | `@Async`、`@Scheduled`、消息队列入门 |
| 11 | [integration-testing.md](integration-testing.md) | `@SpringBootTest`、MockMvc |

## 环境准备

**第 1 步：安装 JDK**

安装 **JDK 17 或 21（LTS）**，从 [Adoptium](https://adoptium.net/) 或 [Oracle](https://www.oracle.com/java/technologies/downloads/) 下载。

```powershell
# 检查 Java 版本；应显示 17 或 21
java -version
```

**第 2 步：安装构建工具（二选一）**

- **Maven**：https://maven.apache.org/download.cgi  
- **Gradle**：https://gradle.org/install/

```powershell
mvn -version
# 或
gradle -version
```

**第 3 步：用 Spring Initializr 创建项目**

访问 https://start.spring.io/ ，选择：

- Project: Maven  
- Language: Java  
- Spring Boot: 3.x  
- Dependencies: **Spring Web**、**Spring Data JPA**、**H2 Database**（练习用）

下载解压后：

```powershell
cd demo
# Maven
./mvnw spring-boot:run
# Windows: mvnw.cmd spring-boot:run
```

浏览器打开 `http://localhost:8080`（若加了 springdoc，则为 `/swagger-ui.html`）。

## 动手练习（递进）

| 步骤 | 练习内容 | 主要文档 |
|------|----------|----------|
| 1 | CRUD API + H2/PostgreSQL | `api-development.md`、`spring-data-jpa.md` |
| 2 | JWT 登录，保护写接口 | `authentication-jwt.md` |
| 3 | 分页 + `@Valid` 校验 | `validation-and-pagination.md` |
| 4 | `@Cacheable` + Redis | `caching.md` |
| 5 | 下单后 `@Async` 发邮件 | `async-and-scheduling.md` |
| 6 | MockMvc 集成测试 | `integration-testing.md` |

## 权威资源

- [Spring Boot 官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Framework 文档](https://docs.spring.io/spring-framework/reference/)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Security](https://docs.spring.io/spring-security/reference/)

## 延伸阅读

- 对照学习：[../csharp/README.md](../csharp/README.md)（.NET 同主题笔记）
- 上级目录：[../README.md](../README.md)
- 跨目录：`database/`、`deployment/`、`testing/`
