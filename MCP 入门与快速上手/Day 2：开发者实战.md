好的，我们进入 **Day 2：开发者实战（C# 实作篇 1/3）**
Today’s focus: **将 Day1 的原型升级为更接近正式 MCP Server 的可扩展架构**

---

# ✅ **Day 2 学习目标 · Goals**

| 中文目标                              | English Goal                                                 |
| --------------------------------- | ------------------------------------------------------------ |
| 建立更正式的 MCP Server 架构（可扩展、可维护）     | Build a more scalable & maintainable MCP server architecture |
| 支持 **工具 (Tools)** 加入参数定义 & 类型校验   | Add parameter schema + validation for tools                  |
| 支持 **Prompts 模板**（可被模型调用使用的提示词模板） | Add *Prompts* module for reusable prompt templates           |
| 准备 Minimal API 版本骨架（为 Day3 铺路）    | Prepare Minimal API skeleton for HTTP-based MCP              |

---

## 📍 Day 2 重点方向（我们今天完成以下内容）

| 模块      | Day1 状态        | Day2 完成后                      |
| ------- | -------------- | ----------------------------- |
| Tools   | 只能执行，无法描述参数    | 支持 **参数Schema**、**描述**、**验证** |
| Prompts | 静态假数据          | 支持模板系统，可渲染变量                  |
| 架构      | 单文件/模块耦合       | 分层结构，更接近真实 MCP                |
| 通讯      | 单纯 JSON-RPC行通讯 | 保持不变，但加 Handler 结构化           |

---

# 🧱 第 1 步：升级项目结构（可扩展架构）

当前结构（Day1）比较偏 demo，我们升级为：

```
McpServer/
  ├─ Program.cs
  ├─ Rpc/
  │   ├─ JsonRpcModels.cs
  │   ├─ JsonRpcServer.cs
  │   └─ Handlers/
  │       ├─ InitializeHandler.cs
  │       ├─ ToolsHandler.cs
  │       └─ PromptsHandler.cs
  ├─ Core/
  │   ├─ ITool.cs
  │   ├─ ToolRegistry.cs
  │   ├─ ToolParameter.cs        <-- 新增
  │   ├─ IPrompt.cs              <-- 新增
  │   ├─ PromptRegistry.cs       <-- 新增
  │   ├─ Tools/
  │   └─ Prompts/
  └─ Resources/
```

---

# 🧩 第 2 步：为 Tools 加入 Schema（参数定义 + 验证）

### ✨ 新建：`Core/ToolParameter.cs`

```csharp
namespace McpServer.Core;

public record ToolParameter(
    string Name,
    string Type,          // string, number, boolean, array, object
    bool Required,
    string? Description = null
);
```

### 修改 `ITool.cs` 支持参数定义

```csharp
using System.Text.Json;

namespace McpServer.Core;

public interface ITool
{
    string Name { get; }
    string Description { get; }

    // 新增：参数定义（类似 JSON Schema）
    IEnumerable<ToolParameter> Parameters { get; }

    Task<object> InvokeAsync(JsonElement? args);
}
```

### ✅ 示例：重构 `SumTool` 支持参数 & 验证

```csharp
using System.Text.Json;

namespace McpServer.Core.Tools;

public class SumTool : ITool
{
    public string Name => "sum";
    public string Description => "Sum an array of numbers.";

    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("numbers", "array", true, "Array of numbers to sum")
    };

    public Task<object> InvokeAsync(JsonElement? args)
    {
        if (args is null || !args.Value.TryGetProperty("numbers", out var arr))
            throw new ArgumentException("Missing required parameter: numbers");

        double total = arr.EnumerateArray().Sum(n => n.GetDouble());

        return Task.FromResult<object>(new { total });
    }
}
```

---

# 🧠 第 3 步：加入 Prompts 模块（模板化提示词）

MCP 标准允许 Server 提供 Prompt 模板，例如：

* “summarize”
* “translate”
* “fix grammar”
* “write unit test”
* “generate SQL query”

我们先做一个小型 Prompt 模板引擎。

### 🆕 新建：`Core/IPrompt.cs`

```csharp
namespace McpServer.Core;

public interface ITemplatePrompt
{
    string Name { get; }
    string Description { get; }
    string Template { get; }

    string Render(Dictionary<string, string> vars);
}
```

### 🆕 新建：`Core/PromptRegistry.cs`

```csharp
namespace McpServer.Core;

public class PromptRegistry
{
    private readonly Dictionary<string, ITemplatePrompt> _prompts = new();

    public void Register(ITemplatePrompt prompt) => _prompts[prompt.Name] = prompt;

    public IEnumerable<object> List() =>
        _prompts.Values.Select(p => new {
            name = p.Name,
            description = p.Description
        });

    public string Render(string name, Dictionary<string, string> vars)
    {
        if (!_prompts.TryGetValue(name, out var p))
            throw new InvalidOperationException($"Prompt '{name}' not found");
        return p.Render(vars);
    }
}
```

