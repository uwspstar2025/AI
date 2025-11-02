我们进入 **Day 3：开发者实战（C# 实作篇 2/3）**
Today’s theme: **让 MCP Server 更接近真实可对接客户端（ChatGPT / Claude / Cursor）运行的版本**

---

# 🚀 **Day 3 学习目标 · Goals**

| 中文目标                                       | English Goal                                         |
| ------------------------------------------ | ---------------------------------------------------- |
| 支持 **Minimal API** 版本（HTTP 模式的 MCP Server） | Add Minimal API version to serve MCP over HTTP       |
| Prompts 支持参数 Schema & 校验                   | Add parameter schema + validation to Prompts         |
| Resources 扩展（支持列目录与多资源）                    | Expand Resources to list & manage multiple resources |
| MCP 客户端接入概念（如何让 ChatGPT/Claude/Cursor 调用）  | Explain how to connect MCP servers to LLM clients    |

---

Day 3 会交付 3 个核心成果：

✅ **成果 1**：Console 版 MCP Server（Day1+2）升级 → 可注册多资源、Prompt 参数化
✅ **成果 2**：**Minimal API** 方式运行 MCP Server（支持 HTTP 调用）
✅ **成果 3**：你将理解 MCP 如何被 ChatGPT/Claude/Cursor 调用（明天 Day 4 将真正连上）

---

## 🧱 第 1 部分：扩展 Resources 模块（更接近 MCP 规范）

**Day 1** 我们只有一个简单的 `resources/read`
**Day 3** 我们新增：

| 功能             | 描述                          |
| -------------- | --------------------------- |
| resources/list | 列资源目录（文件列表、数据库资源等）          |
| resources/read | 读取单一资源内容                    |
| resources/info | 返回资源的 Metadata（类型、大小、更新时间等） |

> MCP 规范中 Resources 是一级公民，必须具备“可发现性”。

### ✨ 创建资源模型

📄 `Resources/IResourceProvider.cs`

```csharp
namespace McpServer.Resources;

public interface IResourceProvider
{
    string Name { get; }
    IEnumerable<string> List();        // list resource IDs
    object Read(string id);            // read one resource
    object Info(string id);            // metadata
}
```

📄 新增一个文件系统资源提供者 `FileResourceProvider.cs`

```csharp
using System.Text;

namespace McpServer.Resources;

public class FileResourceProvider : IResourceProvider
{
    private readonly string _root;

    public FileResourceProvider(string rootDir) => _root = rootDir;

    public string Name => "files";

    public IEnumerable<string> List() =>
        Directory.Exists(_root)
            ? Directory.GetFiles(_root).Select(Path.GetFileName)!
            : Enumerable.Empty<string>();

    public object Read(string id)
    {
        var path = Path.Combine(_root, id);
        if (!File.Exists(path)) throw new FileNotFoundException(id);
        return new { id, content = File.ReadAllText(path) };
    }

    public object Info(string id)
    {
        var path = Path.Combine(_root, id);
        var fi = new FileInfo(path);
        return new {
            id,
            size = fi.Length,
            modified = fi.LastWriteTimeUtc
        };
    }
}
```

后面会注册：

```csharp
var resources = new ResourceRegistry();
resources.Register(new FileResourceProvider("./data"));
```

---

## ✍️ 第 2 部分：扩展 Prompts — 支持参数 Schema + 校验

Day 2 能渲染模板，但参数不规范
Day 3 → 支持参数定义

📄 `Core/PromptParameter.cs`

```csharp
namespace McpServer.Core;

public record PromptParameter(
    string Name,
    string Type,
    bool Required,
    string? Description = null
);
```

修改 `ITemplatePrompt` 接口：

```csharp
public interface ITemplatePrompt
{
    string Name { get; }
    string Description { get; }
    string Template { get; }
    IEnumerable<PromptParameter> Parameters { get; }

    string Render(Dictionary<string, string> vars);
}
```

