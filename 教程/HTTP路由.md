# HTTP 路由

你可以创建自定义的 HTTP 接口，让游戏客户端或其他程序与你的模组通信。

**源文件：** [10CustomRoute/](../10CustomRoute/)

## 应用场景

- 为客户端模组提供数据接口
- 创建 Web 管理面板 API
- 与外部工具集成

## 基本结构

HTTP 路由由两部分组成：
1. **路由定义**：定义 URL 路径和处理方法
2. **回调处理**：实际处理请求的逻辑

## 创建路由

### 1. 定义路由类

```csharp
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.Routers;
using SPTarkov.Server.Core.Models.Eft.Common.Tables;

[Injectable]
public class MyStaticRouter(
    JsonUtil jsonUtil,
    MyRouteCallback callback
) : StaticRouter(jsonUtil, [
    // 定义路由
    new RouteAction<MyRequestData>(
        "/my-mod/get-data",  // URL 路径
        async (url, info, sessionId, output) => 
            await callback.HandleGetData(url, info, sessionId)
    )
])
{ }
```

### 2. 定义回调类

```csharp
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.Models.Utils;
using SPTarkov.Server.Core.Models.Eft.Common.Tables;

[Injectable]
public class MyRouteCallback(
    ISptLogger<MyRouteCallback> logger,
    HttpResponseUtil httpResponseUtil
)
{
    public ValueTask<string> HandleGetData(
        string url, 
        MyRequestData info, 
        MongoId sessionId
    )
    {
        logger.Info($"收到请求，玩家ID: {sessionId}");
        
        // 返回 JSON 响应
        return new ValueTask<string>(
            httpResponseUtil.GetBody(new { message = "Hello!", data = info.SomeField })
        );
    }
}
```

### 3. 定义请求数据模型

```csharp
using SPTarkov.Server.Core.Models.Eft.Common.Tables;

// 自定义请求数据结构
public record MyRequestData : IRequestData
{
    public string SomeField { get; init; }
    public int SomeNumber { get; init; }
}
```

## 完整示例

```csharp
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.DI;
using SPTarkov.Server.Core.Models.Spt.Mod;
using SPTarkov.Server.Core.Models.Utils;
using SPTarkov.Server.Core.Routers;
using SPTarkov.Server.Core.Models.Eft.Common.Tables;
using SPTarkov.Server.Core.Extensions;

namespace MyMod;

public record ModMetadata : AbstractModMetadata
{
    public override string ModGuid { get; init; } = "com.example.httproute";
    public override string Name { get; init; } = "HTTP路由示例";
    public override string Author { get; init; } = "开发者";
    public override SemanticVersioning.Version Version { get; init; } = new("1.0.0");
    public override SemanticVersioning.Range SptVersion { get; init; } = new("~4.0.0");
    public override string License { get; init; } = "MIT";
}

// 请求数据模型
public record PlayerQueryRequest : IRequestData
{
    public string PlayerName { get; init; }
}

// 路由定义
[Injectable]
public class CustomStaticRouter(
    JsonUtil jsonUtil,
    CustomRouteCallback callback
) : StaticRouter(jsonUtil, [
    // 带参数的路由
    new RouteAction<PlayerQueryRequest>(
        "/my-mod/query-player",
        async (url, info, sessionId, output) => 
            await callback.QueryPlayer(url, info, sessionId)
    ),
    
    // 无参数的路由
    new RouteAction<EmptyRequestData>(
        "/my-mod/status",
        async (url, info, sessionId, output) => 
            await callback.GetStatus(url, info, sessionId)
    )
])
{ }

// 回调处理
[Injectable]
public class CustomRouteCallback(
    ISptLogger<CustomRouteCallback> logger,
    HttpResponseUtil httpResponseUtil
)
{
    public ValueTask<string> QueryPlayer(
        string url, 
        PlayerQueryRequest info, 
        MongoId sessionId
    )
    {
        logger.Info($"查询玩家: {info.PlayerName}");
        
        var result = new
        {
            success = true,
            playerName = info.PlayerName,
            level = 42,
            pmcSide = "USEC"
        };
        
        return new ValueTask<string>(httpResponseUtil.GetBody(result));
    }
    
    public ValueTask<string> GetStatus(
        string url, 
        EmptyRequestData info, 
        MongoId sessionId
    )
    {
        return new ValueTask<string>(
            httpResponseUtil.GetBody(new { status = "online", version = "1.0.0" })
        );
    }
}
```

## 响应工具方法

```csharp
// 返回 JSON 数据
httpResponseUtil.GetBody(new { key = "value" });

// 返回空响应
httpResponseUtil.NullResponse();

// 返回错误
httpResponseUtil.GetBody(new { error = "出错了" });

// 返回未找到
httpResponseUtil.GetBody(null, 404);
```

## 小贴士

1. **路由分离**：路由定义和回调处理分开，代码更清晰
2. **使用 EmptyRequestData**：不需要参数的路由使用这个类型
3. **sessionId**：可以用来识别发起请求的玩家
4. **日志记录**：在回调中记录日志，方便调试
5. **路径命名**：URL 路径建议用模组名作为前缀，避免冲突
