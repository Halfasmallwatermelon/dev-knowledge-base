# ASP.NET Core 集成测试

> 关键词：WebApplicationFactory、xUnit、TestServer、集成测试 | 前置知识：`api-development.md`、单元测试基础 | 难度：进阶

## 概述

**单元测试**（Unit Test）只测一个类、依赖用 Mock 假对象。**集成测试**（Integration Test）启动**接近真实的 API**，用 HTTP 客户端发请求，验证路由、中间件、DI、数据库一整条链路。

生活类比：单元测试像单独试**一把刀**是否锋利；集成测试像**完整做一道菜**——从接单到上菜全流程走一遍。

ASP.NET Core 提供 **WebApplicationFactory\<TProgram\>**，在测试里托管你的 API，无需手动 `dotnet run`。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| WebApplicationFactory | 测试专用的「迷你宿主」 | 启动 `Program` 定义的 API |
| TestServer | 内存里的 HTTP 服务器 | 不发真实网络包，速度快 |
| HttpClient | 测试里发 GET/POST 的客户端 | `factory.CreateClient()` |
| 测试数据库 | 与生产隔离的库 | SQLite 内存库或 Testcontainers |
| Fixture | 多个测试共享的启动/清理逻辑 | xUnit `IClassFixture<T>` |

## 示例

### 创建测试项目

```powershell
# 在解决方案根目录
dotnet new xunit -n MyApi.Tests -o MyApi.Tests
cd MyApi.Tests
dotnet add reference ../MyApi/MyApi.csproj
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

被测 API 项目的 `.csproj` 需暴露 `Program` 给测试项目：

```xml
<!-- MyApi.csproj 末尾添加 -->
<ItemGroup>
  <InternalsVisibleTo Include="MyApi.Tests" />
</ItemGroup>
```

若 `Program.cs` 使用顶级语句，测试项目需能引用入口——常见做法是在 API 项目添加：

```csharp
// Program.cs 最底部（仅用于测试可见性）
public partial class Program { }
```

### 基础集成测试

```csharp
// MyApi.Tests/ProductsApiTests.cs
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc.Testing;

public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetHealth_ReturnsOk()
    {
        var response = await _client.GetAsync("/health");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task CreateProduct_ThenGetById_ReturnsCreatedProduct()
    {
        // POST 新建
        var createResponse = await _client.PostAsJsonAsync("/api/v1/products", new
        {
            name = "Test Keyboard",
            price = 199.00m
        });
        Assert.Equal(HttpStatusCode.Created, createResponse.StatusCode);

        var created = await createResponse.Content.ReadFromJsonAsync<ProductDto>();
        Assert.NotNull(created);

        // GET 按 id 查
        var getResponse = await _client.GetAsync($"/api/v1/products/{created!.Id}");
        getResponse.EnsureSuccessStatusCode();
        var fetched = await getResponse.Content.ReadFromJsonAsync<ProductDto>();
        Assert.Equal("Test Keyboard", fetched!.Name);
    }
}

public record ProductDto(int Id, string Name, decimal Price);
```

**逐步讲解：**

1. `WebApplicationFactory<Program>` 启动完整 API 管道（中间件、DI 都在）。
2. `CreateClient()` 得到指向 TestServer 的 `HttpClient`。
3. `PostAsJsonAsync` / `ReadFromJsonAsync` 自动 JSON 序列化。
4. `IClassFixture` 让整个测试类共用一个 factory，启动更快。

### 自定义测试配置（替换数据库）

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 移除真实 DbContext 注册
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null) services.Remove(descriptor);

            // 改用 SQLite 内存库，每测独立、无需 SQL Server
            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlite("DataSource=:memory:"));
        });

        builder.UseEnvironment("Development");
    }
}
```

```csharp
public class ProductsApiTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public ProductsApiTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task ListProducts_EmptyDatabase_ReturnsEmptyList()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.OpenConnectionAsync();
        await db.Database.EnsureCreatedAsync();

        var response = await _client.GetAsync("/api/v1/products?page=1&size=10");
        response.EnsureSuccessStatusCode();
    }
}
```

**逐步讲解：**

1. `ConfigureWebHost` 在测试启动前替换 DI 注册。
2. SQLite `:memory:` 速度快，CI 无需装 SQL Server。
3. 每个测试前 `EnsureCreated` 或迁移，保证 schema 存在。

### 测试需 JWT 的接口

```csharp
[Fact]
public async Task AdminEndpoint_WithoutToken_Returns401()
{
    var response = await _client.GetAsync("/api/admin/users");
    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}

[Fact]
public async Task AdminEndpoint_WithValidToken_ReturnsOk()
{
    var token = await LoginAndGetTokenAsync("admin", "password");
    _client.DefaultRequestHeaders.Authorization =
        new AuthenticationHeaderValue("Bearer", token);

    var response = await _client.GetAsync("/api/admin/users");
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
}
```

## 实践步骤

1. 创建 xUnit 项目，引用 `Microsoft.AspNetCore.Mvc.Testing`
2. 为 `/health` 写第一个集成测试，跑 `dotnet test`
3. 用 `CustomWebApplicationFactory` 换 SQLite，测 CRUD 全流程
4. 测 400（非法 body）、404（不存在 id）、401（无 Token）
5. 接入 CI：每次 PR 自动 `dotnet test`

## 常见误区

- ❌ 集成测试连生产数据库 → ✅ 专用测试库或内存/SQLite
- ❌ 测试互相污染数据 → ✅ 每测独立库、或事务回滚、或按测试清理表
- ❌ 只测 200 不测状态码和 body → ✅ 断言 StatusCode + 关键字段
- ❌ 忘记 `public partial class Program` 导致工厂找不到入口 → ✅ 按上文暴露 Program
- ❌ 集成测试过多导致 CI 很慢 → ✅ 金字塔：大量单元 + 适量集成 + 少量 E2E

## 与其他领域的关联

- **API**：契约与状态码，见 `api-development.md`
- **JWT**：鉴权场景测试，见 `authentication-jwt.md`
- **测试通用理论**：见 `testing/` 目录测试金字塔

## 参考资源

- [集成测试官方文档](https://learn.microsoft.com/aspnet/core/test/integration-tests)
- [xUnit 文档](https://xunit.net/)
- [Testcontainers for .NET](https://dotnet.testcontainers.org/)（Docker 起真实 DB）

## 延伸阅读

- 同目录：`api-development.md`、`authentication-jwt.md`
- 跨目录：`testing/` 单元测试与 Mock