更新 `SummarizePrompt` 示例：

```csharp
public IEnumerable<PromptParameter> Parameters => new[]
{
    new PromptParameter("text", "string", true, "Text to summarize")
};
```

> Day 4 我们会对齐到 **JSON Schema** 格式

---

## 🌐 第 3 部分：Minimal API 版本 MCP Server

为什么要做 **Minimal API 版本**？
Because 有的客户端 **不支持 stdio**，只支持 HTTP 调用。

例如：

* 内网 Web 应用
* 前端网页调用 MCP Server
* 需要 Docker & Kubernetes 部署

### 创建 Minimal API 项目

在解决方案下添加新项目：

```bash
dotnet new web -n McpServer.Api
```

Program.cs：

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapPost("/mcp", async (JsonRpcRequest req, ToolRegistry tools, PromptRegistry prompts, ResourceRegistry resources) =>
{
    // TODO: 使用与 Console 版相同 Handler（下一步抽象出来）
    return Results.Ok(new { ok = true });
});

app.Run();
```

> Day 4 我会进行 Handler 抽象，让 Console 与 HTTP 共用核心 MCP Logic。

---

## 🧠 第 4 部分：MCP 如何让 LLM 客户端调用？

不是所有 LLM 都能直接调用 Tools/App。
MCP 目前支持的客户端有：

| 客户端                          | MCP 支持情况                      |
| ---------------------------- | ----------------------------- |
| **Claude Desktop**           | ✅ 官方原生支持 MCP                  |
| **ChatGPT Desktop**          | ✅ 正在逐步开放（需配置 app-config.json） |
| Cursor IDE                   | ✅ 支持，常用于本地代理                  |
| VS Code 插件                   | 🟡 需要 MCP Bridge              |
| Web ChatGPT/Claude (browser) | ❌ 无法直接连接                      |

**核心概念：**
客户端会启动 MCP Server 作为一个 **子进程**，并通过 **stdin/stdout** 通信。

示意：

```
Claude Desktop
     ↓ (launches)
MCP Server (your C# exe)
     ↕ JSON-RPC (stdio)
```

Day 4 将带你写一个 **Claude Desktop 集成文件**：
`mcp.config.json`

---

# ✅ Day 3 阶段成果

到今天，你的 MCP Server 已具备：

| 能力                        | 状态                       |
| ------------------------- | ------------------------ |
| Tools (with parameters)   | ✅                        |
| Prompts (with parameters) | ✅                        |
| Resources: list/read/info | ✅                        |
| 可扩展架构                     | ✅                        |
| Minimal API 版本骨架          | ✅（Day4 完成 Handler 抽象后可跑） |

---

# 📍 Day 4 预告（实战篇 3/3）

Day 4 会非常关键，因为你将看到 MCP “真的能与 AI 连接”：

| Day 4 内容                          | 说明                               |
| --------------------------------- | -------------------------------- |
| Console + API 共用核心 Handler        | 提升为正式 MCP Server 结构              |
| 输出正式 MCP 接口 schema & capabilities | 与原生 MCP 更匹配                      |
| 完整 Claude Desktop 连接演示            | **你将能实际在 Claude 调用你的 C# MCP 工具** |
| 可选：支持流式响应（如 log/stream events）    | 更高级特性                            |

---

在我输出 **Day 3 完整代码包**前，我想确认一个偏好：

请选一个代码包样式（回复数字即可）：

| 选项 | 代码包风格                                                 |
| -- | ----------------------------------------------------- |
| 1  | 最简洁：只含必要文件（适合跟课写）                                     |
| 2  | 含详细注释（推荐）                                             |
| 3  | 含单元测试项目（xUnit）                                        |
| 4  | 企业级目录结构（含 Abstractions/Infrastructure/Application 分层） |
| 5  | 全部提供                                                  |

回复示例：
`3`

你的选择是？
