# Entity Framework Core

> 关键词：EF Core、DbContext、Migration、LINQ、Repository | 前置知识：SQL 基础、表与行 | 难度：入门

## 概述

**Entity Framework Core**（EF Core，实体框架核心）是 .NET 的 **ORM**（Object-Relational Mapping，对象关系映射）：用 **C# 类和 LINQ** 操作数据库，大部分 CRUD 不必手写 SQL。

生活类比：数据库是**仓库货架**（表、行），C# 类是**货物标签**；EF Core 是**仓管员**——你说「拿名叫 Keyboard 的货」，它帮你翻译成 SQL 去查，并把结果装进 C# 对象。

在 ASP.NET Core 里，EF Core 通常放在**数据访问层**，被 Service 调用；API 层不直接操作 DbContext。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| DbContext | 一次「进仓库办事」的会话 | 跟踪实体变更、执行 `SaveChanges` |
| Entity（实体） | 一张表对应的 C# 类 | POCO 类，属性映射列 |
| DbSet\<T\> | 某张表的查询入口 | 如 `DbSet<Product> Products` |
| Migration（迁移） | 数据库结构变更的版本记录 | 代码描述 schema 变更，可生成 SQL |
| Change Tracking | EF 记住你改了哪些字段 | 只读查询用 `AsNoTracking()` 更省资源 |
| Fluent API | 在代码里配置主键、长度、索引 | 比 Data Annotations 更灵活 |

## 示例

### 安装包与迁移命令

```powershell
# 添加 SQL Server 提供程序（也可换 PostgreSQL、SQLite）
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design

# 根据模型生成迁移文件
dotnet ef migrations add InitialCreate

# 把迁移应用到数据库（建表）
dotnet ef database update
```

**逐步讲解：**

1. `Design` 包供 CLI 工具 `dotnet ef` 在设计时使用。
2. `migrations add` 在项目中生成迁移 C# 文件，记录「要建哪些表」。
3. `database update` 真正在数据库执行 DDL（建表、加列等）。

### DbContext 与实体

```csharp
// Data/AppDbContext.cs
public class AppDbContext : DbContext
{
    // 构造函数注入配置（连接哪个数据库）
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // Products 对应数据库里的 Products 表
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);                              // 主键
            e.Property(p => p.Name).HasMaxLength(200).IsRequired(); // 必填、最长 200
            e.HasIndex(p => p.Name);                          // 给 Name 建索引
        });
    }
}

// 实体类：一张表一行记录
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

// Program.cs 注册（默认 Scoped，与 HTTP 请求同生命周期）
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

**逐步讲解：**

1. `DbContextOptions` 由 DI 注入，里面包含连接字符串。
2. `OnModelCreating` 集中配置约束，避免数据库和代码不一致。
3. `AddDbContext` **不要**改成 Singleton，否则多线程/多请求会出问题。

### Repository 模式（可选但推荐）

```csharp
public interface IProductRepository
{
    Task<PagedResult<ProductDto>> ListAsync(int page, int size);
    Task<Product?> GetByIdAsync(int id);
    Task<Product> AddAsync(Product product);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;

    public ProductRepository(AppDbContext db) => _db = db;

    public async Task<PagedResult<ProductDto>> ListAsync(int page, int size)
    {
        var query = _db.Products.AsNoTracking().OrderBy(p => p.Id);
        var total = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * size)   // 分页：跳过前 (page-1)*size 条
            .Take(size)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price))  // 只查需要的列
            .ToListAsync();
        return new PagedResult<ProductDto>(items, total, page, size);
    }

    public Task<Product?> GetByIdAsync(int id)
        => _db.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id);

    public async Task<Product> AddAsync(Product product)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync();  // 真正写入数据库
        return product;
    }
}
```

**逐步讲解：**

1. `AsNoTracking()` 只读时不跟踪变更，列表查询更快。
2. `Skip/Take` 在数据库端分页，不要 `ToList()` 后再在内存里分页。
3. `Select` 投影成 DTO，减少数据传输，也避免把实体直接暴露给 API。

### 事务（多步操作要么全成功要么全回滚）

```csharp
public async Task TransferInventoryAsync(int fromId, int toId, int qty)
{
    await using var tx = await _db.Database.BeginTransactionAsync();
    try
    {
        // ... 改两条库存记录 ...
        await _db.SaveChangesAsync();
        await tx.CommitAsync();    // 全部成功才提交
    }
    catch
    {
        await tx.RollbackAsync();  // 任一步失败则撤销
        throw;
    }
}
```

Service 与 DbContext 同为 **Scoped** 时，一次 HTTP 请求共享同一 DbContext，适合「一个请求 = 一个事务单元」。

## 实践步骤

1. 在 `appsettings.json` 配置 `ConnectionStrings:Default`；开发可用 SQLite 或 Docker SQL Server
2. 定义实体与 `DbContext`，用 Fluent API 设长度、索引
3. `dotnet ef migrations add InitialCreate` → `dotnet ef database update`
4. 列表接口必须分页；用 `EXPLAIN` 或日志看是否全表扫描
5. 生产部署通过 CI 执行迁移或生成 SQL 脚本，不用 `EnsureCreated()`

## 常见误区

- ❌ DbContext 注册为 Singleton → ✅ 使用 `AddDbContext`，保持 Scoped
- ❌ 循环里逐条查询（N+1 问题）→ ✅ `Include` 预加载或一次 `Select` 联表
- ❌ API 返回带导航属性的实体 → ✅ DTO + `AsNoTracking()`
- ❌ 生产用 `EnsureCreated()` 代替 Migration → ✅ Migration 版本化管理表结构
- ❌ 异步方法里 `.Result` / `.Wait()` 阻塞 → ✅ 全链路 `async/await`
- ❌ 字符串字段不设最大长度 → ✅ 与数据库列一致，防止脏数据和过大索引

## 数据库提供程序

| 数据库 | NuGet 包 |
|--------|----------|
| SQL Server | `Microsoft.EntityFrameworkCore.SqlServer` |
| PostgreSQL | `Npgsql.EntityFrameworkCore.PostgreSQL` |
| SQLite | `Microsoft.EntityFrameworkCore.Sqlite` |
| MySQL | `Pomelo.EntityFrameworkCore.MySql` |

## 与其他领域的关联

- **API**：Service 调 Repository，见 `api-development.md`
- **DI**：`AddDbContext` 生命周期，见 `dependency-injection.md`
- **数据库理论**：索引、事务隔离，见 `database/` 目录
- **部署**：连接字符串用环境变量；迁移在 CI/CD 执行

## 参考资源

- [EF Core 文档](https://learn.microsoft.com/ef/core/)
- [DbContext 配置](https://learn.microsoft.com/ef/core/dbcontext-configuration/)
- [迁移概述](https://learn.microsoft.com/ef/core/managing-schemas/migrations/)
- [性能最佳实践](https://learn.microsoft.com/ef/core/performance/)

## 延伸阅读

- 同目录：`api-development.md`、`dependency-injection.md`、`caching.md`
- 跨目录：`database/` SQL 与索引优化
