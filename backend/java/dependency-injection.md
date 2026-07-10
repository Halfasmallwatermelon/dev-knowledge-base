# Spring 依赖注入

> 关键词：IoC、Bean、@Autowired、@Component、Scope | 前置知识：`spring-boot-overview.md`、接口 | 难度：入门

## 概述

**依赖注入**（Dependency Injection，DI）和 **IoC**（Inversion of Control，控制反转）在 Spring 里是一体两面：**对象的创建和依赖关系由 Spring 容器负责**，你的类只在构造函数或字段上「声明需要什么」。

生活类比：你是厨师（Service），需要砧板（Repository）。传统写法自己买砧板（`new Repository()`）；Spring 像**中央厨房后勤**，按单配好工具送到工位。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| Bean | 交给 Spring 管理的对象 | 容器中的组件实例 |
| 容器 | 存 Bean、解析依赖的工厂 | ApplicationContext |
| @Component | 标记「请把我注册成 Bean」 | 通用组件注解 |
| @Service / @Repository | 语义化 @Component | 业务层 / 持久层 |
| @Autowired | 让 Spring 注入依赖 | 可用于构造器、字段、setter |
| Scope 作用域 | Bean 活多久、几份 | singleton、prototype、request 等 |

### 常见作用域

| 作用域 | 说明 | 典型 |
|--------|------|------|
| singleton（默认） | 整个应用一个实例 | Service、Repository |
| prototype | 每次注入/获取新建 | 有状态非线程安全对象 |
| request | 一次 HTTP 请求一个 | Web 应用中的请求级 Bean |

## 示例

### 接口 + 实现 + 构造器注入（推荐）

```java
// repository/ProductRepository.java — 数据访问接口
public interface ProductRepository {
    Optional<Product> findById(Long id);
}

// repository/JpaProductRepository.java
@Repository  // = @Component，且会把持久化异常转译
public class JpaProductRepository implements ProductRepository {

    private final ProductJpaRepository jpa;  // Spring Data 自动生成

    public JpaProductRepository(ProductJpaRepository jpa) {
        this.jpa = jpa;
    }

    @Override
    public Optional<Product> findById(Long id) {
        return jpa.findById(id);
    }
}

// service/ProductService.java
@Service
public class ProductService {

    private final ProductRepository repository;

    // 构造器注入：Spring 4.3+ 单构造器时可省略 @Autowired
    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    public ProductDto getById(Long id) {
        return repository.findById(id)
            .map(ProductDto::from)
            .orElse(null);
    }
}

// controller/ProductController.java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> get(@PathVariable Long id) {
        ProductDto dto = productService.getById(id);
        return dto == null ? ResponseEntity.notFound().build() : ResponseEntity.ok(dto);
    }
}
```

**逐步讲解：**

1. `@Repository` / `@Service` / `@RestController` 都会被扫描为 Bean。
2. **构造器注入**便于单元测试（手动 `new Service(mockRepo)`）且依赖 `final` 不可变。
3. 避免字段 `@Autowired`（难测、隐藏依赖）——团队规范常禁止 field injection。

### @Configuration 手动声明 Bean

```java
@Configuration
public class AppConfig {

    @Bean  // 方法的返回值注册为 Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    @ConditionalOnProperty(name = "feature.email.enabled", havingValue = "true")
    public EmailSender emailSender() {
        return new SmtpEmailSender();
    }
}
```

**逐步讲解：**

1. 第三方类无法加 `@Component` 时，用 `@Bean` 方法注册。
2. `@ConditionalOnProperty` 按配置开关决定是否创建 Bean。

### 循环依赖问题

```text
A 依赖 B，B 又依赖 A → 启动失败（构造器注入时）
```

解决： redesign 抽第三组件、用 `@Lazy` 延迟其一、或事件驱动解耦——**不要**强行 singleton 互 new。

## 实践步骤

1. 把 `new XxxRepository()` 改成接口 + `@Repository` 实现
2. Controller 只保留构造器注入 Service
3. 写一个不依赖 Spring 的单元测试：`new ProductService(mockRepo)`
4. 用 `@Profile("dev")` 注册 Mock 实现，prod 用真实实现
5. 启动失败时读 Caused by：常见是「找不到 Bean」或循环依赖

## 常见误区

- ❌ 在 Bean 里 `new` 本应注入的依赖 → ✅ 构造器注入
- ❌ 到处 `@Autowired` 字段 → ✅ 构造器注入 + `final`
- ❌ 把 Entity 当 Singleton Bean 共享可变状态 → ✅ 无状态 Service，状态放数据库
- ❌ Service 直接依赖 HttpServletRequest → ✅ 参数下传或使用 `@RequestScope` DTO

## 与其他领域的关联

- **配置**：`@ConfigurationProperties` Bean，见 `configuration-and-logging.md`
- **数据层**：Repository Bean，见 `spring-data-jpa.md`
- **测试**：`@MockBean` 替换容器内 Bean，见 `integration-testing.md`

## 参考资源

- [IoC 容器](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Bean 作用域](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)

## 延伸阅读

- 同目录：`api-development.md`、`spring-data-jpa.md`
- 对照：[../csharp/dependency-injection.md](../csharp/dependency-injection.md)
