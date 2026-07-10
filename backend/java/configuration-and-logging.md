# Spring Boot 配置与日志

> 关键词：application.yml、Profile、@ConfigurationProperties、SLF4J | 前置知识：`spring-boot-overview.md` | 难度：入门

## 概述

**配置**让同一套代码在开发、测试、生产用不同数据库地址、密钥等，而不改 Java 源码。**日志**记录运行信息，排错时 indispensable。

生活类比：配置是**分店参数表**（价格、地址）；日志是**值班记录本**。

Spring Boot 默认读 `src/main/resources/application.yml`（或 `.properties`），并支持 **Profile**（按环境加载不同文件）。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| application.yml | 主配置文件，YAML 格式 | 键值嵌套，比 properties 更易读 |
| Profile | 环境标签 dev/prod | `spring.profiles.active` |
| @ConfigurationProperties | 把配置前缀绑到 Java 类 | 类型安全，替代散落 `@Value` |
| @Value | 读单个配置项 | 适合简单值 |
| SLF4J | 日志门面 API | 实际实现常用 Logback |
| Logger | 写日志的对象 | `LoggerFactory.getLogger(Xxx.class)` |

## 示例

### application.yml

```yaml
server:
  port: 8080   # 监听端口

spring:
  application:
    name: demo-api
  profiles:
    active: dev   # 激活 dev 配置（可改为 prod）

  datasource:
    url: jdbc:postgresql://localhost:5432/demo
    username: demo
    password: ${DB_PASSWORD:localdev}  # 优先环境变量 DB_PASSWORD

jwt:
  issuer: https://api.example.com
  secret: ${JWT_SECRET:dev-only-secret-change-in-prod}
  expire-minutes: 60

logging:
  level:
    root: INFO
    com.example.demo: DEBUG        # 本包 DEBUG，框架少刷屏
    org.springframework.web: WARN
```

`application-dev.yml` 仅在 `active: dev` 时合并加载，可覆盖 datasource 为 H2。

**逐步讲解：**

1. `${DB_PASSWORD:localdev}`：有环境变量用环境变量，否则默认 `localdev`。
2. 敏感项生产环境**只**放环境变量或密钥服务，不写进 Git。
3. `logging.level` 控制包级别日志是否输出。

### @ConfigurationProperties 强类型配置

```java
package com.example.demo.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "jwt")  // 对应 yml 里 jwt.*
public record JwtProperties(
    String issuer,
    String secret,
    int expireMinutes
) {}
```

```java
// 在某个 @Configuration 类上启用
@Configuration
@EnableConfigurationProperties(JwtProperties.class)
public class AppConfig {
}
```

```java
@Service
public class TokenService {
    private final JwtProperties jwt;

    public TokenService(JwtProperties jwt) {
        this.jwt = jwt;  // 注入后 jwt.secret() 即可用
    }
}
```

**逐步讲解：**

1. `prefix = "jwt"` 绑定 `jwt.issuer` → `issuer()`（record 或 getter）。
2. 比多处 `@Value("${jwt.secret}")` 更清晰、易测。
3. Java 16+ 可用 `record` 简化不可变配置类。

### 使用 SLF4J 写日志

```java
package com.example.demo.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    // 常见写法：每个类一个 logger
    private static final Logger log = LoggerFactory.getLogger(ProductService.class);

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    public ProductDto findById(Long id) {
        log.info("查询商品 id={}", id);  // {} 占位，避免字符串拼接

        return repository.findById(id)
            .map(this::toDto)
            .orElseGet(() -> {
                log.warn("商品不存在 id={}", id);
                return null;
            });
    }
}
```

**逐步讲解：**

1. 用 `log.info/warn/error`，不要 `System.out.println`。
2. 占位符 `{}` 仅在对应级别启用时才格式化，性能更好。
3. **禁止**打印密码、完整 JWT、身份证号。

### 环境变量（生产）

```powershell
$env:SPRING_PROFILES_ACTIVE = "prod"
$env:DB_PASSWORD = "强密码"
$env:JWT_SECRET = "至少32字节随机串"
java -jar demo.jar
```

## 实践步骤

1. 创建 `application-dev.yml`，把 H2 配在 dev，PostgreSQL 配在 prod
2. 新增 `JwtProperties`，在 Service 里注入打印（勿提交真实 secret）
3. 给 Service 加 `log.info`，控制台观察级别变化
4. 用环境变量覆盖 `server.port`，确认端口改变
5. 生产 checklist：Profile=prod、密钥外置、日志落盘或接入 ELK

## 常见误区

- ❌ 生产密钥写在 yml 并提交 Git → ✅ 环境变量 / Vault
- ❌ 到处 `@Value` 散落 → ✅ 相关项用 `@ConfigurationProperties` 成组
- ❌ 日志打印巨大 JSON 或 PII → ✅ 脱敏、控制体积
- ❌ dev 与 prod 共用一个数据库 → ✅ Profile 隔离

## 与其他领域的关联

- **依赖注入**：配置类也是 Bean，见 `dependency-injection.md`
- **JWT**：`jwt.*` 配置，见 `authentication-jwt.md`
- **部署**：K8s ConfigMap/Secret，见 `deployment/`

## 参考资源

- [外部化配置](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
- [日志](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)

## 延伸阅读

- 同目录：`dependency-injection.md`、`authentication-jwt.md`
- 对照：[../csharp/configuration-and-logging.md](../csharp/configuration-and-logging.md)
