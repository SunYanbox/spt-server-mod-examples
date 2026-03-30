# SPT Server Mod Examples - 项目上下文

## 项目概述

这是 **SPT (Single Player Tarkov)** 服务器模组的示例代码集合，使用 **C# / .NET 9** 开发。项目提供从简单到复杂的多层次示例，帮助开发者学习如何为 SPT 服务器创建模组。

### 核心技术
- **语言**: C# (.NET 9)
- **框架**: SPTarkov.Server.Core
- **架构**: 依赖注入 (DI) + 控制反转 (IoC)
- **IDE**: Visual Studio Community / JetBrains Rider

## 项目结构

```
server-mod-examples/
├── ModExamples.sln          # Visual Studio 解决方案文件
├── 1Logging/                # 日志记录基础示例
├── 2EditDatabase/           # 数据库编辑示例
├── 3EditSptConfig/          # SPT 配置修改示例
├── 5ReadCustomJsonConfig/   # 读取自定义 JSON 配置
├── 6OverrideMethod/         # 方法覆盖 (继承方式)
├── 6.1OverrideMethodHarmony/# 方法覆盖 (Harmony 补丁)
├── 7UseMultipleClasses/     # 多类协作示例
├── 8OnLoad/                 # 服务器加载时执行
├── 9OnUpdate/               # 每帧更新循环
├── 10CustomRoute/           # 自定义 HTTP 路由
├── 11RegisterClassesInDI/   # 依赖注入注册
├── 12Bundle/                # 资源包加载
├── 13AddTraderWithAssortJson/       # 添加商人 (JSON assort)
├── 13.1AddTraderWithDynamicAssorts/ # 添加商人 (动态 assort)
├── 14AfterDBLoadHook/       # 数据库加载后钩子
├── 15HttpListenerExample/   # HTTP 监听器
├── 18CustomItemService/     # 自定义物品服务
├── 18.1CustomItemServiceLootBox/ # 战利品箱服务
├── 20CustomChatBot/         # 自定义聊天机器人
├── 21CustomCommandoCommand/ # 自定义 Commando 命令
├── 22CustomSptCommand/      # 自定义 SPT 命令
├── 23CustomAbstractChatBot/ # 抽象聊天机器人实现
├── 24Websocket/             # WebSocket 连接示例
└── 25AddCustomLocales/      # 自定义本地化文本
```

## 构建与运行

### 前置要求
- [.NET 9 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/9.0)
- Visual Studio Community 或 JetBrains Rider

### 构建命令
```bash
# 恢复依赖
dotnet restore

# Debug 构建
dotnet build --configuration Debug

# Release 构建
dotnet build --configuration Release
```

### 分发模组
1. 使用 Release 模式构建项目
2. 复制 `mod\bin\Release\<模组名称>` 文件夹到 SPT 服务器的 `/mods` 目录
3. 启动服务器

## 开发约定

### 必需组件: ModMetadata

每个模组必须定义 `ModMetadata` 记录，包含模组元数据：

```csharp
public record ModMetadata : AbstractModMetadata
{
    public override string ModGuid { get; init; } = "com.example.mymod";  // 唯一标识符
    public override string Name { get; init; } = "MyMod";                  // 模组名称
    public override string Author { get; init; } = "AuthorName";           // 作者
    public override SemanticVersioning.Version Version { get; init; } = new("1.0.0");
    public override SemanticVersioning.Range SptVersion { get; init; } = new("~4.0.0");
    public override string License { get; init; } = "MIT";
    // 其他可选属性: Contributors, Incompatibilities, ModDependencies, Url, IsBundleMod
}
```

### 依赖注入

使用 `[Injectable]` 属性标记可注入类：

```csharp
// 单例模式 - 全局唯一实例
[Injectable(InjectionType.Singleton)]
public class MySingletonService { }

// 作用域模式 - 每次请求创建新实例
[Injectable(InjectionType.Scoped)]
public class MyScopedService { }

// 指定加载顺序
[Injectable(TypePriority = OnLoadOrder.PostDBModLoader + 1)]
public class MyMod : IOnLoad { }
```

### 加载顺序优先级

常用 `OnLoadOrder` 值：
- `PreSptModLoader` - SPT 模组加载器之前
- `PostSptModLoader` - SPT 模组加载器之后
- `PostDBModLoader` - 数据库加载之后
- `Watermark` - 水印阶段

### 生命周期接口

- **IOnLoad**: 服务器加载时执行一次
- **IOnUpdate**: 每帧执行（游戏循环）

### 构造函数注入

依赖通过构造函数参数自动注入：

```csharp
[Injectable(TypePriority = OnLoadOrder.PostDBModLoader + 1)]
public class MyMod(
    ISptLogger<MyMod> logger,      // 日志服务
    ConfigServer configServer,      // 配置服务
    DatabaseServer databaseServer   // 数据库服务
) : IOnLoad
{
    public Task OnLoad()
    {
        logger.Success("Mod loaded!");
        return Task.CompletedTask;
    }
}
```

### 方法覆盖

通过继承并设置相同的 `TypePriority` 来覆盖服务器方法：

```csharp
[Injectable(TypePriority = OnLoadOrder.Watermark)]
public class MyWatermark : Watermark  // 继承原始类
{
    public override async Task OnLoad()
    {
        // 自定义逻辑
        await base.OnLoad();  // 可选：调用原始方法
    }
}
```

## NuGet 包依赖

每个模组项目引用以下核心包：

```xml
<PackageReference Include="SPTarkov.Common" Version="4.0.13" />
<PackageReference Include="SPTarkov.DI" Version="4.0.13" />
<PackageReference Include="SPTarkov.Server.Core" Version="4.0.13" />
```

## 常用命名空间

```csharp
using SPTarkov.DI.Annotations;                    // Injectable 属性
using SPTarkov.Server.Core.DI;                    // DI 相关接口
using SPTarkov.Server.Core.Models.Spt.Mod;        // ModMetadata 基类
using SPTarkov.Server.Core.Models.Utils;          // 日志等工具
using SPTarkov.Server.Core.Servers;               // 服务器类
using SPTarkov.Server.Core.Services;              // 各种服务
using SPTarkov.Server.Core.Helpers;               // 辅助类
```

## 示例学习路径

按编号顺序学习：
1. **1Logging** - 了解日志系统
2. **2EditDatabase** - 数据库操作基础
3. **3EditSptConfig** - 配置修改
4. **5ReadCustomJsonConfig** - 自定义配置文件
5. **6OverrideMethod** - 方法覆盖基础
6. **7UseMultipleClasses** - 代码组织
7. **11RegisterClassesInDI** - 依赖注入深入
8. **13AddTraderWithAssortJson** - 添加商人 (综合示例)

## CI/CD

项目使用 GitHub Actions 自动构建：
- 触发条件: push 到 main 分支 或 pull request
- 构建环境: Ubuntu + .NET 9
- 配置文件: `.github/workflows/build.yml`
