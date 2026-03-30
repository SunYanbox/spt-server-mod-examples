# 自定义 JSON 配置

如果你的模组需要自己的配置文件（比如让玩家自定义某些设置），可以使用 `ModHelper` 来读取 JSON 文件。

**源文件：** [5ReadCustomJsonConfig/ReadJsonConfig.cs](../5ReadCustomJsonConfig/ReadJsonConfig.cs)

## 应用场景

- 让玩家自定义模组参数
- 存储模组需要的额外数据
- 配置文件可以热修改，不需要重新编译

## 基本步骤

### 1. 创建配置文件

在模组目录下创建 `config.json`：

```json
{
    "ExampleProperty": "这是配置值",
    "SomeNumber": 100,
    "Enabled": true
}
```

### 2. 创建配置类

定义一个与 JSON 结构匹配的类：

```csharp
// 这个类要和 JSON 文件的结构一致
public record ModConfig
{
    public required string ExampleProperty { get; set; }
    public int SomeNumber { get; set; }
    public bool Enabled { get; set; }
}
```

### 3. 读取配置文件

使用 `ModHelper` 读取并解析：

```csharp
using System.Reflection;
using SPTarkov.Server.Core.Helpers;

public class MyMod(
    ModHelper modHelper
) : IOnLoad
{
    public Task OnLoad()
    {
        // 获取模组文件夹的绝对路径
        var pathToMod = modHelper.GetAbsolutePathToModFolder(Assembly.GetExecutingAssembly());
        
        // 读取配置文件，尖括号里是配置类的类型
        var config = modHelper.GetJsonDataFromFile<ModConfig>(pathToMod, "config.json");
        
        // 使用配置
        Console.WriteLine($"配置值: {config.ExampleProperty}");
        
        return Task.CompletedTask;
    }
}
```

## 完整示例

**config.json:**
```json
{
    "WelcomeMessage": "欢迎来到我的服务器！",
    "MaxItemsPerPlayer": 50,
    "EnableSpecialFeatures": true,
    "BlacklistedItems": [
        "item_id_1",
        "item_id_2"
    ]
}
```

**ModConfig.cs:**
```csharp
public record ModConfig
{
    public required string WelcomeMessage { get; set; }
    public int MaxItemsPerPlayer { get; set; }
    public bool EnableSpecialFeatures { get; set; }
    public List<string> BlacklistedItems { get; set; }
}
```

**MyMod.cs:**
```csharp
using System.Reflection;
using SPTarkov.DI.Annotations;
using SPTarkov.Server.Core.DI;
using SPTarkov.Server.Core.Helpers;
using SPTarkov.Server.Core.Models.Spt.Mod;
using SPTarkov.Server.Core.Models.Utils;

namespace MyMod;

public record ModMetadata : AbstractModMetadata
{
    public override string ModGuid { get; init; } = "com.example.jsonconfig";
    public override string Name { get; init; } = "JSON配置示例";
    public override string Author { get; init; } = "开发者";
    public override SemanticVersioning.Version Version { get; init; } = new("1.0.0");
    public override SemanticVersioning.Range SptVersion { get; init; } = new("~4.0.0");
    public override string License { get; init; } = "MIT";
}

[Injectable(TypePriority = OnLoadOrder.PreSptModLoader + 1)]
public class MyMod(
    ISptLogger<MyMod> logger,
    ModHelper modHelper
) : IOnLoad
{
    public Task OnLoad()
    {
        // 获取模组路径
        var pathToMod = modHelper.GetAbsolutePathToModFolder(Assembly.GetExecutingAssembly());
        
        // 读取配置
        var config = modHelper.GetJsonDataFromFile<ModConfig>(pathToMod, "config.json");
        
        // 输出配置信息
        logger.Success($"欢迎消息: {config.WelcomeMessage}");
        logger.Info($"每玩家最大物品数: {config.MaxItemsPerPlayer}");
        logger.Info($"特殊功能启用: {config.EnableSpecialFeatures}");
        
        // 遍历列表
        foreach (var itemId in config.BlacklistedItems)
        {
            logger.Debug($"黑名单物品: {itemId}");
        }
        
        return Task.CompletedTask;
    }
}

public record ModConfig
{
    public required string WelcomeMessage { get; set; }
    public int MaxItemsPerPlayer { get; set; }
    public bool EnableSpecialFeatures { get; set; }
    public List<string> BlacklistedItems { get; set; } = [];
}
```

## 嵌套配置

JSON 支持嵌套结构：

```json
{
    "Database": {
        "Host": "localhost",
        "Port": 3306
    },
    "Features": {
        "AutoSave": true,
        "SaveInterval": 60
    }
}
```

对应的 C# 类：

```csharp
public record ModConfig
{
    public DatabaseConfig Database { get; set; }
    public FeaturesConfig Features { get; set; }
}

public record DatabaseConfig
{
    public string Host { get; set; }
    public int Port { get; set; }
}

public record FeaturesConfig
{
    public bool AutoSave { get; set; }
    public int SaveInterval { get; set; }
}
```

## 小贴士

1. **类名可自定义**：配置类不一定要叫 `ModConfig`，可以按需命名
2. **JSON 要格式正确**：使用 JSON 验证器检查格式，避免解析错误
3. **属性要匹配**：C# 类的属性名要和 JSON 的键名一致
4. **路径用相对路径**：`config.json` 表示模组目录下的文件，也可以用子目录如 `data/settings.json`
5. **配置热更新**：玩家修改 JSON 文件后重启服务器即可生效，无需重新编译模组