### ✨ 示例 Prompt：`Prompts/SummarizePrompt.cs`

```csharp
using System.Text;

namespace McpServer.Core.Prompts;

public class SummarizePrompt : ITemplatePrompt
{
    public string Name => "summarize";
    public string Description => "Summarize the given text into 3 bullet points.";
    public string Template => "Summarize the following text into 3 key bullet points:\n\n{{text}}";

    public string Render(Dictionary<string, string> vars)
    {
        var output = Template;
        foreach (var v in vars)
            output = output.Replace("{{" + v.Key + "}}", v.Value);
        return output;
    }
}
```

> 🔥 在 Day3 我会加入变量 Schema + 类型检查，类似 Prompt 参数定义。

---

# 🧪 测试 Prompt 渲染（手动）

你可在 `Program.cs` 注册并测试：

```csharp
var prompts = new PromptRegistry();
prompts.Register(new SummarizePrompt());

var rendered = prompts.Render("summarize", new() { { "text", "MCP makes AI modular and interoperable." } });
Console.WriteLine(rendered);
```

输出示例：

```
Summarize the following text into 3 key bullet points:

MCP makes AI modular and interoperable.
```

---

# ✅ Day 2 阶段成果

| 完成内容           | 意义                        |
| -------------- | ------------------------- |
| Tool 参数 Schema | Tool 可被 IDE/LLM 自动理解 & 校验 |
| Prompt 模板引擎    | 实现 MCP Prompts 基础功能       |
| 架构拆分           | 开始支持扩展与模块化                |

---

# 📍 明天 Day 3 预告

Tomorrow we will:

| 模块                | 功能                                    |
| ----------------- | ------------------------------------- |
| **Minimal API**   | 提供 HTTP 兼容版本                          |
| Prompts 参数 Schema | Prompt 支持参数定义与校验                      |
| Resources 扩展      | 支持资源目录、列表、权限模型                        |
| 与客户端连接            | 教你怎么让 ChatGPT、Claude 或 Cursor 调用此 MCP |

---

如果你希望，我可以把 **Day1+Day2 的完整整理代码** 打包成 ZIP 发给你。
If you want the **combined Day1+Day2 full code**, tell me:

回复：
**“打包 Day1+Day2 代码”**

我会打包并提供下载。

是否需要？
好的，我们进入 **Day 2：开发者实战（C# 实作篇 1/3）**
Today’s focus: **将 Day1 的原型升级为更接近正式 MCP Server 的可扩展架构**

---

# ✅ **Day 2 学习目标 · Goals**

| 中文目标                              | English Goal                                                 |
| --------------------------------- | ------------------------------------------------------------ |
| 建立更正式的 MCP Server 架构（可扩展、可维护）     | Build a more scalable & maintainable MCP server architecture |
| 支持 **工具 (Tools)** 加入参数定义 & 类型校验   | Add parameter schema + validation for tools                  |
| 支持 **Prompts 模板**（可被模型调用使用的提示词模板） | Add *Prompts* module for reusable prompt templates           |
| 准备 Minimal API 版本骨架（为 Day3 铺路）    | Prepare Minimal API skeleton for HTTP-based MCP              |

---

## 📍 Day 2 重点方向（我们今天完成以下内容）

| 模块      | Day1 状态        | Day2 完成后                      |
| ------- | -------------- | ----------------------------- |
| Tools   | 只能执行，无法描述参数    | 支持 **参数Schema**、**描述**、**验证** |
| Prompts | 静态假数据          | 支持模板系统，可渲染变量                  |
| 架构      | 单文件/模块耦合       | 分层结构，更接近真实 MCP                |
| 通讯      | 单纯 JSON-RPC行通讯 | 保持不变，但加 Handler 结构化           |

---

# 🧱 第 1 步：升级项目结构（可扩展架构）

当前结构（Day1）比较偏 demo，我们升级为：

```
McpServer/
  ├─ Program.cs
  ├─ Rpc/
  │   ├─ JsonRpcModels.cs
  │   ├─ JsonRpcServer.cs
  │   └─ Handlers/
  │       ├─ InitializeHandler.cs
  │       ├─ ToolsHandler.cs
  │       └─ PromptsHandler.cs
  ├─ Core/
  │   ├─ ITool.cs
  │   ├─ ToolRegistry.cs
  │   ├─ ToolParameter.cs        <-- 新增
  │   ├─ IPrompt.cs              <-- 新增
  │   ├─ PromptRegistry.cs       <-- 新增
  │   ├─ Tools/
  │   └─ Prompts/
  └─ Resources/
```

---

# 🧩 第 2 步：为 Tools 加入 Schema（参数定义 + 验证）

