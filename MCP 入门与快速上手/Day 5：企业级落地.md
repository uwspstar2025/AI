**Day 5：企业级落地（1/2）**
Alright—**Day 5: Enterprise Integration (1/2)**

> 你选择了 **A, B, C, D, E 全部接入**。今天我们把 MCP Server 变成“企业内 AI 能力层”：能**安全**读写 SQL、调用 REST、用 Redis、读写文件，并预留“其它集成（E）”的标准位。
> You picked **A, B, C, D, E**. Today we’ll turn your MCP server into an **Enterprise AI Capability Layer** with **safe** SQL access, REST calls, Redis, file storage, and a plug-in slot for “others (E).”

---

# 🎯 今日目标 · Today’s Goals

* **中文**：为 SQL / REST / Redis / 文件存储 各自提供 **Tools + Resources + Prompts**，并加上 **权限与审计**。
* **EN**: Deliver **Tools + Resources + Prompts** for SQL / REST / Redis / Files, with **security & audit** baked in.

---

# 🗺️ 架构全景 · Architecture at a Glance

```
LLM Client (Claude/ChatGPT/Cursor)
        │ JSON-RPC (stdio/HTTP)
        ▼
  C# MCP Server
  ├─ RpcEngine (统一分发)
  ├─ Policies (RBAC/ABAC + DryRun + AllowList)
  ├─ Audit (结构化审计日志)
  ├─ ToolRegistry / PromptRegistry / ResourceRegistry
  ├─ SqlModule     (A)
  ├─ RestModule    (B)
  ├─ RedisModule   (C)
  ├─ FilesModule   (D)
  └─ ExtModule(E)  (Webhook/Queue/Graph/ServiceBus…)
```

---

# ⚙️ 环境准备 · Environment

