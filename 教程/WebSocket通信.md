# WebSocket 通信

WebSocket 提供了实时双向通信能力，适合需要持续数据同步的场景。

**源文件：** [24Websocket/](../24Websocket/)

## 应用场景

- 实时数据推送
- 游戏状态同步
- 外部工具集成

## 实现 WebSocket 连接处理

### 连接处理器

```csharp
using System.Text;
using Microsoft.AspNetCore.Http;
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.Models.Common;
using SPTarkov.Server.Core.Servers;
using SPTarkov.Server.Core.Models.Utils;

[Injectable(InjectionType = InjectionType.Singleton)]
public class MyWebSocketHandler(
    ISptLogger<MyWebSocketHandler> logger
) : IWebSocketConnectionHandler
{
    // WebSocket 端点 URL
    public string GetHookUrl() => "/my-mod/socket/";
    
    // 连接标识
    public string GetSocketId() => "My Custom WebSocket";

    // 客户端连接时
    public Task OnConnection(WebSocket ws, HttpContext context, string sessionIdContext)
    {
        logger.Info($"WebSocket 已连接! Session: {sessionIdContext}");
        return Task.CompletedTask;
    }

    // 收到消息时
    public async Task OnMessage(byte[] rawData, WebSocketMessageType messageType, 
        WebSocket ws, HttpContext context)
    {
        var message = Encoding.UTF8.GetString(rawData);
        logger.Info($"收到消息: {message}");
        
        // 回复消息
        if (message == "ping")
        {
            await ws.SendAsync(
                Encoding.UTF8.GetBytes("pong"),
                WebSocketMessageType.Text,
                true,
                CancellationToken.None
            );
        }
    }

    // 连接关闭时
    public Task OnClose(WebSocket ws, HttpContext context, string sessionIdContext)
    {
        logger.Info("WebSocket 已断开");
        return Task.CompletedTask;
    }
}
```

### 消息处理器

处理 SPT 内部的 WebSocket 消息：

```csharp
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.Models.Common;
using SPTarkov.Server.Core.Servers;
using SPTarkov.Server.Core.Models.Utils;

[Injectable]
public class MySptWebSocketHandler(
    ISptLogger<MySptWebSocketHandler> logger
) : ISptWebSocketMessageHandler
{
    public Task OnSptMessage(string sessionID, WebSocket client, byte[] rawData)
    {
        var message = Encoding.UTF8.GetString(rawData);
        logger.Info($"SPT WebSocket 消息 [{sessionID}]: {message}");
        
        // 处理消息...
        
        return Task.CompletedTask;
    }
}
```

## 完整示例

```csharp
using System.Text;
using Microsoft.AspNetCore.Http;
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.DI;
using SPTarkov.Server.Core.Models.Common;
using SPTarkov.Server.Core.Models.Spt.Mod;
using SPTarkov.Server.Core.Models.Utils;
using SPTarkov.Server.Core.Servers;

namespace MyMod;

public record ModMetadata : AbstractModMetadata
{
    public override string ModGuid { get; init; } = "com.example.websocket";
    public override string Name { get; init; } = "WebSocket 示例";
    public override string Author { get; init; } = "开发者";
    public override SemanticVersioning.Version Version { get; init; } = new("1.0.0");
    public override SemanticVersioning.Range SptVersion { get; init; } = new("~4.0.0");
    public override string License { get; init; } = "MIT";
}

[Injectable(InjectionType = InjectionType.Singleton)]
public class CustomWebSocketConnectionHandler(
    ISptLogger<CustomWebSocketConnectionHandler> logger
) : IWebSocketConnectionHandler
{
    private readonly List<WebSocket> _connectedClients = [];

    public string GetHookUrl() => "/custom/socket/";
    public string GetSocketId() => "CustomWebSocket";

    public Task OnConnection(WebSocket ws, HttpContext context, string sessionIdContext)
    {
        _connectedClients.Add(ws);
        logger.Success($"新客户端连接! 当前连接数: {_connectedClients.Count}");
        return Task.CompletedTask;
    }

    public async Task OnMessage(byte[] rawData, WebSocketMessageType messageType, 
        WebSocket ws, HttpContext context)
    {
        var message = Encoding.UTF8.GetString(rawData);
        logger.Info($"收到消息: {message}");

        // 简单的命令处理
        var response = message switch
        {
            "ping" => "pong",
            "time" => DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"),
            "count" => $"当前连接数: {_connectedClients.Count}",
            _ => $"收到: {message}"
        };

        await ws.SendAsync(
            Encoding.UTF8.GetBytes(response),
            WebSocketMessageType.Text,
            true,
            CancellationToken.None
        );
    }

    public Task OnClose(WebSocket ws, HttpContext context, string sessionIdContext)
    {
        _connectedClients.Remove(ws);
        logger.Info($"客户端断开连接. 剩余连接数: {_connectedClients.Count}");
        return Task.CompletedTask;
    }
    
    // 广播消息给所有客户端
    public async Task BroadcastAsync(string message)
    {
        var data = Encoding.UTF8.GetBytes(message);
        foreach (var client in _connectedClients.ToList())
        {
            if (client.State == WebSocketState.Open)
            {
                await client.SendAsync(data, WebSocketMessageType.Text, true, CancellationToken.None);
            }
        }
    }
}

[Injectable]
public class CustomSptWebSocketHandler(
    ISptLogger<CustomSptWebSocketHandler> logger
) : ISptWebSocketMessageHandler
{
    public Task OnSptMessage(string sessionID, WebSocket client, byte[] rawData)
    {
        var message = Encoding.UTF8.GetString(rawData);
        logger.Info($"SPT 消息 [{sessionID}]: {message}");
        return Task.CompletedTask;
    }
}
```

## WebSocket 消息类型

| 类型 | 说明 |
|------|------|
| `Text` | 文本消息 |
| `Binary` | 二进制消息 |
| `Close` | 关闭连接 |
| `Ping` / `Pong` | 心跳检测 |

## 客户端连接示例（JavaScript）

```javascript
const ws = new WebSocket('ws://localhost:6969/custom/socket/');

ws.onopen = () => {
    console.log('已连接');
    ws.send('ping');
};

ws.onmessage = (event) => {
    console.log('收到消息:', event.data);
};

ws.onerror = (error) => {
    console.error('WebSocket 错误:', error);
};

ws.onclose = () => {
    console.log('连接已关闭');
};
```

## 小贴士

1. **使用单例模式**：连接处理器用 Singleton 确保全局唯一
2. **管理连接列表**：保存连接以便广播消息
3. **检查连接状态**：发送前检查 `WebSocketState.Open`
4. **异常处理**：WebSocket 操作可能抛出异常，要用 try-catch
5. **心跳机制**：定期发送 ping/pong 保持连接活跃
