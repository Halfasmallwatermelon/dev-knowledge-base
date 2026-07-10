# ASP.NET Core 配置与日志

> 关键词：Configuration、appsettings、ILogger、Serilog | 前置知识：`aspnet-core-overview.md` | 难度：入门

## 概述

**配置**（Configuration）解决「同一份代码，开发/测试/生产用不同设置」——数据库地址、JWT 密钥、功能开关等不写死在代码里，而是从配置文件或环境变量读取。

**日志**（Logging）记录程序运行过程：谁访问了哪个接口、哪里报错、耗时多少。排查线上问题时，日志往往是第一线索。

生活类比：配置像餐厅的**菜单价目表**（不同分店不同价格）；日志像**监控录像**（事后能回放发生了什么）。

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| appsettings.json | 项目里的默认「设置文件」 | JSON 格式配置源，键值嵌套 |
| 环境（Environment） | Development / Production 等标签 | `IHostEnvironment.EnvironmentName` |
| 配置优先级 | 后读到的覆盖先读到的 | 默认：json → 环境变量 → 命令行 |
| IConfiguration | 读配置的通用接口 | `configuration["Key"]` 或 `GetSection` |
| Options 模式 | 把一节配置绑成 C# 类 | `IOptions<T>`，见 `dependency-injection.md` |
| ILogger | 写日志的接口 | 按级别 Debug/Info/Warning/Error 输出 |

## 示例

### appsettings.json 结构

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyDb;Trusted_Connection=True;"
  },
  "Jwt": {
    "Issuer": "https://api.example.com",
    "Key": "开发环境密钥-生产请用环境变量",
    "ExpireMinutes": 60
  },
  "FeatureFlags": {
    "EnableNewCheckout": false
  }
}
```

`appsettings.Development.json` 只在开发环境加载，可覆盖上面的值（如更详细的日志、本地数据库）。

**逐步讲解：**

1. `Logging:LogLevel` 控制哪些日志会输出；`Microsoft.AspNetCore` 设为 Warning 可减少框架刷屏。
2. `ConnectionStrings` 是约定俗成的节名，EF Core 常用 `GetConnectionString("Default")`。
3. 自定义节如 `Jwt`、`FeatureFlags` 可配合 Options 模式绑定。

### 读取配置

```csharp
var builder = WebApplication.CreateBuilder(args);

// 方式 1：直接读字符串（简单键）
var envName = builder.Environment.EnvironmentName;

// 方式 2：GetSection 读一整节
var jwtSection = builder.Configuration.GetSection("Jwt");
var issuer = jwtSection["Issuer"];

// 方式 3：强类型绑定（推荐）
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));

// 方式 4：连接字符串快捷方法
var conn = builder.Configuration.GetConnectionString("Default");
```

**逐步讲解：**

1. `CreateBuilder(args)` 已自动加载 `appsettings.json`、环境专用 json、环境变量。
2. 键名用 `:` 表示层级，环境变量里常写成 `Jwt__Key`（双下划线）。
3. 强类型绑定避免魔法字符串，改配置只需改类属性名与 JSON 对应。

### 环境变量覆盖（生产常用）

```powershell
# Windows PowerShell — 设置 JWT 密钥，不写入仓库
$env:Jwt__Key = "生产环境32字符以上随机密钥"
$env:ASPNETCORE_ENVIRONMENT = "Production"
dotnet run
```

Linux / Docker 同理：`Jwt__Key=xxx`、`ConnectionStrings__Default=xxx`。

### 使用 ILogger 写日志

```csharp
public class ProductService : IProductService
{
    private readonly ILogger<ProductService> _logger;
    private readonly IProductRepository _repo;

    public ProductService(ILogger<ProductService> logger, IProductRepository repo)
    {
        _logger = logger;
        _repo = repo;
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        _logger.LogInformation("查询商品 {ProductId}", id);  // 结构化日志，{ProductId} 可被检索

        var product = await _repo.GetByIdAsync(id);
        if (product is null)
        {
            _logger.LogWarning("商品不存在 {ProductId}", id);
            return null;
        }

        return product;
    }
}
```

**逐步讲解：**

1. `ILogger<T>` 由 DI 自动注入，**不要** `new Logger()`。
2. 用占位符 `{ProductId}` 而不是字符串拼接，便于日志系统索引。
3. 级别：`Debug` 开发细节 → `Information` 正常流程 → `Warning` 可恢复异常 → `Error` 需关注错误。

### 日志级别对照

| 级别 | 何时使用 |
|------|----------|
| Debug | 开发调试，生产通常关闭 |
| Information | 请求处理、业务关键步骤 |
| Warning | 预期内的问题（如资源不存在） |
| Error | 未捕获异常、外部服务失败 |
| Critical | 系统级故障，需立即处理 |

### 可选：Serilog 输出到文件

```powershell
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.File
```

```csharp
using Serilog;

// 在 CreateBuilder 之前配置
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("logs/app-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();  // 替换默认日志管道
```

## 实践步骤

1. 打开 `appsettings.json` 和 `appsettings.Development.json`，对比差异
2. 新增 `FeatureFlags:EnableNewCheckout`，在代码里用 `IOptions<FeatureFlags>` 读开关
3. 在 Service 里加 `LogInformation` / `LogWarning`，运行后看控制台输出
4. 用环境变量覆盖 `Jwt:Key`，确认程序读的是新值而非 json 里的旧值
5. 生产部署：密钥、连接字符串**只**放环境变量或密钥管理服务，不提交 Git

## 常见误区

- ❌ 把生产密钥、数据库密码写在 `appsettings.json` 并提交仓库 → ✅ 敏感项用环境变量或 Key Vault
- ❌ 日志里打印密码、Token、身份证号 → ✅ 脱敏或只打 id
- ❌ 到处 `Console.WriteLine` → ✅ 统一用 `ILogger`
- ❌ 生产仍用 `Development` 环境名 → ✅ 部署时设 `ASPNETCORE_ENVIRONMENT=Production`
- ❌ 配置改完不重启就认为生效 → ✅ 默认配置启动时加载；热更新需 `IOptionsMonitor` 等

## 与其他领域的关联

- **依赖注入**：`Configure<T>` 与 `IOptions<T>`，见 `dependency-injection.md`
- **JWT**：`Jwt` 配置节，见 `authentication-jwt.md`
- **EF Core**：`ConnectionStrings:Default`，见 `entity-framework-core.md`
- **部署**：K8s ConfigMap/Secret、Docker `-e`，见 `deployment/` 目录

## 参考资源

- [ASP.NET Core 配置](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
- [日志官方文档](https://learn.microsoft.com/aspnet/core/fundamentals/logging/)
- [Serilog](https://github.com/serilog/serilog-aspnetcore)

## 延伸阅读

- 同目录：`dependency-injection.md`、`authentication-jwt.md`
- 跨目录：`deployment/` 环境变量与密钥管理