* **.NET 8**、**VS Code (C# Dev Kit)**
* 可选：用 **Docker** 起一个 SQL Server 和 Redis 用于本地联调
  Optional Docker (for quick local dev):

```yaml
# docker-compose.yml
services:
  sql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Passw0rd
    ports: ["1433:1433"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
```

---

# 🧩 公共安全基座 · Common Security Foundation

## 1) 权限与策略 · Policy Guard

**中文**：所有“写操作（或敏感读）”必须走策略：角色/资源白名单/字段白名单/DryRun/二次确认码。
**EN**: Any **writes** (or sensitive reads) must pass policy: RBAC/allow-lists/field allow-lists/**dry-run**/**confirm code**.

```csharp
public interface IPolicyEvaluator
{
    // action: "sql.exec", "rest.post", "redis.set", "files.write" ...
    Task<PolicyDecision> EvaluateAsync(string action, IDictionary<string, object?> ctx);
}

public record PolicyDecision(bool Allowed, bool RequiresConfirm, string? ConfirmCode = null, string? Reason = null);

// Example: allow-list + role-based
public class DefaultPolicyEvaluator : IPolicyEvaluator
{
    private readonly HashSet<string> _writeActions = ["sql.exec","rest.post","redis.set","files.write"];
    public Task<PolicyDecision> EvaluateAsync(string action, IDictionary<string, object?> ctx)
    {
        var role = (string?)ctx.GetValueOrDefault("role") ?? "viewer";
        var dryRun = (bool?)ctx.GetValueOrDefault("dryRun") == true;

        if (!_writeActions.Contains(action)) return Task.FromResult(new PolicyDecision(true, false));

        if (role is "admin" or "ops") return Task.FromResult(new PolicyDecision(true, false));

        // Non-admin writes require confirm code (two-step)
        if (!dryRun)
        {
            var code = Random.Shared.Next(100000, 999999).ToString();
            return Task.FromResult(new PolicyDecision(false, true, code, "Need confirm code"));
        }
        return Task.FromResult(new PolicyDecision(true, false)); // dry run allowed
    }
}
```

## 2) 审计日志 · Audit Log

**中文**：一切操作落地 **结构化日志**（时间、调用方、参数摘要、策略结果、是否 DryRun、耗时、数据大小）。
**EN**: Everything is **structured-logged** (time, caller, param digest, policy result, dryRun flag, latency, size).

```csharp
public interface IAuditLogger { Task WriteAsync(AuditEntry entry); }

public record AuditEntry(
    DateTimeOffset Time,
    string Action,
    string Caller,
    string Resource,
    bool DryRun,
    bool Allowed,
    string? PolicyNote,
    long? Affected,
    TimeSpan Duration
);

public class JsonFileAuditLogger : IAuditLogger
{
    private readonly string _path;
    public JsonFileAuditLogger(string path) => _path = path;

    public async Task WriteAsync(AuditEntry e)
    {
        var line = System.Text.Json.JsonSerializer.Serialize(e);
        await File.AppendAllTextAsync(_path, line + Environment.NewLine);
    }
}
```

> **实务建议 / Practical**：日志写到本地 + ELK/Splunk/CloudWatch；敏感字段打码（masking）。

---

# 🅰️ SQL Server 模块（A）

# SQL Server Module (A)

## 用例 · Use-Cases

* **读**：安全查询/分页（只读白名单 View）
* **写**：受控执行（UPDATE/INSERT/DELETE），仅管理员或二次确认
* **DDL**：默认禁止（或仅限专用维护 Tool）

## 连接配置 · Connection

`appsettings.json`

```json
{
  "Sql": {
    "ConnectionString": "Server=localhost,1433;User ID=sa;Password=YourStrong!Passw0rd;TrustServerCertificate=True;",
    "ReadOnlySchemas": [ "dbo", "report" ],
    "WritableTables": [ "dbo.Orders", "dbo.Inventory" ]
  }
}
```

## 查询工具（只读）· Read Tool

```csharp
public class SqlQueryTool : ITool
{
    public string Name => "sql.query";
    public string Description => "Run a safe, read-only SQL query with optional parameters and paging.";
    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("sql", "string", true, "SELECT only"),
        new ToolParameter("parameters", "object", false, "Named parameters"),
        new ToolParameter("page", "number", false, "1-based page index"),
        new ToolParameter("size", "number", false, "page size (<=200)")
    };

    private readonly string _conn;
    private readonly IPolicyEvaluator _policy;
    private readonly IAuditLogger _audit;

    public SqlQueryTool(IConfiguration cfg, IPolicyEvaluator policy, IAuditLogger audit)
    {
        _conn = cfg.GetSection("Sql")["ConnectionString"]!;
        _policy = policy; _audit = audit;
    }

    public async Task<object> InvokeAsync(JsonElement? args)
    {
        var sql = args!.Value.GetProperty("sql").GetString()!;
        if (!sql.TrimStart().StartsWith("SELECT", StringComparison.OrdinalIgnoreCase))
            throw new ArgumentException("Only SELECT is allowed in sql.query");

        var page = args.Value.TryGetProperty("page", out var p) ? p.GetInt32() : 1;
        var size = args.Value.TryGetProperty("size", out var s) ? Math.Min(200, s.GetInt32()) : 50;

        var sw = System.Diagnostics.Stopwatch.StartNew();
        var allowed = (await _policy.EvaluateAsync("sql.query", new Dictionary<string, object?>{
            ["role"]="viewer", ["dryRun"]=false
        })).Allowed;

        if (!allowed) throw new UnauthorizedAccessException("Not allowed");

        using var conn = new Microsoft.Data.SqlClient.SqlConnection(_conn);
        await conn.OpenAsync();

        var wrapped = $"WITH src AS ({sql}) SELECT * FROM src ORDER BY 1 OFFSET {(page-1)*size} ROWS FETCH NEXT {size} ROWS ONLY;";
        var cmd = new Microsoft.Data.SqlClient.SqlCommand(wrapped, conn);
        using var rdr = await cmd.ExecuteReaderAsync();

        var table = new List<Dictionary<string, object?>>();
        while (await rdr.ReadAsync())
        {
            var row = new Dictionary<string, object?>();
            for (int i=0;i<rdr.FieldCount;i++) row[rdr.GetName(i)] = rdr.GetValue(i);
            table.Add(row);
        }
        sw.Stop();

        await _audit.WriteAsync(new AuditEntry(DateTimeOffset.UtcNow,"sql.query","mcp","db",false,true,null,table.Count,sw.Elapsed));
        return new { page, size, rows = table, count = table.Count };
    }
}
```

## 写工具（带 DryRun / Confirm）· Exec Tool

```csharp
public class SqlExecTool : ITool
{
    public string Name => "sql.exec";
    public string Description => "Execute parameterized DML (INSERT/UPDATE/DELETE) with dryRun/confirm.";
    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("sql", "string", true, "DML only, parameterized"),
        new ToolParameter("parameters", "object", false, "Named parameters"),
        new ToolParameter("dryRun", "boolean", false, "Default true"),
        new ToolParameter("confirmCode", "string", false, "Required if policy demands")
    };

    private readonly string _conn;
    private readonly IPolicyEvaluator _policy;
    private readonly IAuditLogger _audit;

    public SqlExecTool(IConfiguration cfg, IPolicyEvaluator policy, IAuditLogger audit)
    {
        _conn = cfg.GetSection("Sql")["ConnectionString"]!;
        _policy = policy; _audit = audit;
    }

    public async Task<object> InvokeAsync(JsonElement? args)
    {
        var sql = args!.Value.GetProperty("sql").GetString()!;
        var dryRun = args.Value.TryGetProperty("dryRun", out var d) && d.GetBoolean();

        if (!Regex.IsMatch(sql, @"^\s*(INSERT|UPDATE|DELETE)\b", RegexOptions.IgnoreCase))
            throw new ArgumentException("Only INSERT/UPDATE/DELETE are allowed");

        var ctx = new Dictionary<string, object?> { ["role"]="analyst", ["dryRun"]=dryRun };
        var dec = await _policy.EvaluateAsync("sql.exec", ctx);
        if (!dec.Allowed)
        {
            if (dec.RequiresConfirm)
            {
                var provided = args.Value.TryGetProperty("confirmCode", out var cc) ? cc.GetString() : null;
                if (provided != dec.ConfirmCode) return new { requireConfirm = true, code = dec.ConfirmCode };
            }
            else throw new UnauthorizedAccessException(dec.Reason ?? "Denied");
        }

        if (dryRun) return new { dryRun = true, note = "Policy allowed dry run only" };

        var sw = System.Diagnostics.Stopwatch.StartNew();
        using var conn = new Microsoft.Data.SqlClient.SqlConnection(_conn);
        await conn.OpenAsync();
        using var cmd = new Microsoft.Data.SqlClient.SqlCommand(sql, conn);
        var affected = await cmd.ExecuteNonQueryAsync();
        sw.Stop();

        await _audit.WriteAsync(new AuditEntry(DateTimeOffset.UtcNow,"sql.exec","mcp","db",false,true,"applied",affected,sw.Elapsed));
        return new { affected };
    }
}
```

> **提示 / Tip**：生产中应**强制参数化**、列名白名单、表白名单；可用 AST/正则做更严 DML 校验。

---

# 🅱️ REST 模块（B）

# REST Module (B)

## 用例 · Use-Cases

* 调内部 API（GET/POST/PUT/DELETE）
* 支持 **服务目录**（allow-list baseUrls + endpoints）
* 灰度：默认 **GET 允许，非 GET 需要 confirm/dry-run**

```csharp
public class RestCallTool : ITool
{
    public string Name => "rest.call";
    public string Description => "Call an allow-listed REST endpoint with method, headers, and body.";
    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("base", "string", true, "Base alias, e.g., crm, billing"),
        new ToolParameter("path", "string", true, "Endpoint path"),
        new ToolParameter("method", "string", false, "GET/POST/PUT/DELETE"),
        new ToolParameter("headers", "object", false, "Optional headers"),
        new ToolParameter("body", "object", false, "Optional JSON body"),
        new ToolParameter("dryRun", "boolean", false, "Default false")
    };

    private readonly IDictionary<string,string> _bases;
    private readonly IPolicyEvaluator _policy;
    private readonly IAuditLogger _audit;
    private readonly HttpClient _http = new();

    public RestCallTool(IConfiguration cfg, IPolicyEvaluator policy, IAuditLogger audit)
    {
        _bases = cfg.GetSection("Rest:Bases").Get<Dictionary<string,string>>() ?? new();
        _policy = policy; _audit = audit;
    }

    public async Task<object> InvokeAsync(JsonElement? args)
    {
        var b = args!.Value.GetProperty("base").GetString()!;
        var path = args.Value.GetProperty("path").GetString()!;
        var method = args.Value.TryGetProperty("method", out var m) ? m.GetString()!.ToUpperInvariant() : "GET";
        var dry = args.Value.TryGetProperty("dryRun", out var d) && d.GetBoolean();

        if (!_bases.TryGetValue(b, out var baseUrl)) throw new ArgumentException($"Unknown base alias: {b}");
        if (!Uri.TryCreate(new Uri(baseUrl), path, out var url)) throw new ArgumentException("Invalid path");

        var action = method == "GET" ? "rest.get" : "rest.post";
        var dec = await _policy.EvaluateAsync(action, new Dictionary<string, object?> { ["role"]="analyst", ["dryRun"]=dry });
        if (!dec.Allowed) return new { requireConfirm = dec.RequiresConfirm, code = dec.ConfirmCode };

        if (dry) return new { dryRun = true, url = url.ToString(), method };

        using var req = new HttpRequestMessage(new HttpMethod(method), url);
        if (args.Value.TryGetProperty("headers", out var hdrs) && hdrs.ValueKind == JsonValueKind.Object)
            foreach (var kv in hdrs.EnumerateObject()) req.Headers.TryAddWithoutValidation(kv.Name, kv.Value.GetString());

        if (args.Value.TryGetProperty("body", out var body) && body.ValueKind is JsonValueKind.Object or JsonValueKind.Array)
            req.Content = new StringContent(body.ToString(), System.Text.Encoding.UTF8, "application/json");

        var sw = System.Diagnostics.Stopwatch.StartNew();
        var resp = await _http.SendAsync(req);
        var text = await resp.Content.ReadAsStringAsync();
        sw.Stop();

        await _audit.WriteAsync(new AuditEntry(DateTimeOffset.UtcNow, $"rest.{method.ToLower()}", "mcp", url.Host, false, true, resp.StatusCode.ToString(), text.Length, sw.Elapsed));
        return new { status = (int)resp.StatusCode, body = text };
    }
}
```

`appsettings.json` 里配置允许的基地址：

```json
{
  "Rest": {
    "Bases": {
      "crm": "https://internal.crm.example/",
      "billing": "https://billing.example/api/"
    }
  }
}
```

---

# 🅲 Redis 模块（C）

# Redis Module (C)

## 用例 · Use-Cases

* KV 读写、过期控制、发布订阅、简易队列
* 写操作同样要求 confirm/dry-run

```csharp
public class RedisToolSet
{
    public class RedisGetTool : ITool
    {
        public string Name => "redis.get";
        public string Description => "Get a value by key.";
        public IEnumerable<ToolParameter> Parameters => new[] { new ToolParameter("key","string",true) };

        private readonly StackExchange.Redis.IDatabase _db;
        public RedisGetTool(StackExchange.Redis.IConnectionMultiplexer mux) => _db = mux.GetDatabase();

        public async Task<object> InvokeAsync(JsonElement? args)
        {
            var key = args!.Value.GetProperty("key").GetString()!;
            var val = await _db.StringGetAsync(key);
            return new { key, value = (string?)val };
        }
    }

    public class RedisSetTool : ITool
    {
        public string Name => "redis.set";
        public string Description => "Set key with optional ttl (seconds).";
        public IEnumerable<ToolParameter> Parameters => new[]
        {
            new ToolParameter("key","string",true),
            new ToolParameter("value","string",true),
            new ToolParameter("ttlSec","number",false)
        };

        private readonly StackExchange.Redis.IDatabase _db;
        private readonly IPolicyEvaluator _policy;
        public RedisSetTool(StackExchange.Redis.IConnectionMultiplexer mux, IPolicyEvaluator policy)
        { _db = mux.GetDatabase(); _policy = policy; }

        public async Task<object> InvokeAsync(JsonElement? args)
        {
            var key = args!.Value.GetProperty("key").GetString()!;
            var value = args.Value.GetProperty("value").GetString()!;
            var ttl = args.Value.TryGetProperty("ttlSec", out var t) ? TimeSpan.FromSeconds(t.GetInt32()) : (TimeSpan?)null;

            var dec = await _policy.EvaluateAsync("redis.set", new Dictionary<string, object?>{ ["role"]="analyst", ["dryRun"]=false });
            if (!dec.Allowed) return new { requireConfirm = dec.RequiresConfirm, code = dec.ConfirmCode };

            var ok = await _db.StringSetAsync(key, value, ttl);
            return new { ok };
        }
    }
}
```

> 订阅/发布（Pub/Sub）可在 Day 6 实现为**流式事件**：把 Redis 频道消息转发为 MCP 的事件流。

---

# 🅳 文件存储模块（D）

# Files Module (D)

## 用例 · Use-Cases

* 列目录、读文件、写文件（带扩展名白名单与最大尺寸限制）
* 可切换后端：**本地目录** 或 **S3/Blob**（今天先做本地，Day 6 再扩展对象存储）

```csharp
public class FileResourceProvider : IResourceProvider
{
    private readonly string _root;
    private readonly HashSet<string> _allowExt = [".txt",".json",".md",".csv"];
    private const int MaxWriteBytes = 1_000_000; // 1MB

    public FileResourceProvider(string rootDir) => _root = rootDir;
    public string Name => "files";

    public IEnumerable<string> List() =>
        Directory.Exists(_root) ? Directory.GetFiles(_root).Select(Path.GetFileName)! : Enumerable.Empty<string>();

    public object Read(string id)
    {
        var path = Path.Combine(_root, id);
        if (!File.Exists(path)) throw new FileNotFoundException(id);
        return new { id, content = File.ReadAllText(path) };
    }

    public object Info(string id)
    {
        var fi = new FileInfo(Path.Combine(_root, id));
        return new { id, size = fi.Length, modified = fi.LastWriteTimeUtc };
    }

    public object Write(string id, string content)
    {
        var ext = Path.GetExtension(id).ToLowerInvariant();
        if (!_allowExt.Contains(ext)) throw new InvalidOperationException("Extension not allowed");
        var bytes = System.Text.Encoding.UTF8.GetByteCount(content);
        if (bytes > MaxWriteBytes) throw new InvalidOperationException("File too large");
        File.WriteAllText(Path.Combine(_root, id), content);
        return new { ok = true, id, bytes };
    }
}
```

**写文件 Tool（带策略）· File Write Tool**

```csharp
public class FileWriteTool : ITool
{
    public string Name => "files.write";
    public string Description => "Write a text file under managed root.";
    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("id","string",true,"filename with extension"),
        new ToolParameter("content","string",true),
        new ToolParameter("dryRun","boolean",false)
    };

    private readonly FileResourceProvider _provider;
    private readonly IPolicyEvaluator _policy;

    public FileWriteTool(FileResourceProvider provider, IPolicyEvaluator policy)
    { _provider = provider; _policy = policy; }

    public async Task<object> InvokeAsync(JsonElement? args)
    {
        var id = args!.Value.GetProperty("id").GetString()!;
        var content = args.Value.GetProperty("content").GetString()!;
        var dry = args.Value.TryGetProperty("dryRun", out var d) && d.GetBoolean();

        var dec = await _policy.EvaluateAsync("files.write", new Dictionary<string, object?>{ ["role"]="analyst", ["dryRun"]=dry });
        if (!dec.Allowed) return new { requireConfirm = dec.RequiresConfirm, code = dec.ConfirmCode };
        if (dry) return new { dryRun = true, id, bytes = content.Length };

        return _provider.Write(id, content);
    }
}
```

---

# 🅴 扩展模块（E）— 模板位

# Extension Slot (E) — Template

**你可以把 E 用于**：Webhook、消息队列（Azure Service Bus / RabbitMQ / Kafka）、Graph API、Jira/ServiceNow、邮件、CI/CD。
**用途**：把“危险操作”做成**异步命令**（写入队列，人工审批后执行）。

**示例：Webhook 触发器（只记录命令，真正执行由下游服务完成）**

```csharp
public class WebhookTriggerTool : ITool
{
    public string Name => "ext.webhook.trigger";
    public string Description => "Emit a signed command to internal webhook bus.";
    public IEnumerable<ToolParameter> Parameters => new[]
    {
        new ToolParameter("topic","string",true),
        new ToolParameter("payload","object",true),
        new ToolParameter("dryRun","boolean",false)
    };

    private readonly HttpClient _http = new();
    private readonly IPolicyEvaluator _policy;
    private readonly string _endpoint;
    private readonly string _secret;

    public WebhookTriggerTool(IConfiguration cfg, IPolicyEvaluator policy)
    {
        _policy = policy;
        _endpoint = cfg["Ext:Webhook:Endpoint"]!;
        _secret = cfg["Ext:Webhook:Secret"]!;
    }

    public async Task<object> InvokeAsync(JsonElement? args)
    {
        var dry = args!.Value.TryGetProperty("dryRun", out var d) && d.GetBoolean();
        var dec = await _policy.EvaluateAsync("ext.webhook", new Dictionary<string, object?>{ ["role"]="analyst", ["dryRun"]=dry });
        if (!dec.Allowed) return new { requireConfirm = dec.RequiresConfirm, code = dec.ConfirmCode };
        if (dry) return new { dryRun = true };

        var topic = args.Value.GetProperty("topic").GetString()!;
        var payload = args.Value.GetProperty("payload").ToString();
        var body = new StringContent(payload, System.Text.Encoding.UTF8, "application/json");
        var req = new HttpRequestMessage(HttpMethod.Post, $"{_endpoint.TrimEnd('/')}/{topic}");
        req.Content = body;
        req.Headers.Add("X-Signature", _secret); // 简化签名示例
        var resp = await _http.SendAsync(req);
        return new { status = (int)resp.StatusCode };
    }
}
```

---

# 🧱 在 `Program.cs` 注册模块 · Wire-Up in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
var services = builder.Services;
var cfg = builder.Configuration;

// Infra
services.AddSingleton<IPolicyEvaluator, DefaultPolicyEvaluator>();
services.AddSingleton<IAuditLogger>(_ => new JsonFileAuditLogger("audit.log"));

// SQL
services.AddSingleton<ITool, SqlQueryTool>();
services.AddSingleton<ITool, SqlExecTool>();

// REST
services.AddSingleton<ITool, RestCallTool>();

// Redis
var mux = await StackExchange.Redis.ConnectionMultiplexer.ConnectAsync("localhost:6379");
services.AddSingleton(mux);
services.AddSingleton<ITool, RedisToolSet.RedisGetTool>();
services.AddSingleton<ITool, RedisToolSet.RedisSetTool>();

// Files
var filesProvider = new FileResourceProvider("./data");
Directory.CreateDirectory("./data");
services.AddSingleton(filesProvider);
services.AddSingleton<IResourceProvider>(filesProvider);
services.AddSingleton<ITool, FileWriteTool>();

// Ext
services.AddSingleton<ITool, WebhookTriggerTool>();

// MCP Core
services.AddSingleton<ToolRegistry>(sp =>
{
    var reg = new ToolRegistry();
    foreach (var tool in sp.GetServices<ITool>()) reg.Register(tool);
    return reg;
});
services.AddSingleton<ResourceRegistry>(sp =>
{
    var reg = new ResourceRegistry();
    foreach (var res in sp.GetServices<IResourceProvider>()) reg.Register(res);
    return reg;
});
services.AddSingleton<PromptRegistry>(sp =>
{
    var reg = new PromptRegistry();
    reg.Register(new SummarizePrompt());
    return reg;
});
services.AddSingleton<RpcEngine>();

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

> 如果你使用 **Console（stdio）版**，把相同的服务注册逻辑放到 **Host** 里，然后把 `RpcEngine` 注入给 `JsonRpcServer` 即可。
> For the **Console version**, register the same services into a Host and inject `RpcEngine` into `JsonRpcServer`.

---

# 🧪 快速验收 · Quick Validation

**示例 1：列出资源（文件） / List resources (files)**

```json
{"id":"1","method":"resources/list","params":{}}
```

**示例 2：读文件 / Read file**

```json
{"id":"2","method":"resources/read","params":{"id":"readme.md"}}
```

**示例 3：写文件（DryRun） / Write file (dryRun)**

```json
{"id":"3","method":"tools/call","params":{"name":"files.write","args":{"id":"note.txt","content":"hello","dryRun":true}}}
```

**示例 4：SQL 查询 / SQL SELECT**

```json
{"id":"4","method":"tools/call","params":{"name":"sql.query","args":{"sql":"SELECT 1 AS A"}}}
```

**示例 5：SQL 写（需要确认码） / SQL DML with confirm**

```json
{"id":"5","method":"tools/call","params":{"name":"sql.exec","args":{"sql":"UPDATE dbo.Inventory SET Qty=Qty+1 WHERE Id=42"}}}
```

若返回 `requireConfirm=true`，将 `code` 回填到 `confirmCode` 再发送一次。
If it returns `requireConfirm`, re-send with `confirmCode`.

**示例 6：REST GET**

```json
{"id":"6","method":"tools/call","params":{"name":"rest.call","args":{"base":"crm","path":"api/customers?top=5","method":"GET"}}}
```

**示例 7：Redis GET/SET**

```json
{"id":"7","method":"tools/call","params":{"name":"redis.set","args":{"key":"greet","value":"hi","ttlSec":30}}}
{"id":"8","method":"tools/call","params":{"name":"redis.get","args":{"key":"greet"}}}
```

**示例 8：扩展 E（Webhook）**

```json
{"id":"9","method":"tools/call","params":{"name":"ext.webhook.trigger","args":{"topic":"deploy","payload":{"app":"mcp-api","env":"staging"}}}}
```

---

# 🔐 生产级安全清单 · Security Checklist (Prod)

* **最小权限账号**：SQL 用只读/只写分离账号，REST 用细粒度 token；Redis 需要 ACL。
* **写操作默认 DryRun**，非管理员需二次确认（带过期时间）。
* **强制参数化**、表/列白名单、最大行数/超时限制。
* **审计不可篡改**：落地后推到集中日志；敏感字段脱敏。
* **密钥管理**：不要进代码，放到 KeyVault / AWS Secrets / Azure Key Vault。
* **速率限制 / 节流**：防止 LLM 误触发高频写。
* **沙箱**：先连到影子库/影子环境，灰度到生产。

---

# ✅ 你今天得到的东西 · What You Got Today

* 可即用的 **A/B/C/D/E 模块化接入**（C#）
* **统一策略 + 审计** 基座
* **Console + HTTP** 双形态复用

---

# ⏭️ Day 6 预告 · Next (Ops/Platform)

* CI/CD（GitHub Actions / Azure DevOps）
* 配置分层（dev/staging/prod）与机密管理
* 监控/报警（Health endpoints、OpenTelemetry）
* **事件流**（Redis Pub/Sub → MCP 事件）
* S3/Blob 对象存储版 Files Provider

---

如果你想要，我可以把 **Day 1~5 的整合代码**打包给你（含注释与示例配置）。
If you want, I can **package Day 1–5 code** (with comments & sample configs).
直接回一句：**“打包 Day1–5 代码”** 即可。
