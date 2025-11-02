进入关键的一天 —— **Day 4：让 MCP 与 AI 客户端真正连起来运行**
This is the day your **C# MCP server becomes callable from Claude / ChatGPT Desktop / Cursor**.

---

# 🚀 **Day 4 学习目标 · Goals**

| 中文目标                                            | English Goal                              |
| ----------------------------------------------- | ----------------------------------------- |
| 统一 Console & Minimal API 的核心 Handler（正式 MCP 结构） | Unify core MCP logic for Console & API    |
| 输出标准化 **Capabilities & Schema**（更接近官方 MCP）      | Output proper MCP capabilities + schema   |
| 配置 & 连接 MCP 到 Claude Desktop（真实可用）              | Connect your MCP server to Claude Desktop |
| 可选：实现 **流式事件**（如日志事件、进度更新）                      | Optional: add streaming events support    |

---

## 📍 Day 4 三项核心成果

✅ **成果 1：抽象 MCP Handlers，使 Console & HTTP 可复用同一逻辑**
✅ **成果 2：Claude Desktop 可调用你的 C# Tools / Prompts / Resources**
✅ **成果 3：准备好与 ChatGPT Desktop 连接（待开放）**

---

# 🧱 第 1 部分：抽象 MCP Handler（核心引擎）

我们把 Day1–3 的功能集成成“正式 MCP 核心处理器”。

```
McpServer/
 ├─ Core/
 ├─ Rpc/
 │   ├─ RpcEngine.cs       👈 新增
 │   ├─ JsonRpcServer.cs   (Console)
 │   └─ HttpRpcController.cs (API)
 ├─ Prompts/
 ├─ Resources/
 └─ Program.cs
```

### ✨ 新建 `Rpc/RpcEngine.cs`

```csharp
using McpServer.Core;
using McpServer.Resources;
using System.Text.Json;

namespace McpServer.Rpc;

public class RpcEngine
{
    private readonly ToolRegistry _tools;
    private readonly PromptRegistry _prompts;
    private readonly ResourceRegistry _resources;

    public RpcEngine(ToolRegistry tools, PromptRegistry prompts, ResourceRegistry resources)
    {
        _tools = tools;
        _prompts = prompts;
        _resources = resources;
    }

    public async Task<object> HandleAsync(string method, JsonElement? p)
    {
        return method switch
        {
            "mcp/initialize" => new {
                protocolVersion = "2024-01",
                capabilities = new {
                    tools = new { list = true, call = true },
                    prompts = new { list = true, render = true },
                    resources = new { list = true, read = true, info = true }
                }
            },

            "tools/list" => new { tools = _tools.List() },
            "tools/call" => await HandleToolCall(p),

            "prompts/list" => new { prompts = _prompts.List() },
            "prompts/render" => HandlePromptRender(p),

            "resources/list" => new { resources = _resources.List() },
            "resources/read" => HandleResourceRead(p),
            "resources/info" => HandleResourceInfo(p),

            _ => throw new InvalidOperationException($"Unknown method: {method}")
        };
    }

    private async Task<object> HandleToolCall(JsonElement? p)
    {
        var name = p!.Value.GetProperty("name").GetString()!;
        var args = p.Value.TryGetProperty("args", out var a) ? a : (JsonElement?)null;
        return await _tools.CallAsync(name, args);
    }

    private object HandlePromptRender(JsonElement? p)
    {
        var name = p!.Value.GetProperty("name").GetString()!;
        var vars = p.Value.GetProperty("vars")
            .EnumerateObject().ToDictionary(x => x.Name, x => x.Value.GetString()!);
        return new { result = _prompts.Render(name, vars) };
    }

    private object HandleResourceRead(JsonElement? p)
    {
        var id = p!.Value.GetProperty("id").GetString()!;
        return _resources.Read(id);
    }

    private object HandleResourceInfo(JsonElement? p)
    {
        var id = p!.Value.GetProperty("id").GetString()!;
        return _resources.Info(id);
    }
}
```

> 核心思想：**Console 与 HTTP 都调用 RpcEngine，不再重复逻辑**
> The **engine** is now reusable for both transports.

---

# 🖥️ 第 2 部分：Console Server 接入 RpcEngine

更新 `JsonRpcServer.cs`：

