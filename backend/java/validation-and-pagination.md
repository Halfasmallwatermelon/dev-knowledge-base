# Spring Boot 校验与分页

> 关键词：Bean Validation、@Valid、Pageable、Page | 前置知识：`api-development.md` | 难度：入门

## 概述

**校验**在数据进入 Service 前检查格式；**分页**避免一次查询返回全表。Spring Boot 集成 **Jakarta Bean Validation**（原 Hibernate Validator），分页常用 **Spring Data** 的 `Pageable` / `Page`。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| @Valid | 触发对参数的校验 | 配合 `@RequestBody`、record/class |
| @NotBlank 等 | 字段上的规则注解 | Jakarta Validation 标准 |
| BindingResult | 校验失败详情 | 非 REST 表单常用 |
| Pageable | 分页+排序参数对象 | `page`、`size`、`sort` |
| Page\<T\> | 一页数据 + 总数 + 总页数 | Repository 返回类型 |

## 示例

### DTO 校验

```java
public record CreateProductRequest(
    @NotBlank(message = "商品名称不能为空")
    @Size(max = 200, message = "名称最长 200 字")
    String name,

    @NotNull(message = "价格不能为空")
    @DecimalMin(value = "0.01", message = "价格必须大于 0")
    BigDecimal price
) {}
```

```java
@PostMapping
public ResponseEntity<ProductDto> create(@Valid @RequestBody CreateProductRequest req) {
    // 校验失败时在进入方法前被 GlobalExceptionHandler 拦截为 400
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(productService.create(req));
}
```

依赖（Spring Boot 3 通常已带）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**逐步讲解：**

1. 注解写在 **DTO** 上，不要写在 JPA Entity 上（API 契约 ≠ 表结构）。
2. `message` 会出现在 400 响应里，可国际化。
3. 复杂规则可写自定义 `@Constraint` 校验类。

### 方法级校验

```java
@Service
@Validated  // 启用方法参数校验
public class ProductService {

    public ProductDto updateName(
            @NotNull Long id,
            @NotBlank @Size(max = 200) String newName) {
        // ...
    }
}
```

### Spring Data 分页

```java
// Repository
public interface ProductJpaRepository extends JpaRepository<Product, Long> {
    Page<Product> findByNameContainingIgnoreCase(String keyword, Pageable pageable);
}

// Service
public Page<ProductDto> list(String keyword, Pageable pageable) {
    Page<Product> page = repository.findByNameContainingIgnoreCase(
        keyword == null ? "" : keyword,
        pageable);
    return page.map(ProductDto::from);
}

// Controller — GET /api/v1/products?page=0&size=10&sort=price,desc
@GetMapping
public Page<ProductDto> list(
        @RequestParam(required = false) String keyword,
        @PageableDefault(size = 20, sort = "id") Pageable pageable) {
    return productService.list(keyword, pageable);
}
```

**逐步讲解：**

1. Spring Data 的 **page 从 0 开始**（第 1 页是 `page=0`），文档里要写清楚。
2. `sort=price,desc` 多字段排序用 `sort=name,asc&sort=id,desc`。
3. `Page` JSON 含 `content`、`totalElements`、`totalPages` 等，前端可直接用。

```json
// GET /api/v1/products?page=0&size=10
{
  "content": [{ "id": 1, "name": "Mouse", "price": 99.00 }],
  "totalElements": 45,
  "totalPages": 5,
  "size": 10,
  "number": 0,
  "first": true,
  "last": false
}
```

### 限制每页最大条数

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        // 也可自定义 PageableHandlerMethodArgumentResolver 设置 maxPageSize
    }
}

// 简单做法：Service 里 clamp
int size = Math.min(pageable.getPageSize(), 100);
Pageable limited = PageRequest.of(pageable.getPageNumber(), size, pageable.getSort());
```

## 实践步骤

1. POST 空 `name`，确认 400 与 fieldErrors
2. 列表 `page=0&size=5` 与 `page=1&size=5` 对比 content
3. 文档注明 page 从 0 开始（或封装成从 1 开始的 DTO 参数）
4. `size` 上限 100，防恶意大页
5. 排序字段白名单，禁止用户随意指定未索引列

## 常见误区

- ❌ 只在前端校验 → ✅ 后端必须校验
- ❌ page 从 1 开始却直接用 Spring 默认 → ✅ 统一约定或做 -1 转换
- ❌ `findAll()` 再内存分页 → ✅ DB 层 `Pageable`
- ❌ 用户传 `sort=password` 泄露字段 → ✅ 白名单映射

## 与其他领域的关联

- **异常**：`MethodArgumentNotValidException`，见 `api-development.md`
- **JPA**：`JpaRepository` + `Pageable`，见 `spring-data-jpa.md`

## 参考资源

- [Validation](https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.validation)
- [分页与排序](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.details)

## 延伸阅读

- 同目录：`api-development.md`、`spring-data-jpa.md`
- 对照：[../csharp/validation-and-pagination.md](../csharp/validation-and-pagination.md)
