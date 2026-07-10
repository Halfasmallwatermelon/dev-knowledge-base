# Spring Data JPA

> 关键词：JPA、Entity、Repository、Flyway、@Transactional | 前置知识：SQL 基础、表与行 | 难度：入门

## 概述

**JPA**（Java Persistence API）是 Java 的 ORM 标准；**Spring Data JPA** 在之上提供 `JpaRepository`，按方法名或 `@Query` 自动生成 SQL。类似 .NET 的 **Entity Framework Core**。

生活类比：Entity 是**货物标签**，Repository 是**仓管接口**，JPA 帮你把标签和货架（表）对上号。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| Entity | 映射到表的 Java 类 | `@Entity` + `@Table` |
| Id | 主键 | `@Id`、`@GeneratedValue` |
| JpaRepository | 自带 CRUD 的接口 | `findById`、`save`、`delete` |
| 派生查询 | 方法名表达查询 | `findByNameContaining` |
| @Transactional | 事务边界 | 多步写操作要么全成功要么回滚 |
| Flyway/Liquibase | 数据库版本迁移 | 类似 EF Migrations |

## 示例

### 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/demo
    username: demo
    password: ${DB_PASSWORD:localdev}
  jpa:
    hibernate:
      ddl-auto: validate   # 生产：validate；开发可 none + Flyway
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  flyway:
    enabled: true
```

**逐步讲解：**

1. 生产用 **Flyway** 管理 schema，不用 `ddl-auto=update` 裸奔。
2. `validate` 只检查 Entity 与表是否一致，不自动改表。

### Entity

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 数据库自增
    private Long id;

    @Column(nullable = false, length = 200)
    private String name;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt = Instant.now();

    // 无参构造器 JPA 要求；getter/setter 或使用 Lombok @Getter @Setter
    protected Product() {}

    public Product(String name, BigDecimal price) {
        this.name = name;
        this.price = price;
    }

    // getters / setters ...
}
```

### Repository

```java
public interface ProductJpaRepository extends JpaRepository<Product, Long> {

    // 派生查询：SELECT ... WHERE LOWER(name) LIKE ...
    Page<Product> findByNameContainingIgnoreCase(String keyword, Pageable pageable);

    Optional<Product> findByName(String name);

    @Query("SELECT p FROM Product p WHERE p.price >= :minPrice")
    List<Product> findExpensive(@Param("minPrice") BigDecimal minPrice);
}
```

**逐步讲解：**

1. 继承 `JpaRepository<Product, Long>` 即有 `save`、`findById` 等。
2. 方法名 `findBy...` 由 Spring Data 解析成 SQL。
3. 复杂 SQL 用 `@Query` JPQL 或 `nativeQuery = true`。

### Service 与事务

```java
@Service
public class ProductService {

    private final ProductJpaRepository repository;

    public ProductService(ProductJpaRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)  // 只读查询，可优化连接
    public Optional<ProductDto> findById(Long id) {
        return repository.findById(id).map(ProductDto::from);
    }

    @Transactional  // 写操作默认读写事务
    public ProductDto create(CreateProductRequest req) {
        Product product = new Product(req.name(), req.price());
        return ProductDto.from(repository.save(product));
    }

    @Transactional
    public void transferStock(Long fromId, Long toId, int qty) {
        Product from = repository.findById(fromId).orElseThrow();
        Product to = repository.findById(toId).orElseThrow();
        // 业务扣减 ... 任一步异常则整方法回滚
        repository.save(from);
        repository.save(to);
    }
}
```

### Flyway 迁移脚本

```sql
-- src/main/resources/db/migration/V1__create_products.sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       NUMERIC(12, 2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_name ON products (name);
```

**逐步讲解：**

1. 文件名 `V1__描述.sql`，版本号递增。
2. 启动时 Flyway 执行未跑过的脚本。
3. CI/CD 部署时同样依赖 Flyway，保证各环境 schema 一致。

## 实践步骤

1. 本地 Docker 起 PostgreSQL，配置 datasource
2. 写 V1 迁移建表，Entity 与表对齐，`ddl-auto=validate`
3. 实现 CRUD Repository + Service
4. 列表用 `Pageable`，避免 `findAll()`
5. 打开 SQL 日志，确认无 N+1（必要时 `@EntityGraph` 或 join fetch）

## 常见误区

- ❌ 生产 `ddl-auto=create-drop` → ✅ Flyway + validate
- ❌ Controller 返回 Entity（懒加载报错）→ ✅ DTO + `@Transactional(readOnly=true)` 内转换
- ❌ 循环 `@OneToMany` JSON 序列化死循环 → ✅ DTO 或 `@JsonIgnore`
- ❌ 长事务里调外部 HTTP → ✅ 缩小 `@Transactional` 范围
- ❌ 在 Web 线程外访问懒加载关联 → ✅ 事务内加载或 DTO 投影

## 与其他领域的关联

- **API**：Controller 调 Service，见 `api-development.md`
- **DI**：Repository 是 Spring Bean，见 `dependency-injection.md`
- **数据库理论**：见 `database/` 目录

## 参考资源

- [Spring Data JPA 参考](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Flyway 文档](https://flywaydb.org/documentation/)
- [JPA 实体](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.jpa-and-spring-data)

## 延伸阅读

- 同目录：`api-development.md`、`validation-and-pagination.md`
- 对照：[../csharp/entity-framework-core.md](../csharp/entity-framework-core.md)
