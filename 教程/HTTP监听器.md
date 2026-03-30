# HTTP 监听器

与静态路由不同，HTTP 监听器可以动态处理请求，适合需要更灵活控制的场景。

**源文件：** [15HttpListenerExample/HttpListenerExample.cs](../15HttpListenerExample/HttpListenerExample.cs)

## 与静态路由的区别

| 特性 | 静态路由 (StaticRouter) | HTTP 监听器 (IHttpListener) |
|------|------------------------|---------------------------|
| 路由定义 | 预定义路径列表 | 动态判断 |
| 灵活性 | 较低 | 较高 |
| 适用场景 | 固定 API 端点 | 动态 URL、特殊处理 |

## 实现 IHttpListener

### 基本结构

```csharp
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.Models.Common;
using SPTarkov.Server.Core.Servers;

[Injectable(TypePriority = 0)]
public class MyHttpListener : IHttpListener
{
    // 判断是否能处理这个请求
    public bool CanHandle(MongoId sessionId, HttpContext context)
    {
        // 返回 true 表示这个监听器处理该请求
        return context.Request.Path.Value!.Contains("/my-mod/");
    }
    
    // 处理请求
    public async Task Handle(MongoId sessionId, HttpContext context)
    {
        context.Response.StatusCode = 200;
        await context.Response.Body.WriteAsync(
            Encoding.UTF8.GetBytes("Hello from my mod!"));
        await context.Response.StartAsync();
        await context.Response.CompleteAsync();
    }
}
```

## 完整示例

```csharp
using System.Text;
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.Models.Common;
using SPTarkov.Server.Core.Models.Spt.Mod;
using SPTarkov.Server.Core.Servers;
using Microsoft.AspNetCore.Http;

namespace MyMod;

public record ModMetadata : AbstractModMetadata
{
    public override string ModGuid { get; init; } = "com.example.httplistener";
    public override string Name { get; init; } = "HTTP监听器示例";
    public override string Author { get; init; } = "开发者";
    public override SemanticVersioning.Version Version { get; init; } = new("1.0.0");
    public override SemanticVersioning.Range SptVersion { get; init; } = new("~4.0.0");
    public override string License { get; init; } = "MIT";
}

[Injectable(TypePriority = 0)]
public class CustomHttpListener : IHttpListener
{
    public bool CanHandle(MongoId sessionId, HttpContext context)
    {
        // 只处理 GET 请求且路径包含 /custom-url
        return context.Request.Method == "GET" 
            && context.Request.Path.Value!.Contains("/custom-url");
    }

    public async Task Handle(MongoId sessionId, HttpContext context)
    {
        // 获取查询参数
        var query = context.Request.Query;
        var paramName = query.ContainsKey("name") ? query["name"].ToString() : "Unknown";
        
        // 构建响应
        var responseText = $"Hello, {paramName}! Your session: {sessionId}";
        var responseBytes = Encoding.UTF8.GetBytes(responseText);
        
        // 发送响应
        context.Response.StatusCode = 200;
        context.Response.ContentType = "text/plain; charset=utf-8";
        await context.Response.Body.WriteAsync(responseBytes);
        await context.Response.StartAsync();
        await context.Response.CompleteAsync();
    }
}
```

## 处理不同请求类型

### GET 请求

```csharp
if (context.Request.Method == "GET")
{
    var path = context.Request.Path.Value;
    var query = context.Request.Query;
    
    // 处理 GET 请求...
}
```

### POST 请求

```csharp
if (context.Request.Method == "POST")
{
    // 读取请求体
    using var reader = new StreamReader(context.Request.Body);
    var body = await reader.ReadToEndAsync();
    
    // 解析 JSON
    var data = JsonSerializer.Deserialize<MyRequest>(body);
    
    // 处理 POST 请求...
}
```

### 返回 JSON

```csharp
using System.Text.Json;

public async Task Handle(MongoId sessionId, HttpContext context)
{
    var result = new { success = true, message = "操作成功" };
    var json = JsonSerializer.Serialize(result);
    
    context.Response.StatusCode = 200;
    context.Response.ContentType = "application/json";
    await context.Response.Body.WriteAsync(Encoding.UTF8.GetBytes(json));
    await context.Response.StartAsync();
    await context.Response.CompleteAsync();
}
```

## 多监听器协作

多个监听器可以按优先级顺序检查：

```csharp
// 高优先级监听器
[Injectable(TypePriority = 100)]
public class HighPriorityListener : IHttpListener { }

// 低优先级监听器
[Injectable(TypePriority = 10)]
public class LowPriorityListener : IHttpListener { }
```

优先级高的先检查，如果 `CanHandle` 返回 `true`，则不再检查其他监听器。

## 小贴士

1. **CanHandle 要快**：这个方法会被频繁调用，不要做耗时操作
2. **路径匹配要精确**：避免误处理其他模组的请求
3. **正确关闭响应**：使用 `CompleteAsync()` 确保响应完整发送
4. **错误处理**：用 try-catch 处理可能的异常
5. **日志记录**：记录请求信息便于调试