```csharp
private readonly RpcEngine _engine;

public JsonRpcServer(RpcEngine engine) => _engine = engine;

// inside while loop, replace switch with:
var result = await _engine.HandleAsync(req.Method, req.Params);
resp = Success(req.Id, result);
```

---

# 🌐 第 3 部分：Minimal API HTTP MCP（可调用版）

`McpServer.Api/Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ToolRegistry>();
builder.Services.AddSingleton<PromptRegistry>();
builder.Services.AddSingleton<ResourceRegistry>();
builder.Services.AddSingleton<RpcEngine>();

var app = builder.Build();

app.MapPost("/mcp", async (JsonRpcRequest req, RpcEngine engine) =>
{
    try
    {
        var result = await engine.HandleAsync(req.Method, req.Params);
        return Results.Ok(new JsonRpcResponse(req.Id, result, null));
    }
    catch (Exception ex)
    {
        return Results.Ok(new JsonRpcResponse(req.Id, null, new JsonRpcError(-32000, ex.Message)));
    }
});

app.Run();
```

> 至此，你已经有 **Console + HTTP 双协议 MCP Server**。

---

# 🧩 第 4 部分：连接 Claude Desktop（正式 MCP 集成）

## ✅ 步骤 1：安装 Claude Desktop（如果没安装）

