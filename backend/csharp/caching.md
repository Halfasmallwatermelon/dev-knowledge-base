# ASP.NET Core 缓存

> 关键词：IMemoryCache、IDistributedCache、Redis、Cache-Aside | 前置知识：`api-development.md`、`dependency-injection.md` | 难度：进阶

## 概述

**缓存**（Caching）把经常访问、变化不频繁的数据放在**更快的存储**里（内存或 Redis），下次请求直接读缓存，少打数据库。

生活类比：餐厅**今日特价**写在门口小黑板（缓存），客人不用每次都进厨房问厨师（数据库）；特价变了再擦掉重写（失效/更新）。

ASP.NET Core 提供：
- **IMemoryCache**：单机内存，适合开发或单实例部署
- **IDistributedCache**：多实例共享，常用 **Redis** 实现

## 核心概念

| 概念 | 通俗解释 | 正式说明 |
|------|----------|----------|
| Cache-Aside | 先查缓存，没有再查库并回填 | 应用代码控制的读穿模式 |
| TTL（Time To Live） | 缓存活多久自动过期 | `AbsoluteExpirationRelativeToNow` |
| 缓存键 Key | 唯一标识一条缓存 | 如 `product:42`、`products:list:1:20` |
| 失效（Eviction） | 数据变了让缓存作废 | 更新/删除时 `Remove(key)` |
| 穿透 | 查不存在的数据，缓存也没有，每次都打库 | 可缓存空值或布隆过滤器 |
| 雪崩 | 大量 key 同时过期，数据库压力骤增 | 过期时间加随机抖动 |

## 示例

### 内存缓存 IMemoryCache

```csharp
// Program.cs 注册
builder.Services.AddMemoryCache();

public class ProductService : IProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;

    public ProductService(IMemoryCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        var cacheKey = $"product:{id}";

        // GetOrCreateAsync：没有则执行 factory 查库并写入缓存
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);  // 5 分钟过期
            entry.Size = 1;  // 若配置了 SizeLimit，需设置项大小

            var product = await _repo.GetByIdAsync(id);
            return product is null ? null : MapToDto(product);
        });
    }

    public async Task UpdateAsync(int id, UpdateProductRequest req)
    {
        await _repo.UpdateAsync(id, req);
        _cache.Remove($"product:{id}");  // 写后删缓存，下次读会刷新
    }
}
```

**逐步讲解：**

1. `AddMemoryCache()` 注册后，构造函数注入 `IMemoryCache`。
2. 键名建议有命名空间前缀 `product:`，避免不同业务冲突。
3. 更新/删除数据时**主动 Remove**，否则用户看到旧数据。
4. 内存缓存只在**当前进程**有效，多实例部署各有一份，可能不一致。

### Redis 分布式缓存

```powershell
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    // options.InstanceName = "MyApi:";  // 可选：键前缀，多应用共用 Redis 时区分
});

// appsettings.json
// "ConnectionStrings": { "Redis": "localhost:6379" }
```

```csharp
public class ProductService : IProductService
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repo;

    public ProductService(IDistributedCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        var cacheKey = $"product:{id}";
        var bytes = await _cache.GetAsync(cacheKey);

        if (bytes is not null)
        {
            // 命中缓存：反序列化 JSON
            return JsonSerializer.Deserialize<ProductDto>(bytes);
        }

        // 未命中：查库
        var product = await _repo.GetByIdAsync(id);
        if (product is null) return null;

        var dto = MapToDto(product);
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
        };
        await _cache.SetAsync(cacheKey, JsonSerializer.SerializeToUtf8Bytes(dto), options);
        return dto;
    }
}
```

**逐步讲解：**

1. `IDistributedCache` 存的是 `byte[]`，常用 JSON 序列化。
2. 多台 API 服务器连同一 Redis，缓存**共享**。
3. Docker 本地试 Redis：`docker run -d -p 6379:6379 redis`。

### HTTP 响应缓存（只读 GET）

对变化少、可公开的 GET 接口，可加响应缓存头（与数据缓存不同层）：

```csharp
builder.Services.AddResponseCaching();
app.UseResponseCaching();

app.MapGet("/api/v1/categories", async (ICategoryService svc) => Results.Ok(await svc.ListAsync()))
   .CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(10)));
```

适用于分类列表等**全员相同**且更新不频的数据；**不要**对用户私有数据盲目缓存。

## 何时用、何时不用

| 适合缓存 | 不适合缓存 |
|----------|------------|
| 读多写少（商品详情、配置） | 强一致金融余额（或极短 TTL + 失效策略） |
| 计算昂贵（报表、聚合） | 每次必须最新的库存扣减 |
| 可接受短暂旧数据 | 用户个人敏感数据未隔离键名 |

## 实践步骤

1. 给 `GetById` 加 `IMemoryCache`，对比加缓存前后数据库查询次数（EF 日志或 SQL Profiler）
2. 更新商品后调用 `Remove`，确认再 GET 拿到新数据
3. 本地 Docker 起 Redis，改用 `IDistributedCache`
4. 列表接口谨慎缓存：键要含 `page/size/过滤条件`，或只缓存热门 id
5. 压测观察：缓存命中率高时 P99 延迟应明显下降

## 常见误区

- ❌ 只缓存不失效，数据永远旧 → ✅ 写操作后 Remove 或设合理 TTL
- ❌ 缓存键不含参数，所有用户共用一个 key → ✅ 含 id、用户 id 等维度
- ❌ 把巨大列表整表塞进内存 → ✅ 只缓存热点或分页结果
- ❌ 多实例仍只用 IMemoryCache → ✅ 生产多副本用 Redis 等分布式缓存
- ❌ 缓存异常导致整个请求 500 → ✅ try/catch 缓存层，失败则降级查库

## 与其他领域的关联

- **数据库**：减轻读压力，见 `entity-framework-core.md`、`database/` 索引
- **部署**：Redis 作为独立服务，见 `deployment/` Docker Compose
- **API**：GET 与 Cache-Aside 配合，见 `api-development.md`

## 参考资源

- [ASP.NET Core 缓存概述](https://learn.microsoft.com/aspnet/core/performance/caching/overview)
- [IMemoryCache](https://learn.microsoft.com/aspnet/core/performance/caching/memory)
- [分布式 Redis 缓存](https://learn.microsoft.com/aspnet/core/performance/caching/distributed)
- [输出缓存](https://learn.microsoft.com/aspnet/core/performance/caching/output)

## 延伸阅读

- 同目录：`api-development.md`、`entity-framework-core.md`
- 跨目录：`database/` Redis 数据结构
