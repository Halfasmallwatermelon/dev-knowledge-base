# ASP.NET Core 校验与分页

> 关键词：Validation、Data Annotations、FluentValidation、Pagination | 前置知识：`api-development.md` | 难度：入门

## 概述

**校验**（Validation）在数据进业务层之前检查「格式对不对、必填有没有填」——避免脏数据进数据库。**分页**（Pagination）把长列表拆成多页返回，避免一次查上万条拖垮数据库和前端。

生活类比：校验像快递**验货**（破损拒收）；分页像**翻书**（每次只看一页，而不是一次撕下全书）。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| Data Annotations | 在属性上贴「规则标签」 | `[Required]`、`[MaxLength]` 等特性 |
| FluentValidation | 用独立类写规则，更灵活 | `AbstractValidator<T>` |
| ModelState | 框架收集的校验结果 | 失败时自动返回 400 ProblemDetails |
| 分页参数 page/size | 第几页、每页几条 | 常见 Query：`?page=1&size=20` |
| PagedResult | 列表 + 总数 + 页码 的响应形状 | `{ items, total, page, size }` |

## 示例

### Data Annotations 校验

```csharp
// Models/CreateProductRequest.cs
public class CreateProductRequest
{
    [Required(ErrorMessage = "商品名称不能为空")]
    [MaxLength(200, ErrorMessage = "名称最长 200 字")]
    public string Name { get; set; } = "";

    [Range(0.01, 999999, ErrorMessage = "价格必须在 0.01～999999 之间")]
    public decimal Price { get; set; }
}

// Controller — [ApiController] 会自动校验，失败返回 400
[HttpPost]
public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductRequest req)
{
    // 若走到这里，req 已通过校验
    var created = await _svc.CreateAsync(req);
    return CreatedAtAction(nameof(Get), new { id = created.Id }, created);
}
```

**逐步讲解：**

1. 特性写在 DTO 上，不是数据库实体上（API 契约与存储分离）。
2. `[ApiController]` 会在 Action 执行前自动校验 `[FromBody]` 模型。
3. 失败响应是 ProblemDetails，字段错误在 `errors` 里。

### FluentValidation（复杂规则推荐）

```powershell
dotnet add package FluentValidation.AspNetCore
```

```csharp
// Validators/CreateProductRequestValidator.cs
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("商品名称不能为空")
            .MaximumLength(200);

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("价格必须大于 0");

        // 跨字段规则示例：高价商品名称不能过短
        RuleFor(x => x.Name)
            .MinimumLength(5)
            .When(x => x.Price > 10000);
    }
}

// Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductRequestValidator>();
```

**逐步讲解：**

1. 规则集中在 Validator 类，DTO 保持干净。
2. `When` 可写「仅当某条件时」才生效的规则。
3. Minimal API 可配合 `AddEndpointsApiExplorer` 与过滤器；Controller 与 Minimal API 均支持自动校验集成。

### 分页请求与响应模型

```csharp
// 请求：Query 参数
public record PagedQuery(int Page = 1, int Size = 20)
{
    // 防止有人传 page=0 或 size=10000
    public int NormalizedPage => Page < 1 ? 1 : Page;
    public int NormalizedSize => Size switch
    {
        < 1 => 20,
        > 100 => 100,   // 单页最多 100 条
        _ => Size
    };
}

// 响应：统一分页结构
public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int Total,
    int Page,
    int Size)
{
    public int TotalPages => (int)Math.Ceiling(Total / (double)Size);
    public bool HasNext => Page < TotalPages;
}
```

### API 端点示例

```csharp
// GET /api/v1/products?page=2&size=10
products.MapGet("/", async (IProductService svc, [AsParameters] PagedQuery query) =>
{
    var result = await svc.ListAsync(query.NormalizedPage, query.NormalizedSize);
    return Results.Ok(result);
});
```

```json
// 响应 200
{
  "items": [
    { "id": 11, "name": "Mouse", "price": 99.00 }
  ],
  "total": 45,
  "page": 2,
  "size": 10,
  "totalPages": 5,
  "hasNext": true
}
```

**逐步讲解：**

1. `[AsParameters]` 把 Query 字符串绑到 `PagedQuery`  record 的属性。
2. Service/Repository 里用 `Skip((page-1)*size).Take(size)`，见 `entity-framework-core.md`。
3. 返回 `total` 让前端能算总页数、显示「共 45 条」。

### 排序与过滤（扩展）

```csharp
// GET /api/v1/products?page=1&size=20&sort=price&order=desc&keyword=keyboard
public record ProductListQuery(
    int Page = 1,
    int Size = 20,
    string? Sort = "id",      // 允许排序的字段白名单在 Service 里校验
    string Order = "asc",
    string? Keyword = null);
```

**安全提示：** `Sort` 字段必须**白名单**映射到列名，禁止直接把用户输入拼进 SQL（防注入）。

## 实践步骤

1. 给 `CreateProductRequest` 加 `[Required]`，用 Swagger 发空 body，确认 400
2. 引入 FluentValidation，写一条 `When` 条件规则
3. 列表接口加分页，传 `page=1&size=5` 与 `page=2&size=5` 对比结果
4. 限制 `size` 上限（如 100），防止恶意大页拖库
5. 文档里写清 Query 参数含义，与 OpenAPI 描述一致

## 常见误区

- ❌ 只校验前端，后端不校验 → ✅ **后端必须校验**，前端可被绕过
- ❌ 分页在内存里 `list.Skip.Take`（已 ToList 全表）→ ✅ 在 SQL/EF 层分页
- ❌ `size` 无上限，用户传 `size=1000000` → ✅ 服务端 clamp 最大值
- ❌ 校验规则写在 Controller 里一堆 if → ✅ Annotations 或 FluentValidation 集中管理
- ❌ 分页从 0 开始却文档写从 1 开始 → ✅ 团队统一，并在 API 文档写明

## 与其他领域的关联

- **API 设计**：ProblemDetails 400，见 `api-development.md`
- **数据库**：EF `Skip/Take` + `CountAsync`，见 `entity-framework-core.md`
- **前端**：表格组件消费 `items/total/page`，见 `frontend/` 目录

## 参考资源

- [模型验证](https://learn.microsoft.com/aspnet/core/mvc/models/validation)
- [FluentValidation 文档](https://docs.fluentvalidation.net/)
- [Minimal API 参数绑定](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/parameter-binding)

## 延伸阅读

- 同目录：`api-development.md`、`entity-framework-core.md`
- 跨目录：`database/` 索引与慢查询优化