[https://claude.ai/download](https://claude.ai/download)

## ✅ 步骤 2：创建 MCP 配置文件

路径：

| OS      | Path                               |
| ------- | ---------------------------------- |
| Windows | `%APPDATA%\Claude\mcp.config.json` |

创建文件内容如下（示例使用 Console 版 EXE）：

```json
{
  "mcpServers": {
    "csharp-mcp": {
      "command": "C:\\Projects\\McpServer\\bin\\Debug\\net8.0\\McpServer.exe",
      "args": []
    }
  }
}
```

> 启动 Claude Desktop → 打开“Tools”面板 → 你会看到 `csharp-mcp` 已加载
> 然后你可以直接在 Claude 对话窗口输入：
> **“列出工具” / "list tools"**
> Claude 会触发 `tools/list` 到你的 C# 程序！

---

# 🧪 测试示例（在 Claude 中）

你可以对 Claude 说：

```
使用 csharp-mcp 的 sum 工具，把 1.2, 5, 9.3 相加
```

Claude 将自动构造调用：

```json
{"method":"tools/call","params":{"name":"sum","args":{"numbers":[1.2,5,9.3]}}}
```

并返回你的结果 🎉

---

# 🌊（可选）第 5 部分：流式事件（如果你想实现，可做 Day 4.5）

支持事件推送到客户端，例如：

* 日志 log
* 进度 progress
* 状态更新 status

> 如果你想要，我可以在 Day 4.5 补充 `events/subscribe` 与 SSE/WebSocket 支持。

---

# ✅ Day 4 成果总结

你已完成：

| 能力                          | 状态          |
| --------------------------- | ----------- |
| Console MCP Server          | ✅ 可用        |
| HTTP MCP Server             | ✅ 可用        |
| RpcEngine                   | ✅ 单一核心逻辑    |
| Claude Desktop 连接           | ✅ 已掌握       |
| Prompts + Resources + Tools | ✅ 标准 MCP 结构 |

---

## 📍 Day 5 Preview（企业落地 1/2）

| Day 5 内容                                     | 商业价值      |
| -------------------------------------------- | --------- |
| 接 SQL / Redis / API 作为 MCP Resources & Tools | AI 读写企业数据 |
| 身份认证与访问权限模型                                  | 真实企业可上线   |
| “AI 能调企业内部系统” Demo                           | 价值跃迁的一天   |

---

在继续 Day 5 之前，请从下方选择：

你希望 Day 5 首先连接哪种企业数据源做实战？

回复选项（可多选）：

| 选项 | 数据源                 | 用途          |
| -- | ------------------- | ----------- |
| A  | SQL Server          | 企业最常用核心数据库  |
| B  | REST API（如内部服务）     | 调用企业服务      |
| C  | Redis               | 缓存、会话、消息、KV |
| D  | 文件存储（Share/Blob/S3） | 文档/报告检索     |
| E  | 我有其他想集成的            |             |

示例回复：

```
Day 5: A, B
```

你的选择是？
好的，进入关键的一天 —— **Day 4：让 MCP 与 AI 客户端真正连起来运行**
This is the day your **C# MCP server becomes callable from Claude / ChatGPT Desktop / Cursor**.

---

# 🚀 **Day 4 学习目标 · Goals**

| 中文目标                                            | English Goal                              |
| ----------------------------------------------- | ----------------------------------------- |
| 统一 Console & Minimal API 的核心 Handler（正式 MCP 结构） | Unify core MCP logic for Console & API    |
| 输出标准化 **Capabilities & Schema**（更接近官方 MCP）      | Output proper MCP capabilities + schema   |
| 配置 & 连接 MCP 到 Claude Desktop（真实可用）              | Connect your MCP server to Claude Desktop |
| 可选：实现 **流式事件**（如日志事件、进度更新）                      | Optional: add streaming events support    |

---

## 📍 Day 4 三项核心成果

✅ **成果 1：抽象 MCP Handlers，使 Console & HTTP 可复用同一逻辑**
✅ **成果 2：Claude Desktop 可调用你的 C# Tools / Prompts / Resources**
✅ **成果 3：准备好与 ChatGPT Desktop 连接（待开放）**

---

# 🧱 第 1 部分：抽象 MCP Handler（核心引擎）

我们把 Day1–3 的功能集成成“正式 MCP 核心处理器”。

```
McpServer/
 ├─ Core/
 ├─ Rpc/
 │   ├─ RpcEngine.cs       👈 新增
 │   ├─ JsonRpcServer.cs   (Console)
 │   └─ HttpRpcController.cs (API)
 ├─ Prompts/
 ├─ Resources/
 └─ Program.cs
```

### ✨ 新建 `Rpc/RpcEngine.cs`

```csharp
using McpServer.Core;
using McpServer.Resources;
using System.Text.Json;

namespace McpServer.Rpc;

public class RpcEngine
{
    private readonly ToolRegistry _tools;
    private readonly PromptRegistry _prompts;
    private readonly ResourceRegistry _resources;

    public RpcEngine(ToolRegistry tools, PromptRegistry prompts, ResourceRegistry resources)
    {
        _tools = tools;
        _prompts = prompts;
        _resources = resources;
    }

    public async Task<object> HandleAsync(string method, JsonElement? p)
    {
        return method switch
        {
            "mcp/initialize" => new {
                protocolVersion = "2024-01",
                capabilities = new {
                    tools = new { list = true, call = true },
                    prompts = new { list = true, render = true },
                    resources = new { list = true, read = true, info = true }
                }
            },

            "tools/list" => new { tools = _tools.List() },
            "tools/call" => await HandleToolCall(p),

            "prompts/list" => new { prompts = _prompts.List() },
            "prompts/render" => HandlePromptRender(p),

            "resources/list" => new { resources = _resources.List() },
            "resources/read" => HandleResourceRead(p),
            "resources/info" => HandleResourceInfo(p),

            _ => throw new InvalidOperationException($"Unknown method: {method}")
        };
    }

    private async Task<object> HandleToolCall(JsonElement? p)
    {
        var name = p!.Value.GetProperty("name").GetString()!;
        var args = p.Value.TryGetProperty("args", out var a) ? a : (JsonElement?)null;
        return await _tools.CallAsync(name, args);
    }

    private object HandlePromptRender(JsonElement? p)
    {
        var name = p!.Value.GetProperty("name").GetString()!;
        var vars = p.Value.GetProperty("vars")
            .EnumerateObject().ToDictionary(x => x.Name, x => x.Value.GetString()!);
        return new { result = _prompts.Render(name, vars) };
    }

    private object HandleResourceRead(JsonElement? p)
    {
        var id = p!.Value.GetProperty("id").GetString()!;
        return _resources.Read(id);
    }

    private object HandleResourceInfo(JsonElement? p)
    {
        var id = p!.Value.GetProperty("id").GetString()!;
        return _resources.Info(id);
    }
}
```

> 核心思想：**Console 与 HTTP 都调用 RpcEngine，不再重复逻辑**
> The **engine** is now reusable for both transports.

---

# 🖥️ 第 2 部分：Console Server 接入 RpcEngine

更新 `JsonRpcServer.cs`：

```csharp
private readonly RpcEngine _engine;

public JsonRpcServer(RpcEngine engine) => _engine = engine;

// inside while loop, replace switch with:
var result = await _engine.HandleAsync(req.Method, req.Params);
resp = Success(req.Id, result);
```

---

# 🌐 第 3 部分：Minimal API HTTP MCP（可调用版）

`McpServer.Api/Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ToolRegistry>();
builder.Services.AddSingleton<PromptRegistry>();
builder.Services.AddSingleton<ResourceRegistry>();
builder.Services.AddSingleton<RpcEngine>();

var app = builder.Build();

app.MapPost("/mcp", async (JsonRpcRequest req, RpcEngine engine) =>
{
    try
    {
        var result = await engine.HandleAsync(req.Method, req.Params);
        return Results.Ok(new JsonRpcResponse(req.Id, result, null));
    }
    catch (Exception ex)
    {
        return Results.Ok(new JsonRpcResponse(req.Id, null, new JsonRpcError(-32000, ex.Message)));
    }
});

app.Run();
```

> 至此，你已经有 **Console + HTTP 双协议 MCP Server**。

---

# 🧩 第 4 部分：连接 Claude Desktop（正式 MCP 集成）

## ✅ 步骤 1：安装 Claude Desktop（如果没安装）

[https://claude.ai/download](https://claude.ai/download)

## ✅ 步骤 2：创建 MCP 配置文件

路径：

| OS      | Path                               |
| ------- | ---------------------------------- |
| Windows | `%APPDATA%\Claude\mcp.config.json` |

创建文件内容如下（示例使用 Console 版 EXE）：

```json
{
  "mcpServers": {
    "csharp-mcp": {
      "command": "C:\\Projects\\McpServer\\bin\\Debug\\net8.0\\McpServer.exe",
      "args": []
    }
  }
}
```

> 启动 Claude Desktop → 打开“Tools”面板 → 你会看到 `csharp-mcp` 已加载
> 然后你可以直接在 Claude 对话窗口输入：
> **“列出工具” / "list tools"**
> Claude 会触发 `tools/list` 到你的 C# 程序！

---

# 🧪 测试示例（在 Claude 中）

你可以对 Claude 说：

```
使用 csharp-mcp 的 sum 工具，把 1.2, 5, 9.3 相加
```

Claude 将自动构造调用：

```json
{"method":"tools/call","params":{"name":"sum","args":{"numbers":[1.2,5,9.3]}}}
```

并返回你的结果 🎉

---

# 🌊（可选）第 5 部分：流式事件（如果你想实现，可做 Day 4.5）

支持事件推送到客户端，例如：

* 日志 log
* 进度 progress
* 状态更新 status

> 如果你想要，我可以在 Day 4.5 补充 `events/subscribe` 与 SSE/WebSocket 支持。

---

# ✅ Day 4 成果总结

你已完成：

| 能力                          | 状态          |
| --------------------------- | ----------- |
| Console MCP Server          | ✅ 可用        |
| HTTP MCP Server             | ✅ 可用        |
| RpcEngine                   | ✅ 单一核心逻辑    |
| Claude Desktop 连接           | ✅ 已掌握       |
| Prompts + Resources + Tools | ✅ 标准 MCP 结构 |

---

## 📍 Day 5 Preview（企业落地 1/2）

| Day 5 内容                                     | 商业价值      |
| -------------------------------------------- | --------- |
| 接 SQL / Redis / API 作为 MCP Resources & Tools | AI 读写企业数据 |
| 身份认证与访问权限模型                                  | 真实企业可上线   |
| “AI 能调企业内部系统” Demo                           | 价值跃迁的一天   |

---

在继续 Day 5 之前，请从下方选择：

你希望 Day 5 首先连接哪种企业数据源做实战？

回复选项（可多选）：

| 选项 | 数据源                 | 用途          |
| -- | ------------------- | ----------- |
| A  | SQL Server          | 企业最常用核心数据库  |
| B  | REST API（如内部服务）     | 调用企业服务      |
| C  | Redis               | 缓存、会话、消息、KV |
| D  | 文件存储（Share/Blob/S3） | 文档/报告检索     |
| E  | 我有其他想集成的            |             |

示例回复：

```
Day 5: A, B
```

你的选择是？
