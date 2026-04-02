# .NET Code Review Checklist

> **Last updated:** 2026-04-02

A practical checklist for reviewing .NET / C# codebases. Each rule includes severity, grep patterns for scanning, and bad/good code examples.

---

## Table of Contents

1. [Dispose All Streams (IDisposable)](#1-dispose-all-streams-idisposable)
2. [HttpClient Must Be Singleton or Static](#2-httpclient-must-be-singleton-or-static)
3. [Async/Await — No .Result or .Wait()](#3-asyncawait--no-result-or-wait)
4. [SerialPort Must Set ReadTimeout and WriteTimeout](#4-serialport-must-set-readtimeout-and-writetimeout)
5. [Use SqlParameter — No String Concatenation](#5-use-sqlparameter--no-string-concatenation)
6. [Dispose SqlConnection, SqlCommand, SqlDataReader, SqlTransaction](#6-dispose-sqlconnection-sqlcommand-sqldatareader-sqltransaction)
7. [Follow Microsoft C# Code Conventions](#7-follow-microsoft-c-code-conventions)
8. [Visual Studio .gitignore](#8-visual-studio-gitignore)
9. [Singleton + Lazy Pattern for Redis, CosmosDB, Service Bus](#9-singleton--lazy-pattern-for-redis-cosmosdb-service-bus)
10. [Use Serilog Instead of Custom Logging](#10-use-serilog-instead-of-custom-logging)
11. [Beware of Static Collections (Memory Leak Risk)](#11-beware-of-static-collections-memory-leak-risk)
12. [Unsubscribe Event Handlers and Dispose Timers](#12-unsubscribe-event-handlers-and-dispose-timers)
13. [Never Hardcode Sensitive Data](#13-never-hardcode-sensitive-data)
14. [Use StringBuilder for String Concatenation in Loops](#14-use-stringbuilder-for-string-concatenation-in-loops)
15. [Use Connection Strings from Configuration](#15-use-connection-strings-from-configuration)
16. [Avoid async void — Use async Task](#16-avoid-async-void--use-async-task)
17. [Use throw; Not throw ex;](#17-use-throw-not-throw-ex)
18. [Use Concurrent Collections for Shared State](#18-use-concurrent-collections-for-shared-state)
19. [Avoid Task.Run() in ASP.NET Request Handlers](#19-avoid-taskrun-in-aspnet-request-handlers)
20. [Avoid the dynamic Keyword](#20-avoid-the-dynamic-keyword)

---

## 1. Dispose All Streams (IDisposable)

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Resource Management | `new FileStream`, `new MemoryStream`, `new StreamReader`, `new StreamWriter`, `new NetworkStream`, `new BufferedStream` |

**Why it matters:** Streams hold unmanaged resources (file handles, sockets). Failing to dispose them causes file locks, handle exhaustion, and memory pressure in long-running services.

### Bad Example

```csharp
public byte[] ReadFile(string path)
{
    var stream = new FileStream(path, FileMode.Open);
    var reader = new StreamReader(stream);
    var content = reader.ReadToEnd();
    return Encoding.UTF8.GetBytes(content);
    // stream and reader are never disposed — file handle leaked
}
```

### Good Example

```csharp
public byte[] ReadFile(string path)
{
    using var stream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(stream);
    var content = reader.ReadToEnd();
    return Encoding.UTF8.GetBytes(content);
}
```

**Reviewer tip:** Look for any `new *Stream*` or `new StreamReader/Writer` that is **not** preceded by `using`.

---

## 2. HttpClient Must Be Singleton or Static

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Resource Management / Networking | `new HttpClient()`, `new HttpClient(` |

**Why it matters:** Each `HttpClient` instance opens its own connection pool. Creating and disposing them per request leads to **socket exhaustion** (`TIME_WAIT` state) and can bring down the application under load.

### Bad Example

```csharp
public async Task<string> GetDataAsync(string url)
{
    using var client = new HttpClient(); // new instance per call — socket exhaustion
    var response = await client.GetAsync(url);
    return await response.Content.ReadAsStringAsync();
}
```

### Good Example — Static Instance

```csharp
public class ApiService
{
    private static readonly HttpClient _httpClient = new HttpClient();

    public async Task<string> GetDataAsync(string url)
    {
        var response = await _httpClient.GetAsync(url);
        return await response.Content.ReadAsStringAsync();
    }
}
```

### Good Example — IHttpClientFactory (Recommended for DI)

```csharp
// In Startup / Program.cs
builder.Services.AddHttpClient();

// In your service
public class ApiService
{
    private readonly IHttpClientFactory _httpClientFactory;

    public ApiService(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public async Task<string> GetDataAsync(string url)
    {
        var client = _httpClientFactory.CreateClient();
        var response = await client.GetAsync(url);
        return await response.Content.ReadAsStringAsync();
    }
}
```

**Reviewer tip:** Search for `new HttpClient()`. It should appear **at most once** as a static/singleton field, or not at all if `IHttpClientFactory` is used.

---

## 3. Async/Await — No .Result or .Wait()

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Critical** | Concurrency | `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` |

**Why it matters:** Calling `.Result` or `.Wait()` on a `Task` **blocks the calling thread** and can cause a **deadlock** in environments with a synchronization context (ASP.NET, WinForms, WPF). The blocked thread holds the sync context, while the awaited task needs it to complete — resulting in a permanent hang.

### Bad Example

```csharp
public string GetData()
{
    // DEADLOCK: blocks the thread while waiting for the async result
    var result = GetDataAsync().Result;
    return result;
}

public void SendNotification()
{
    // DEADLOCK: same issue with .Wait()
    SendNotificationAsync().Wait();
}
```

### Good Example

```csharp
public async Task<string> GetDataAsync()
{
    var result = await _httpClient.GetStringAsync("https://api.example.com/data");
    return result;
}

public async Task SendNotificationAsync()
{
    await _notificationService.SendAsync(message);
}
```

> **Note:** If you absolutely must call async from sync code (e.g., `Main` in legacy console apps), use `GetAwaiter().GetResult()` as a last resort — but document why. The proper fix is to make the entire call chain `async`.

**Reviewer tip:** Search for `.Result`, `.Wait()`, and `.GetAwaiter().GetResult()`. Each occurrence needs justification or refactoring to `await`.

---

## 4. SerialPort Must Set ReadTimeout and WriteTimeout

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Hardware / IO | `new SerialPort`, `SerialPort` |

**Why it matters:** The default timeout for `SerialPort` is `InfiniteTimeout` (-1). If the connected device does not respond, `Read` or `Write` will **block the thread forever**, freezing the application.

### Bad Example

```csharp
var port = new SerialPort("COM3", 9600);
port.Open();
string data = port.ReadLine(); // blocks forever if device does not respond
```

### Good Example

```csharp
var port = new SerialPort("COM3", 9600)
{
    ReadTimeout = 3000,  // 3 seconds
    WriteTimeout = 3000  // 3 seconds
};

port.Open();

try
{
    string data = port.ReadLine();
}
catch (TimeoutException)
{
    // handle timeout — log, retry, or alert
}
finally
{
    port.Close();
}
```

**Reviewer tip:** Every `new SerialPort` should have `ReadTimeout` and `WriteTimeout` set before or immediately after opening.

---

## 5. Use SqlParameter — No String Concatenation

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Critical** | Security (SQL Injection) | `"SELECT " +`, `"INSERT " +`, `"UPDATE " +`, `"DELETE " +`, `$"SELECT`, `$"INSERT`, `$"UPDATE`, `$"DELETE`, `string.Format("SELECT` |

**Why it matters:** String concatenation in SQL queries enables **SQL injection** — an attacker can manipulate input to execute arbitrary SQL, potentially compromising the entire database. This is [OWASP Top 10 #3 (Injection)](https://owasp.org/Top10/).

### Bad Example

```csharp
public User GetUser(string userId)
{
    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand();
    command.Connection = connection;

    // SQL INJECTION: user input directly concatenated
    command.CommandText = "SELECT * FROM Users WHERE Id = '" + userId + "'";

    connection.Open();
    using var reader = command.ExecuteReader();
    // ...
}
```

### Good Example — SqlParameter

```csharp
public User GetUser(string userId)
{
    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand("SELECT * FROM Users WHERE Id = @Id", connection);
    command.Parameters.Add(new SqlParameter("@Id", SqlDbType.NVarChar) { Value = userId });

    connection.Open();
    using var reader = command.ExecuteReader();
    // ...
}
```

### Good Example — Dapper (Parameterized)

```csharp
public User GetUser(string userId)
{
    using var connection = new SqlConnection(connectionString);
    return connection.QueryFirstOrDefault<User>(
        "SELECT * FROM Users WHERE Id = @Id",
        new { Id = userId });
}
```

**Reviewer tip:** Search for string concatenation or interpolation near SQL keywords (`SELECT`, `INSERT`, `UPDATE`, `DELETE`). Every query parameter must use `SqlParameter` or an ORM's parameterized API.

---

## 6. Dispose SqlConnection, SqlCommand, SqlDataReader, SqlTransaction

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Resource Management / Database | `new SqlConnection`, `new SqlCommand`, `ExecuteReader`, `BeginTransaction` |

**Why it matters:** Undisposed `SqlConnection` objects are **not returned to the connection pool**, leading to **connection pool exhaustion**. Undisposed `SqlTransaction` objects can hold database locks indefinitely.

### Bad Example

```csharp
public List<Order> GetOrders()
{
    var connection = new SqlConnection(connectionString);
    var command = new SqlCommand("SELECT * FROM Orders", connection);

    connection.Open();
    var reader = command.ExecuteReader();

    var orders = new List<Order>();
    while (reader.Read())
    {
        orders.Add(MapOrder(reader));
    }

    // connection, command, and reader are never disposed
    // connection is NOT returned to the pool
    return orders;
}
```

### Good Example

```csharp
public List<Order> GetOrders()
{
    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand("SELECT * FROM Orders", connection);

    connection.Open();
    using var reader = command.ExecuteReader();

    var orders = new List<Order>();
    while (reader.Read())
    {
        orders.Add(MapOrder(reader));
    }

    return orders;
}
```

### Good Example — With Transaction

```csharp
public void TransferFunds(int fromId, int toId, decimal amount)
{
    using var connection = new SqlConnection(connectionString);
    connection.Open();

    using var transaction = connection.BeginTransaction();
    try
    {
        using var debitCmd = new SqlCommand(
            "UPDATE Accounts SET Balance = Balance - @Amount WHERE Id = @Id", connection, transaction);
        debitCmd.Parameters.AddWithValue("@Amount", amount);
        debitCmd.Parameters.AddWithValue("@Id", fromId);
        debitCmd.ExecuteNonQuery();

        using var creditCmd = new SqlCommand(
            "UPDATE Accounts SET Balance = Balance + @Amount WHERE Id = @Id", connection, transaction);
        creditCmd.Parameters.AddWithValue("@Amount", amount);
        creditCmd.Parameters.AddWithValue("@Id", toId);
        creditCmd.ExecuteNonQuery();

        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

**Reviewer tip:** Every `new SqlConnection`, `new SqlCommand`, `ExecuteReader()`, and `BeginTransaction()` must be wrapped in a `using` statement.

---

## 7. Follow Microsoft C# Code Conventions

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Code Style | N/A — use tooling |

**Why it matters:** Consistent code style reduces cognitive load during reviews and makes the codebase easier to maintain. Automated enforcement prevents style debates in PRs.

### Key Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Public members | PascalCase | `public void ProcessOrder()` |
| Private fields | `_camelCase` with underscore prefix | `private readonly int _retryCount;` |
| Local variables | camelCase | `var orderCount = 0;` |
| Constants | PascalCase | `public const int MaxRetries = 3;` |
| Interfaces | `I` prefix + PascalCase | `public interface IOrderService` |
| Braces | New line (Allman style) | See below |
| `var` usage | Use when type is apparent | `var orders = new List<Order>();` |
| File-scoped namespaces | Prefer in C# 10+ | `namespace MyApp.Services;` |

### Enforce with .editorconfig

Add an `.editorconfig` to the repo root to enforce conventions automatically:

```ini
[*.cs]
# Naming
dotnet_naming_rule.private_fields_should_be_camel_case.severity = warning
dotnet_naming_rule.private_fields_should_be_camel_case.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case.style = camel_case_underscore

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.camel_case_underscore.required_prefix = _
dotnet_naming_style.camel_case_underscore.capitalization = camel_case

# Formatting
csharp_new_line_before_open_brace = all
csharp_prefer_braces = true:warning
csharp_style_namespace_declarations = file_scoped:suggestion

# var preferences
csharp_style_var_when_type_is_apparent = true:suggestion
```

Run `dotnet format` to auto-fix style violations before committing.

**Reviewer tip:** Ensure the project has an `.editorconfig`. Run `dotnet format --verify-no-changes` in CI to catch violations automatically.

> **Reference:** [Microsoft C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)

---

## 8. Visual Studio .gitignore

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **N/A** | Configuration | Check for `.gitignore` in repo root |

**Status: Done** — This repository already includes the standard [VisualStudio.gitignore](https://github.com/github/gitignore/blob/main/VisualStudio.gitignore) template.

Every .NET repository should start with this template to avoid committing:
- Build outputs (`bin/`, `obj/`)
- User-specific files (`.suo`, `.user`, `.vs/`)
- NuGet packages
- Test results and logs

**Reviewer tip:** If a new repo is missing a `.gitignore`, copy the standard template from GitHub's [gitignore repository](https://github.com/github/gitignore/blob/main/VisualStudio.gitignore).

---

## 9. Singleton + Lazy Pattern for Redis, CosmosDB, Service Bus

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Resource Management / Connectivity | `ConnectionMultiplexer.Connect`, `new ConnectionMultiplexer`, `new CosmosClient`, `new ServiceBusClient`, `new ServiceBusSender` |

**Why it matters:** These clients manage their own connection pools internally and are designed to be **long-lived singletons**. Creating new instances per request causes connection storms, exceeds connection limits, and degrades performance. `Lazy<T>` ensures **thread-safe, one-time initialization**.

### Bad Example — Redis

```csharp
public class CacheService
{
    public string GetValue(string key)
    {
        // new connection per call — connection storm, exhausts server connections
        var redis = ConnectionMultiplexer.Connect("localhost:6379");
        var db = redis.GetDatabase();
        return db.StringGet(key);
    }
}
```

### Good Example — Redis with Lazy Singleton

```csharp
public class CacheService
{
    private static readonly Lazy<ConnectionMultiplexer> _lazyConnection =
        new Lazy<ConnectionMultiplexer>(() =>
            ConnectionMultiplexer.Connect("localhost:6379"));

    public static ConnectionMultiplexer Connection => _lazyConnection.Value;

    public string GetValue(string key)
    {
        var db = Connection.GetDatabase();
        return db.StringGet(key);
    }
}
```

### Good Example — DI Registration (CosmosDB / Service Bus)

```csharp
// Program.cs — register as singleton
builder.Services.AddSingleton(_ => new CosmosClient(connectionString));
builder.Services.AddSingleton(_ => new ServiceBusClient(connectionString));

// In your service — inject via constructor
public class OrderRepository
{
    private readonly CosmosClient _cosmosClient;

    public OrderRepository(CosmosClient cosmosClient)
    {
        _cosmosClient = cosmosClient;
    }
}
```

> **Same pattern applies to:** `CosmosClient`, `ServiceBusClient`, `ServiceBusSender`, `ServiceBusProcessor`, and similar long-lived connection-based clients.

**Reviewer tip:** Search for `ConnectionMultiplexer.Connect`, `new CosmosClient`, `new ServiceBusClient`. They should appear only in a singleton registration or `Lazy<T>` initializer — never inside a method called per request.

---

## 10. Use Serilog Instead of Custom Logging

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Observability | `Console.WriteLine` (as logging), custom `Logger` or `LogHelper` classes, `Trace.Write`, `File.AppendAllText` (for logging) |

**Why it matters:** Hand-rolled logging is error-prone, lacks structured data, and doesn't integrate with modern observability platforms. **Serilog** provides structured logging, a rich sink ecosystem (Seq, Elasticsearch, Application Insights, File), and high performance out of the box.

### Bad Example — Custom Logging

```csharp
public class OrderService
{
    public void ProcessOrder(int orderId)
    {
        File.AppendAllText("logs.txt",
            $"{DateTime.Now} - Processing order {orderId}\n"); // no structure, no levels, poor performance

        try
        {
            // ... process order
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error: {ex.Message}"); // lost on restart, not searchable
        }
    }
}
```

### Good Example — Serilog

```csharp
// Program.cs — setup
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .WriteTo.Console()
    .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog();
```

```csharp
// In your service — use structured logging
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public void ProcessOrder(int orderId)
    {
        _logger.LogInformation("Processing order {OrderId}", orderId); // structured, searchable

        try
        {
            // ... process order
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process order {OrderId}", orderId);
        }
    }
}
```

> **Key benefit:** Message templates like `"Processing order {OrderId}"` produce **structured log events** where `OrderId` is a searchable, filterable property — not just embedded text.

**Reviewer tip:** Search for `Console.WriteLine`, `File.AppendAllText`, or custom `Logger`/`LogHelper` classes used for logging. These should be replaced with `ILogger<T>` backed by Serilog.

---

## 11. Beware of Static Collections (Memory Leak Risk)

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Memory Management | `static List<`, `static Dictionary<`, `static ArrayList`, `static IEnumerable`, `static ConcurrentDictionary<`, `static HashSet<`, `static IList<`, `static IDictionary<` |

**Why it matters:** Static variables live for the **entire lifetime of the process**. A static collection that grows (items added but never removed) causes a **memory leak** that eventually leads to `OutOfMemoryException`. This is especially dangerous in ASP.NET applications that run for days or weeks.

### Bad Example

```csharp
public class SessionTracker
{
    // grows forever — never cleaned up
    private static readonly List<UserSession> _sessions = new List<UserSession>();

    public void TrackSession(UserSession session)
    {
        _sessions.Add(session); // items are added but never removed
    }
}
```

```csharp
public class DataCache
{
    // unbounded dictionary — every unique key stays forever
    private static readonly Dictionary<string, object> _cache = new Dictionary<string, object>();

    public void Set(string key, object value)
    {
        _cache[key] = value; // entries are never evicted
    }
}
```

### Good Example — Use IMemoryCache with Expiration

```csharp
// Program.cs
builder.Services.AddMemoryCache();

// In your service
public class DataCache
{
    private readonly IMemoryCache _cache;

    public DataCache(IMemoryCache cache)
    {
        _cache = cache;
    }

    public void Set(string key, object value)
    {
        var options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration = TimeSpan.FromMinutes(10)
        };

        _cache.Set(key, value, options);
    }
}
```

### Good Example — Bounded Collection with Cleanup

```csharp
public class SessionTracker
{
    private static readonly ConcurrentDictionary<string, UserSession> _sessions = new();
    private static readonly Timer _cleanupTimer = new Timer(CleanupExpired, null,
        TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(5));

    public void TrackSession(UserSession session)
    {
        _sessions[session.Id] = session;
    }

    private static void CleanupExpired(object state)
    {
        var expired = _sessions.Where(s => s.Value.ExpiresAt < DateTime.UtcNow).ToList();
        foreach (var session in expired)
        {
            _sessions.TryRemove(session.Key, out _);
        }
    }
}
```

**Reviewer tip:** Search for `static List<`, `static Dictionary<`, `static ArrayList`, etc. For each hit, verify: (1) Is there a removal/eviction mechanism? (2) Is the growth bounded? If not, flag it.

---

## 12. Unsubscribe Event Handlers and Dispose Timers

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Memory Management | `+=`, `new Timer(`, `new System.Timers.Timer`, `new System.Threading.Timer` |

**Why it matters:** Event subscriptions (`+=`) create strong references from the event source to the subscriber. If the subscriber is never unsubscribed (`-=`), the garbage collector cannot collect it — even if no other references exist. Similarly, undisposed `Timer` objects continue firing and hold references to callback targets, causing memory leaks in long-running services.

### Bad Example — Event Handler Leak

```csharp
public class OrderMonitor
{
    public void Start(OrderService orderService)
    {
        orderService.OrderReceived += OnOrderReceived; // subscribed, never unsubscribed
    }

    private void OnOrderReceived(object sender, OrderEventArgs e)
    {
        // handle event
    }

    // OrderMonitor can never be garbage collected while OrderService is alive
}
```

### Good Example — Unsubscribe and Implement IDisposable

```csharp
public class OrderMonitor : IDisposable
{
    private readonly OrderService _orderService;

    public OrderMonitor(OrderService orderService)
    {
        _orderService = orderService;
        _orderService.OrderReceived += OnOrderReceived;
    }

    private void OnOrderReceived(object sender, OrderEventArgs e)
    {
        // handle event
    }

    public void Dispose()
    {
        _orderService.OrderReceived -= OnOrderReceived;
    }
}
```

### Bad Example — Timer Never Disposed

```csharp
public class PollingService
{
    public void Start()
    {
        var timer = new System.Threading.Timer(Poll, null, 0, 5000);
        // timer is never stored or disposed — runs forever, holds reference to Poll
    }

    private void Poll(object state) { /* ... */ }
}
```

### Good Example — Timer Disposed

```csharp
public class PollingService : IDisposable
{
    private readonly Timer _timer;

    public PollingService()
    {
        _timer = new Timer(Poll, null, 0, 5000);
    }

    private void Poll(object state) { /* ... */ }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

**Reviewer tip:** Search for `+=` on event subscriptions — verify there is a corresponding `-=`. Search for `new Timer(` — verify the timer is stored in a field and disposed.

---

## 13. Never Hardcode Sensitive Data

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Critical** | Security | `"Server=`, `"Data Source=`, `"Password=`, `"pwd=`, `"ApiKey"`, `"Bearer "`, `"sk-`, `"mongodb://`, `"redis://` |

**Why it matters:** Hardcoded credentials, connection strings, and API keys in source code get committed to version control and are **exposed to anyone with repo access**. Leaked secrets can lead to data breaches, unauthorized access, and compliance violations.

### Bad Example

```csharp
public class DatabaseService
{
    // credentials committed to source control
    private readonly string _connectionString =
        "Server=prod-db.example.com;Database=AppDb;User Id=admin;Password=S3cret!;";

    private readonly string _apiKey = "sk-abc123secret456";
}
```

### Good Example — Configuration + User Secrets / Key Vault

```csharp
// appsettings.json (no secrets here — only non-sensitive config)
// {
//   "ConnectionStrings": { "AppDb": "" },
//   "ApiSettings": { "BaseUrl": "https://api.example.com" }
// }

// For local development: dotnet user-secrets set "ConnectionStrings:AppDb" "Server=..."
// For production: Azure Key Vault, AWS Secrets Manager, or environment variables

public class DatabaseService
{
    private readonly string _connectionString;

    public DatabaseService(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("AppDb")
            ?? throw new InvalidOperationException("AppDb connection string not configured.");
    }
}
```

```csharp
// Program.cs — Azure Key Vault integration
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myapp-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

**Reviewer tip:** Search for connection string patterns (`Server=`, `Password=`, `pwd=`), API key prefixes (`sk-`, `Bearer `), and inline URIs with credentials. These must come from configuration, not source code.

---

## 14. Use StringBuilder for String Concatenation in Loops

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Performance | `+= ` inside loops with string variables, `string.Concat` in loops |

**Why it matters:** Strings in C# are **immutable**. Each `+=` creates a new string object on the heap, copying all previous content. In a loop with N iterations, this results in O(N^2) memory allocations and copies. `StringBuilder` modifies an internal buffer in-place, giving O(N) performance.

### Bad Example

```csharp
public string BuildReport(List<Order> orders)
{
    string report = "";
    foreach (var order in orders)
    {
        report += $"Order {order.Id}: {order.Total}\n"; // new string allocated every iteration
    }
    return report;
}
// 10,000 orders = ~10,000 intermediate string allocations
```

### Good Example

```csharp
public string BuildReport(List<Order> orders)
{
    var sb = new StringBuilder();
    foreach (var order in orders)
    {
        sb.AppendLine($"Order {order.Id}: {order.Total}");
    }
    return sb.ToString();
}
// 10,000 orders = 1 StringBuilder with internal buffer resizing
```

**Reviewer tip:** Look for `+=` on a `string` variable inside `for`, `foreach`, or `while` loops. If the loop may execute more than a few iterations, it should use `StringBuilder`.

---

## 15. Use Connection Strings from Configuration

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Security / Maintainability | `new SqlConnection("`, `ConnectionMultiplexer.Connect("`, `new CosmosClient("` with inline strings |

**Why it matters:** Hardcoded connection strings make it impossible to change environments (dev/staging/prod) without recompiling. They also risk committing credentials to source control (see [Rule 13](#13-never-hardcode-sensitive-data)). Configuration-driven connection strings enable environment-specific overrides and secret management.

### Bad Example

```csharp
public class OrderRepository
{
    public List<Order> GetOrders()
    {
        // hardcoded — can't change per environment, credentials in source
        using var connection = new SqlConnection(
            "Server=192.168.1.100;Database=OrderDb;User Id=sa;Password=P@ssw0rd;");
        connection.Open();
        // ...
    }
}
```

### Good Example

```csharp
// appsettings.json
// {
//   "ConnectionStrings": {
//     "OrderDb": "Server=localhost;Database=OrderDb;Integrated Security=true;"
//   }
// }

// appsettings.Production.json (overrides per environment)
// {
//   "ConnectionStrings": {
//     "OrderDb": "" <-- populated from Key Vault or environment variable
//   }
// }

public class OrderRepository
{
    private readonly string _connectionString;

    public OrderRepository(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("OrderDb")
            ?? throw new InvalidOperationException("OrderDb connection string not configured.");
    }

    public List<Order> GetOrders()
    {
        using var connection = new SqlConnection(_connectionString);
        connection.Open();
        // ...
    }
}
```

**Reviewer tip:** Search for `new SqlConnection("`, `new CosmosClient("`, or `ConnectionMultiplexer.Connect("` with inline string literals. Connection strings should always come from `IConfiguration.GetConnectionString()`.

---

## 16. Avoid async void — Use async Task

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Concurrency / Error Handling | `async void` |

**Why it matters:** Exceptions thrown in `async void` methods **cannot be caught** by the caller — they are raised directly on the `SynchronizationContext` and will **crash the process**. Additionally, the caller has no way to `await` completion, making it impossible to know when the operation finishes. The only valid use of `async void` is for event handlers (e.g., WinForms/WPF button click handlers).

### Bad Example

```csharp
public class NotificationService
{
    // async void — exception crashes the process, caller can't await
    public async void SendNotification(string message)
    {
        await _emailClient.SendAsync(message); // if this throws, app crashes
    }
}

// Caller has no way to await or catch errors
notificationService.SendNotification("Hello");
```

### Good Example

```csharp
public class NotificationService
{
    public async Task SendNotificationAsync(string message)
    {
        await _emailClient.SendAsync(message); // exception propagates normally
    }
}

// Caller can await and handle errors
await notificationService.SendNotificationAsync("Hello");
```

### Exception — Event Handlers (async void is acceptable)

```csharp
// WinForms / WPF — async void is the only option for event handlers
private async void BtnSubmit_Click(object sender, EventArgs e)
{
    try
    {
        await _orderService.SubmitOrderAsync();
    }
    catch (Exception ex)
    {
        MessageBox.Show($"Error: {ex.Message}");
    }
}
```

**Reviewer tip:** Search for `async void`. Every occurrence should be an event handler. If it's a regular method, change the return type to `async Task`.

---

## 17. Use throw; Not throw ex;

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Error Handling / Debugging | `throw ex;` |

**Why it matters:** `throw ex;` **resets the stack trace** to the current line, destroying information about where the exception actually originated. `throw;` preserves the original stack trace, making debugging significantly easier.

### Bad Example

```csharp
try
{
    ProcessOrder(order);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Order processing failed");
    throw ex; // stack trace now points HERE, not where the error actually occurred
}
```

### Good Example — Rethrow with Original Stack Trace

```csharp
try
{
    ProcessOrder(order);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Order processing failed");
    throw; // original stack trace preserved — points to actual error location
}
```

### Good Example — Wrap in a New Exception

```csharp
try
{
    ProcessOrder(order);
}
catch (Exception ex)
{
    // wrapping preserves original as InnerException
    throw new OrderProcessingException("Failed to process order", ex);
}
```

**Reviewer tip:** Search for `throw ex;` — it should almost always be `throw;` or wrapping in a new exception with the original as `InnerException`.

---

## 18. Use Concurrent Collections for Shared State

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **High** | Concurrency / Thread Safety | `static Dictionary<`, `static List<`, `static HashSet<` accessed from multiple threads |

**Why it matters:** `Dictionary<TKey, TValue>`, `List<T>`, and `HashSet<T>` are **not thread-safe**. Concurrent reads and writes corrupt their internal data structures, leading to infinite loops, lost data, `IndexOutOfRangeException`, or silent data corruption. These bugs are intermittent and extremely hard to reproduce.

### Bad Example

```csharp
public class UserCache
{
    // Dictionary is NOT thread-safe — concurrent access corrupts internal state
    private static readonly Dictionary<string, User> _users = new Dictionary<string, User>();

    public void AddUser(User user)
    {
        _users[user.Id] = user; // concurrent writes can corrupt the dictionary
    }

    public User GetUser(string id)
    {
        return _users.TryGetValue(id, out var user) ? user : null;
    }
}
```

### Good Example — ConcurrentDictionary

```csharp
public class UserCache
{
    private static readonly ConcurrentDictionary<string, User> _users = new();

    public void AddUser(User user)
    {
        _users[user.Id] = user; // thread-safe
    }

    public User GetUser(string id)
    {
        return _users.TryGetValue(id, out var user) ? user : null;
    }
}
```

### Other Concurrent Alternatives

| Instead of | Use |
|-----------|-----|
| `Dictionary<K,V>` | `ConcurrentDictionary<K,V>` |
| `List<T>` | `ConcurrentBag<T>` or `ConcurrentQueue<T>` |
| `Queue<T>` | `ConcurrentQueue<T>` |
| `Stack<T>` | `ConcurrentStack<T>` |
| Manual `lock` + `Dictionary` | `ConcurrentDictionary` (simpler, often faster) |

> **Note:** If you need a thread-safe list with index access, use `lock` around a regular `List<T>` — there is no `ConcurrentList<T>` in .NET.

**Reviewer tip:** Search for `static Dictionary<`, `static List<`, `static HashSet<`. If they are accessed from multiple threads (e.g., in ASP.NET request handlers or background services), they must use concurrent alternatives or explicit locking.

---

## 19. Avoid Task.Run() in ASP.NET Request Handlers

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Performance | `Task.Run(`, `Task.Factory.StartNew(` inside controllers or request handlers |

**Why it matters:** ASP.NET already processes each request on a thread pool thread. Wrapping synchronous work in `Task.Run()` inside a request handler **borrows a second thread pool thread** to do the same work, adding overhead (context switch, scheduling) with zero benefit. Under load, this wastes thread pool threads and **reduces throughput**. `Task.Run()` is designed for offloading CPU-bound work from UI threads (WinForms/WPF), not for ASP.NET.

### Bad Example

```csharp
[HttpGet("report")]
public async Task<IActionResult> GetReport()
{
    // wasteful — moves work from one thread pool thread to another
    var report = await Task.Run(() => GenerateReport());
    return Ok(report);
}

private Report GenerateReport()
{
    // CPU-bound work
    return new Report { Data = ComputeData() };
}
```

### Good Example — Call Synchronous Code Directly

```csharp
[HttpGet("report")]
public IActionResult GetReport()
{
    // no unnecessary thread switch — runs on the request thread directly
    var report = GenerateReport();
    return Ok(report);
}
```

### Good Example — True Async for I/O

```csharp
[HttpGet("report")]
public async Task<IActionResult> GetReport()
{
    // truly async I/O — thread is released while waiting
    var data = await _repository.GetReportDataAsync();
    return Ok(data);
}
```

> **When Task.Run() IS appropriate:** In desktop apps (WinForms/WPF) to keep the UI thread responsive, or in background services where you explicitly want to parallelize CPU-bound work across multiple threads.

**Reviewer tip:** Search for `Task.Run(` and `Task.Factory.StartNew(` inside ASP.NET controllers or middleware. If the wrapped code is synchronous, it should be called directly. If it's I/O, it should be made truly async.

---

## 20. Avoid the dynamic Keyword

| Severity | Category | Search Pattern |
|----------|----------|----------------|
| **Medium** | Code Quality / Performance | `dynamic ` |

**Why it matters:** The `dynamic` keyword bypasses **compile-time type checking**. Errors that would normally be caught at build time (typos in property names, wrong argument types, missing methods) only surface at **runtime** as `RuntimeBinderException`. It also incurs significant performance overhead due to runtime binding via the DLR (Dynamic Language Runtime) and makes code harder to refactor, navigate, and understand in an IDE.

### Bad Example

```csharp
public void ProcessPayment(dynamic payment)
{
    // no compile-time safety — typo in property name won't be caught until runtime
    decimal amount = payment.Ammount; // RuntimeBinderException at runtime — "Amount" misspelled
    string method = payment.Method;

    _gateway.Charge(amount, method);
}
```

```csharp
// dynamic used to avoid creating a proper type
dynamic config = JsonConvert.DeserializeObject(json);
string host = config.database.host; // no IntelliSense, no refactoring support
int port = config.database.port;
```

### Good Example — Use Strong Types

```csharp
public class Payment
{
    public decimal Amount { get; set; }
    public string Method { get; set; }
}

public void ProcessPayment(Payment payment)
{
    // compile-time safety — typos caught immediately
    _gateway.Charge(payment.Amount, payment.Method);
}
```

```csharp
// Strongly-typed configuration
public class DatabaseConfig
{
    public string Host { get; set; }
    public int Port { get; set; }
}

var config = JsonConvert.DeserializeObject<DatabaseConfig>(json);
string host = config.Host; // IntelliSense, compile-time checks, refactoring support
```

### Good Example — Use Generics Instead of dynamic

```csharp
// Instead of: dynamic result = GetValue();
// Use generics:
public T GetValue<T>(string key)
{
    return JsonSerializer.Deserialize<T>(cache.Get(key));
}
```

> **When dynamic IS appropriate:** COM interop (e.g., Office automation), interacting with dynamic languages (IronPython), or rare reflection-heavy scenarios where the type is truly unknown at compile time.

**Reviewer tip:** Search for `dynamic ` (with trailing space). Each occurrence should have a documented justification. If it's used to avoid creating a class, create the class instead.

---

## Quick-Scan Reference Table

| # | Rule | Grep / Search Pattern | Severity |
|---|------|-----------------------|----------|
| 1 | Dispose streams | `new FileStream`, `new MemoryStream`, `new StreamReader`, `new StreamWriter` | High |
| 2 | HttpClient singleton | `new HttpClient()` | High |
| 3 | No .Result/.Wait() | `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` | Critical |
| 4 | SerialPort timeouts | `new SerialPort` | Medium |
| 5 | SqlParameter required | `"SELECT " +`, `$"SELECT`, `"INSERT " +`, `"UPDATE " +`, `"DELETE " +` | Critical |
| 6 | Dispose SQL objects | `new SqlConnection`, `new SqlCommand`, `ExecuteReader`, `BeginTransaction` | High |
| 7 | C# code conventions | Use `.editorconfig` + `dotnet format` | Medium |
| 8 | .gitignore | Check repo root for `.gitignore` | N/A |
| 9 | Redis/Cosmos/SB singleton | `ConnectionMultiplexer.Connect`, `new CosmosClient`, `new ServiceBusClient` | High |
| 10 | Use Serilog | `Console.WriteLine` (as log), custom `Logger` class | Medium |
| 11 | Static collections | `static List<`, `static Dictionary<`, `static ArrayList` | High |
| 12 | Unsubscribe events / dispose timers | `+=` (event), `new Timer(` | High |
| 13 | No hardcoded secrets | `"Password=`, `"ApiKey"`, `"Bearer "`, `"sk-` | Critical |
| 14 | StringBuilder in loops | `+=` on string inside loops | Medium |
| 15 | Config-driven connection strings | `new SqlConnection("`, inline connection strings | High |
| 16 | No async void | `async void` | High |
| 17 | Use throw; not throw ex; | `throw ex;` | Medium |
| 18 | Concurrent collections for shared state | `static Dictionary<`, `static List<` with multi-thread access | High |
| 19 | No Task.Run() in ASP.NET handlers | `Task.Run(`, `Task.Factory.StartNew(` in controllers | Medium |
| 20 | Avoid dynamic keyword | `dynamic ` | Medium |

---

## References

- [IDisposable Interface](https://learn.microsoft.com/en-us/dotnet/api/system.idisposable)
- [HttpClient Guidelines](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)
- [Async/Await Best Practices](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [SQL Injection Prevention](https://learn.microsoft.com/en-us/sql/relational-databases/security/sql-injection)
- [C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [Serilog](https://github.com/serilog/serilog)
- [StackExchange.Redis — Basic Usage](https://stackexchange.github.io/StackExchange.Redis/Basics.html)
- [VisualStudio.gitignore](https://github.com/github/gitignore/blob/main/VisualStudio.gitignore)
- [Async Void Guidance](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming#avoid-async-void)
- [ConcurrentDictionary](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2)
- [Safe Storage of Secrets](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets)
- [StringBuilder Performance](https://learn.microsoft.com/en-us/dotnet/standard/base-types/stringbuilder)
