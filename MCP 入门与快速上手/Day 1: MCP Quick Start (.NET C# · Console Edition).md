下面是 **Day 1：MCP 入门与快速上手（.NET C# · Console 版）**
Below is **Day 1: MCP Quick Start (.NET C# · Console Edition)**

---

# 🎯 目标 · Goals

* **中文**：10–15 分钟跑通一个“类 MCP Server”原型：通过 **JSON-RPC(StdIO)** 暴露 Tools / Resources / Capabilities，并可被任意客户端调用。
* **EN**: In 10–15 minutes, run a “mini MCP server” over **JSON-RPC (StdIO)** that exposes **Tools / Resources / Capabilities** callable by any client.

> 说明/Note：官方 MCP 也是以 **类 LSP 的 stdio + JSON-RPC** 进行通信。今天我们先实现一个最小可用骨架，后续 Day2–4 再逐步对齐完整行为、错误码与能力协商细节。
> We’ll implement a minimal viable skeleton today; in Days 2–4 we’ll extend it toward full protocol behavior.

---

## 🧰 环境准备 · Environment

* **中文**：Windows + VS Code（C# Dev Kit 扩展）、.NET 8 SDK
* **EN**: Windows + VS Code (C# Dev Kit), .NET 8 SDK

**验证 / Verify**

```bash
dotnet --version   # 应≥ 8.x  Should be ≥ 8.x
```

---

## 🗂 项目初始化 · Project Init

```bash
mkdir McpMiniServer && cd McpMiniServer
dotnet new console -n McpServer
cd McpServer
dotnet add package System.Text.Json
```

项目结构 / Structure

```
McpMiniServer/
└─ McpServer/
   ├─ Program.cs
   ├─ Rpc/
   │  ├─ JsonRpcModels.cs
   │  └─ JsonRpcServer.cs
   ├─ Core/
   │  ├─ ITool.cs
   │  ├─ ToolRegistry.cs
   │  └─ Tools/
   │     ├─ GetTimeTool.cs
   │     └─ SumTool.cs
   └─ Resources/
      └─ SimpleFileResource.cs
```

在 VS Code 新建对应文件并粘贴以下代码。
Create the files and paste the code below.

---

### `Rpc/JsonRpcModels.cs`

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace McpServer.Rpc;

public record JsonRpcRequest(
    [property: JsonPropertyName("id")] string? Id,
    [property: JsonPropertyName("method")] string Method,
    [property: JsonPropertyName("params")] JsonElement? Params
);

public record JsonRpcError(
    [property: JsonPropertyName("code")] int Code,
    [property: JsonPropertyName("message")] string Message
);

public record JsonRpcResponse(
    [property: JsonPropertyName("id")] string? Id,
    [property: JsonPropertyName("result")] object? Result,
    [property: JsonPropertyName("error")] JsonRpcError? Error
);

public static class JsonUtil
{
    public static readonly JsonSerializerOptions Options = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = false
    };
}
```

---

### `Core/ITool.cs`

```csharp
using System.Text.Json;
using System.Threading.Tasks;

namespace McpServer.Core;

public interface ITool
{
    string Name { get; }
    string Description { get; }
    Task<object> InvokeAsync(JsonElement? args);
}
```

---

### `Core/ToolRegistry.cs`

```csharp
using System.Text.Json;

namespace McpServer.Core;

public class ToolRegistry
{
    private readonly Dictionary<string, ITool> _tools = new();

    public void Register(ITool tool) => _tools[tool.Name] = tool;

    public IEnumerable<object> List() => _tools.Values.Select(t => new {
        name = t.Name,
        description = t.Description
    });

    public async Task<object> CallAsync(string name, JsonElement? args)
    {
        if (!_tools.TryGetValue(name, out var tool))
            throw new InvalidOperationException($"Tool '{name}' not found.");
        return await tool.InvokeAsync(args);
    }
}
```

---

### `Core/Tools/GetTimeTool.cs`

```csharp
using System.Text.Json;

namespace McpServer.Core.Tools;

public class GetTimeTool : ITool
{
    public string Name => "getTime";
    public string Description => "Return current server time in ISO 8601.";

    public Task<object> InvokeAsync(JsonElement? args)
        => Task.FromResult<object>(new { now = DateTimeOffset.UtcNow.ToString("o") });
}
```

---

### `Core/Tools/SumTool.cs`

```csharp
using System.Text.Json;

namespace McpServer.Core.Tools;

public class SumTool : ITool
{
    public string Name => "sum";
    public string Description => "Sum an array of numbers: { numbers: number[] }";

    public Task<object> InvokeAsync(JsonElement? args)
    {
        if (args is null || !args.Value.TryGetProperty("numbers", out var arr) || arr.ValueKind != System.Text.Json.JsonValueKind.Array)
            throw new ArgumentException("Missing 'numbers' array.");

        double total = 0;
        foreach (var n in arr.EnumerateArray())
            total += n.GetDouble();
        return Task.FromResult<object>(new { total });
    }
}
```

---

### `Resources/SimpleFileResource.cs`

```csharp
using System.Text.Json;

namespace McpServer.Resources;

public static class SimpleFileResource
{
    // For demo: read a text file by path { path: "..." }
    public static object Read(JsonElement? args)
    {
        if (args is null || !args.Value.TryGetProperty("path", out var p))
            throw new ArgumentException("Missing 'path'.");
        var path = p.GetString()!;
        if (!File.Exists(path)) throw new FileNotFoundException(path);
        var content = File.ReadAllText(path);
        return new { path, content };
    }
}
```

---

### `Rpc/JsonRpcServer.cs`

```csharp
using System.Text;
using System.Text.Json;
using McpServer.Core;
using McpServer.Resources;

namespace McpServer.Rpc;

public class JsonRpcServer
{
    private readonly ToolRegistry _tools;

    public JsonRpcServer(ToolRegistry tools) => _tools = tools;

    public async Task RunAsync(CancellationToken ct = default)
    {
        using var reader = new StreamReader(Console.OpenStandardInput(), Encoding.UTF8);
        using var writer = new StreamWriter(Console.OpenStandardOutput(), Encoding.UTF8) { AutoFlush = true };

        while (!ct.IsCancellationRequested)
        {
            var line = await reader.ReadLineAsync();
            if (line is null) break;
            if (string.IsNullOrWhiteSpace(line)) continue;

            JsonRpcResponse resp;
            try
            {
                var req = JsonSerializer.Deserialize<JsonRpcRequest>(line, JsonUtil.Options)!;
                resp = await HandleAsync(req);
            }
            catch (Exception ex)
            {
                resp = new JsonRpcResponse(null, null, new JsonRpcError(-32700, $"Parse error: {ex.Message}"));
            }

            var json = JsonSerializer.Serialize(resp, JsonUtil.Options);
            await writer.WriteLineAsync(json);
        }
    }

    private async Task<JsonRpcResponse> HandleAsync(JsonRpcRequest req)
    {
        try
        {
            return req.Method switch
            {
                "mcp/initialize" => Success(req.Id, new {
                    protocol = "mcp-mini/0.1",
                    capabilities = new {
                        tools = true,
                        resources = true,
                        prompts = true
                    }
                }),

                // list tools
                "tools/list" => Success(req.Id, new { tools = _tools.List() }),

                // call tool: params { name: "...", args: {...} }
                "tools/call" => Success(req.Id, await HandleToolCall(req.Params)),

                // resource read: params { path: "..." }
                "resources/read" => Success(req.Id, SimpleFileResource.Read(req.Params)),

                // prompts/list: demo only
                "prompts/list" => Success(req.Id, new { prompts = new[] {
                    new { name = "summarize", description = "Summarize given text." }
                }}),

                _ => Error(req.Id, -32601, $"Method not found: {req.Method}")
            };
        }
        catch (Exception ex)
        {
            return Error(req.Id, -32000, ex.Message);
        }
    }

    private async Task<object> HandleToolCall(JsonElement? p)
    {
        if (p is null) throw new ArgumentException("Missing params.");
        var name = p.Value.GetProperty("name").GetString()!;
        var args = p.Value.TryGetProperty("args", out var a) ? a : (JsonElement?)null;
        return await _tools.CallAsync(name, args);
    }

    private static JsonRpcResponse Success(string? id, object result)
        => new(id, result, null);

    private static JsonRpcResponse Error(string? id, int code, string message)
        => new(id, null, new JsonRpcError(code, message));
}
```

---

### `Program.cs`

```csharp
using McpServer.Core;
using McpServer.Core.Tools;
using McpServer.Rpc;

var tools = new ToolRegistry();
tools.Register(new GetTimeTool());
tools.Register(new SumTool());

var server = new JsonRpcServer(tools);
Console.Error.WriteLine("MCP-mini server started. Send JSON-RPC lines via STDIN.");
await server.RunAsync();
```

---

## ▶️ 运行与测试 · Run & Test

1. **编译 / Build**

```bash
dotnet build
```

2. **运行 / Run**

```bash
dotnet run
```

控制台会等待你通过 **一行一行 JSON** 输入请求。
The server waits for **one-line JSON** requests.

3. **示例请求 / Example requests**
   把以下每行粘贴回车（逐条发送）：
   Paste each line and press Enter:

* **Initialize / 能力协商**

```json
{"id":"1","method":"mcp/initialize","params":{}}
```

* **列出工具 / List tools**

```json
{"id":"2","method":"tools/list","params":{}}
```

* **调用工具：时间 / Call tool: time**

```json
{"id":"3","method":"tools/call","params":{"name":"getTime","args":{}}}
```

* **调用工具：求和 / Call tool: sum**

```json
{"id":"4","method":"tools/call","params":{"name":"sum","args":{"numbers":[1,2,3.5]}}}
```

* **读取资源：文件 / Read resource: file**
  先创建一个文件：

```bash
echo Hello-MCP > demo.txt
```

再请求：

```json
{"id":"5","method":"resources/read","params":{"path":"demo.txt"}}
```

你将看到每条请求对应一条 JSON 响应。
You’ll get one JSON response per request line.

> 提示/Tips：真实客户端（如 ChatGPT/Claude/Cursor）也是通过 stdio 管道与服务通信；我们在 Day2–4 会接上更贴近官方的握手与 schema，并增加批量传输、错误类别、流式等。

---

## 🧪 你可以试着扩展 · Try Extending

* **中文**：新增一个 `fetchUrl` 工具（输入 `url`，返回状态码与前 200 个字符）。
* **EN**: Add a `fetchUrl` tool that returns status code & first 200 chars of the page.

> 明天（Day 2）我会带你：
>
> * 抽象 **Tool schema（参数校验/JSON Schema）**
> * 支持 **Prompts 的模板渲染**（例如 summarize、translate）
> * 提供 **Minimal API** 版本（HTTP 直连）与 **WebAPI 分层架构**版骨架
> * 准备 **Dockerfile** 与本地/远程调试

---

## ❓常见问题 · FAQ

**Q**：为什么用 **一行一个 JSON**？
**A**：简化了 framing（与 LSP/MCP 的 `Content-Length` 头部思路一致），便于入门与手工测试。

**Q**：如何让 LLM 客户端调用？
**A**：许多 IDE/Agent 客户端支持 **stdio 工具进程**。在 Day3 我会给出 **客户端适配器**（把 LLM 的 Tool 调用映射为上述 JSON-RPC）。

---

如果你希望，我可以把以上代码**打包成 ZIP** 或 **加注释版源文件**发给你。
If you’d like, I can package the code as a ZIP or provide a “commented” version of all files.

需要的话直接说 “**打包代码**”。
Say “**package the code**” if you want it prepared.
下面是 **Day 1：MCP 入门与快速上手（.NET C# · Console 版）**
Below is **Day 1: MCP Quick Start (.NET C# · Console Edition)**

---

# 🎯 目标 · Goals

* **中文**：10–15 分钟跑通一个“类 MCP Server”原型：通过 **JSON-RPC(StdIO)** 暴露 Tools / Resources / Capabilities，并可被任意客户端调用。
* **EN**: In 10–15 minutes, run a “mini MCP server” over **JSON-RPC (StdIO)** that exposes **Tools / Resources / Capabilities** callable by any client.

> 说明/Note：官方 MCP 也是以 **类 LSP 的 stdio + JSON-RPC** 进行通信。今天我们先实现一个最小可用骨架，后续 Day2–4 再逐步对齐完整行为、错误码与能力协商细节。
> We’ll implement a minimal viable skeleton today; in Days 2–4 we’ll extend it toward full protocol behavior.

---

## 🧰 环境准备 · Environment

* **中文**：Windows + VS Code（C# Dev Kit 扩展）、.NET 8 SDK
* **EN**: Windows + VS Code (C# Dev Kit), .NET 8 SDK

**验证 / Verify**

```bash
dotnet --version   # 应≥ 8.x  Should be ≥ 8.x
```

---

## 🗂 项目初始化 · Project Init

```bash
mkdir McpMiniServer && cd McpMiniServer
dotnet new console -n McpServer
cd McpServer
dotnet add package System.Text.Json
```

项目结构 / Structure

```
McpMiniServer/
└─ McpServer/
   ├─ Program.cs
   ├─ Rpc/
   │  ├─ JsonRpcModels.cs
   │  └─ JsonRpcServer.cs
   ├─ Core/
   │  ├─ ITool.cs
   │  ├─ ToolRegistry.cs
   │  └─ Tools/
   │     ├─ GetTimeTool.cs
   │     └─ SumTool.cs
   └─ Resources/
      └─ SimpleFileResource.cs
```

在 VS Code 新建对应文件并粘贴以下代码。
Create the files and paste the code below.

---

### `Rpc/JsonRpcModels.cs`

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace McpServer.Rpc;

public record JsonRpcRequest(
    [property: JsonPropertyName("id")] string? Id,
    [property: JsonPropertyName("method")] string Method,
    [property: JsonPropertyName("params")] JsonElement? Params
);

public record JsonRpcError(
    [property: JsonPropertyName("code")] int Code,
    [property: JsonPropertyName("message")] string Message
);

public record JsonRpcResponse(
    [property: JsonPropertyName("id")] string? Id,
    [property: JsonPropertyName("result")] object? Result,
    [property: JsonPropertyName("error")] JsonRpcError? Error
);

public static class JsonUtil
{
    public static readonly JsonSerializerOptions Options = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = false
    };
}
```

---

### `Core/ITool.cs`

```csharp
using System.Text.Json;
using System.Threading.Tasks;

namespace McpServer.Core;

public interface ITool
{
    string Name { get; }
    string Description { get; }
    Task<object> InvokeAsync(JsonElement? args);
}
```

---

### `Core/ToolRegistry.cs`

```csharp
using System.Text.Json;

namespace McpServer.Core;

public class ToolRegistry
{
    private readonly Dictionary<string, ITool> _tools = new();

    public void Register(ITool tool) => _tools[tool.Name] = tool;

    public IEnumerable<object> List() => _tools.Values.Select(t => new {
        name = t.Name,
        description = t.Description
    });

    public async Task<object> CallAsync(string name, JsonElement? args)
    {
        if (!_tools.TryGetValue(name, out var tool))
            throw new InvalidOperationException($"Tool '{name}' not found.");
        return await tool.InvokeAsync(args);
    }
}
```

---

### `Core/Tools/GetTimeTool.cs`

```csharp
using System.Text.Json;

namespace McpServer.Core.Tools;

public class GetTimeTool : ITool
{
    public string Name => "getTime";
    public string Description => "Return current server time in ISO 8601.";

    public Task<object> InvokeAsync(JsonElement? args)
        => Task.FromResult<object>(new { now = DateTimeOffset.UtcNow.ToString("o") });
}
```

---

### `Core/Tools/SumTool.cs`

```csharp
using System.Text.Json;

namespace McpServer.Core.Tools;

public class SumTool : ITool
{
    public string Name => "sum";
    public string Description => "Sum an array of numbers: { numbers: number[] }";

    public Task<object> InvokeAsync(JsonElement? args)
    {
        if (args is null || !args.Value.TryGetProperty("numbers", out var arr) || arr.ValueKind != System.Text.Json.JsonValueKind.Array)
            throw new ArgumentException("Missing 'numbers' array.");

        double total = 0;
        foreach (var n in arr.EnumerateArray())
            total += n.GetDouble();
        return Task.FromResult<object>(new { total });
    }
}
```

---

### `Resources/SimpleFileResource.cs`

```csharp
using System.Text.Json;

namespace McpServer.Resources;

public static class SimpleFileResource
{
    // For demo: read a text file by path { path: "..." }
    public static object Read(JsonElement? args)
    {
        if (args is null || !args.Value.TryGetProperty("path", out var p))
            throw new ArgumentException("Missing 'path'.");
        var path = p.GetString()!;
        if (!File.Exists(path)) throw new FileNotFoundException(path);
        var content = File.ReadAllText(path);
        return new { path, content };
    }
}
```

---

### `Rpc/JsonRpcServer.cs`

```csharp
using System.Text;
using System.Text.Json;
using McpServer.Core;
using McpServer.Resources;

namespace McpServer.Rpc;

public class JsonRpcServer
{
    private readonly ToolRegistry _tools;

    public JsonRpcServer(ToolRegistry tools) => _tools = tools;

    public async Task RunAsync(CancellationToken ct = default)
    {
        using var reader = new StreamReader(Console.OpenStandardInput(), Encoding.UTF8);
        using var writer = new StreamWriter(Console.OpenStandardOutput(), Encoding.UTF8) { AutoFlush = true };

        while (!ct.IsCancellationRequested)
        {
            var line = await reader.ReadLineAsync();
            if (line is null) break;
            if (string.IsNullOrWhiteSpace(line)) continue;

            JsonRpcResponse resp;
            try
            {
                var req = JsonSerializer.Deserialize<JsonRpcRequest>(line, JsonUtil.Options)!;
                resp = await HandleAsync(req);
            }
            catch (Exception ex)
            {
                resp = new JsonRpcResponse(null, null, new JsonRpcError(-32700, $"Parse error: {ex.Message}"));
            }

            var json = JsonSerializer.Serialize(resp, JsonUtil.Options);
            await writer.WriteLineAsync(json);
        }
    }

    private async Task<JsonRpcResponse> HandleAsync(JsonRpcRequest req)
    {
        try
        {
            return req.Method switch
            {
                "mcp/initialize" => Success(req.Id, new {
                    protocol = "mcp-mini/0.1",
                    capabilities = new {
                        tools = true,
                        resources = true,
                        prompts = true
                    }
                }),

                // list tools
                "tools/list" => Success(req.Id, new { tools = _tools.List() }),

                // call tool: params { name: "...", args: {...} }
                "tools/call" => Success(req.Id, await HandleToolCall(req.Params)),

                // resource read: params { path: "..." }
                "resources/read" => Success(req.Id, SimpleFileResource.Read(req.Params)),

                // prompts/list: demo only
                "prompts/list" => Success(req.Id, new { prompts = new[] {
                    new { name = "summarize", description = "Summarize given text." }
                }}),

                _ => Error(req.Id, -32601, $"Method not found: {req.Method}")
            };
        }
        catch (Exception ex)
        {
            return Error(req.Id, -32000, ex.Message);
        }
    }

    private async Task<object> HandleToolCall(JsonElement? p)
    {
        if (p is null) throw new ArgumentException("Missing params.");
        var name = p.Value.GetProperty("name").GetString()!;
        var args = p.Value.TryGetProperty("args", out var a) ? a : (JsonElement?)null;
        return await _tools.CallAsync(name, args);
    }

    private static JsonRpcResponse Success(string? id, object result)
        => new(id, result, null);

    private static JsonRpcResponse Error(string? id, int code, string message)
        => new(id, null, new JsonRpcError(code, message));
}
```

---

### `Program.cs`

```csharp
using McpServer.Core;
using McpServer.Core.Tools;
using McpServer.Rpc;

var tools = new ToolRegistry();
tools.Register(new GetTimeTool());
tools.Register(new SumTool());

var server = new JsonRpcServer(tools);
Console.Error.WriteLine("MCP-mini server started. Send JSON-RPC lines via STDIN.");
await server.RunAsync();
```

---

## ▶️ 运行与测试 · Run & Test

1. **编译 / Build**

```bash
dotnet build
```

2. **运行 / Run**

```bash
dotnet run
```

控制台会等待你通过 **一行一行 JSON** 输入请求。
The server waits for **one-line JSON** requests.

3. **示例请求 / Example requests**
   把以下每行粘贴回车（逐条发送）：
   Paste each line and press Enter:

* **Initialize / 能力协商**

```json
{"id":"1","method":"mcp/initialize","params":{}}
```

* **列出工具 / List tools**

```json
{"id":"2","method":"tools/list","params":{}}
```

* **调用工具：时间 / Call tool: time**

```json
{"id":"3","method":"tools/call","params":{"name":"getTime","args":{}}}
```

* **调用工具：求和 / Call tool: sum**

```json
{"id":"4","method":"tools/call","params":{"name":"sum","args":{"numbers":[1,2,3.5]}}}
```

* **读取资源：文件 / Read resource: file**
  先创建一个文件：

```bash
echo Hello-MCP > demo.txt
```

再请求：

```json
{"id":"5","method":"resources/read","params":{"path":"demo.txt"}}
```

你将看到每条请求对应一条 JSON 响应。
You’ll get one JSON response per request line.

> 提示/Tips：真实客户端（如 ChatGPT/Claude/Cursor）也是通过 stdio 管道与服务通信；我们在 Day2–4 会接上更贴近官方的握手与 schema，并增加批量传输、错误类别、流式等。

---

## 🧪 你可以试着扩展 · Try Extending

* **中文**：新增一个 `fetchUrl` 工具（输入 `url`，返回状态码与前 200 个字符）。
* **EN**: Add a `fetchUrl` tool that returns status code & first 200 chars of the page.

> 明天（Day 2）我会带你：
>
> * 抽象 **Tool schema（参数校验/JSON Schema）**
> * 支持 **Prompts 的模板渲染**（例如 summarize、translate）
> * 提供 **Minimal API** 版本（HTTP 直连）与 **WebAPI 分层架构**版骨架
> * 准备 **Dockerfile** 与本地/远程调试

---

## ❓常见问题 · FAQ

**Q**：为什么用 **一行一个 JSON**？
**A**：简化了 framing（与 LSP/MCP 的 `Content-Length` 头部思路一致），便于入门与手工测试。

**Q**：如何让 LLM 客户端调用？
**A**：许多 IDE/Agent 客户端支持 **stdio 工具进程**。在 Day3 我会给出 **客户端适配器**（把 LLM 的 Tool 调用映射为上述 JSON-RPC）。

---

如果你希望，我可以把以上代码**打包成 ZIP** 或 **加注释版源文件**发给你。
If you’d like, I can package the code as a ZIP or provide a “commented” version of all files.

需要的话直接说 “**打包代码**”。
Say “**package the code**” if you want it prepared.