### ✨ 新建：`Core/ToolParameter.cs`

```csharp
namespace McpServer.Core;

public record ToolParameter(
    string Name,
    string Type,          // string, number, boolean, array, object
    bool Required,
    string? Description = null
);
```

### 修改 `ITool.cs` 支持参数定义

```csharp
using System.Text.Json;

namespace McpServer.Core;

public interface ITool
{
    string Name { get; }
    string Description { get; }

    // 新增：参数定义（类似 JSON Schema）
    IEnumerable<ToolParameter> Parameters { get; }

    Task<object> InvokeAsync(JsonElement? args);
}
```

### ✅ 示例：重构 `SumTool` 支持参数 & 验证

```csharp
using System.Text.Json;

namespace McpServer.Core.Tools;

public class SumTool : ITool
{
    public string Name => "sum";
    public string Description => "Sum an array of numbers.";

    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("numbers", "array", true, "Array of numbers to sum")
    };

    public Task<object> InvokeAsync(JsonElement? args)
    {
        if (args is null || !args.Value.TryGetProperty("numbers", out var arr))
            throw new ArgumentException("Missing required parameter: numbers");

        double total = arr.EnumerateArray().Sum(n => n.GetDouble());

        return Task.FromResult<object>(new { total });
    }
}
```

---

# 🧠 第 3 步：加入 Prompts 模块（模板化提示词）

MCP 标准允许 Server 提供 Prompt 模板，例如：

* “summarize”
* “translate”
* “fix grammar”
* “write unit test”
* “generate SQL query”

我们先做一个小型 Prompt 模板引擎。

### 🆕 新建：`Core/IPrompt.cs`

```csharp
namespace McpServer.Core;

public interface ITemplatePrompt
{
    string Name { get; }
    string Description { get; }
    string Template { get; }

    string Render(Dictionary<string, string> vars);
}
```

### 🆕 新建：`Core/PromptRegistry.cs`

```csharp
namespace McpServer.Core;

public class PromptRegistry
{
    private readonly Dictionary<string, ITemplatePrompt> _prompts = new();

    public void Register(ITemplatePrompt prompt) => _prompts[prompt.Name] = prompt;

    public IEnumerable<object> List() =>
        _prompts.Values.Select(p => new {
            name = p.Name,
            description = p.Description
        });

    public string Render(string name, Dictionary<string, string> vars)
    {
        if (!_prompts.TryGetValue(name, out var p))
            throw new InvalidOperationException($"Prompt '{name}' not found");
        return p.Render(vars);
    }
}
```

### ✨ 示例 Prompt：`Prompts/SummarizePrompt.cs`

```csharp
using System.Text;

namespace McpServer.Core.Prompts;

public class SummarizePrompt : ITemplatePrompt
{
    public string Name => "summarize";
    public string Description => "Summarize the given text into 3 bullet points.";
    public string Template => "Summarize the following text into 3 key bullet points:\n\n{{text}}";

    public string Render(Dictionary<string, string> vars)
    {
        var output = Template;
        foreach (var v in vars)
            output = output.Replace("{{" + v.Key + "}}", v.Value);
        return output;
    }
}
```

> 🔥 在 Day3 我会加入变量 Schema + 类型检查，类似 Prompt 参数定义。

---

# 🧪 测试 Prompt 渲染（手动）

你可在 `Program.cs` 注册并测试：

```csharp
var prompts = new PromptRegistry();
prompts.Register(new SummarizePrompt());

var rendered = prompts.Render("summarize", new() { { "text", "MCP makes AI modular and interoperable." } });
Console.WriteLine(rendered);
```

输出示例：

```
Summarize the following text into 3 key bullet points:

MCP makes AI modular and interoperable.
```

---

# ✅ Day 2 阶段成果

| 完成内容           | 意义                        |
| -------------- | ------------------------- |
| Tool 参数 Schema | Tool 可被 IDE/LLM 自动理解 & 校验 |
| Prompt 模板引擎    | 实现 MCP Prompts 基础功能       |
| 架构拆分           | 开始支持扩展与模块化                |

---

# 📍 明天 Day 3 预告

Tomorrow we will:

| 模块                | 功能                                    |
| ----------------- | ------------------------------------- |
| **Minimal API**   | 提供 HTTP 兼容版本                          |
| Prompts 参数 Schema | Prompt 支持参数定义与校验                      |
| Resources 扩展      | 支持资源目录、列表、权限模型                        |
| 与客户端连接            | 教你怎么让 ChatGPT、Claude 或 Cursor 调用此 MCP |

---

如果你希望，我可以把 **Day1+Day2 的完整整理代码** 打包成 ZIP 发给你。
If you want the **combined Day1+Day2 full code**, tell me:

回复：
**“打包 Day1+Day2 代码”**

我会打包并提供下载。

是否需要？
