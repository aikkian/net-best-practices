---
name: dotnet-review
description: Scan .NET/C# codebase for common best practice violations across 20 rules covering resource management, security, performance, concurrency, and code quality.
---

# .NET Best Practices Code Review Scanner

You are a .NET code reviewer. Scan the current codebase for violations of the rules below. For each violation found, report the **file path**, **line number**, **rule number**, and a **brief description** of the issue.

## Instructions

1. Search all `.cs` files in the project using the grep patterns listed for each rule.
2. For each match, analyze whether it is an actual violation or a false positive (e.g., a `using` statement nearby, a proper singleton pattern, etc.).
3. Group findings by severity: **Critical** → **High** → **Medium**.
4. Output a summary table at the end.
5. If no violations are found, confirm the codebase is clean.

## Rules to Scan

### Critical Severity

**Rule 3 — No .Result or .Wait() (Deadlock Risk)**
- Search: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`
- Violation: Calling `.Result` or `.Wait()` on a Task. Must use `await` instead.
- Exception: `Main` method in legacy console apps (must be documented).

**Rule 5 — Use SqlParameter (SQL Injection Prevention)**
- Search: `"SELECT " +`, `"INSERT " +`, `"UPDATE " +`, `"DELETE " +`, `$"SELECT`, `$"INSERT`, `$"UPDATE`, `$"DELETE`, `string.Format("SELECT`
- Violation: SQL query built with string concatenation or interpolation instead of parameterized queries.

**Rule 13 — No Hardcoded Secrets**
- Search: `"Server=`, `"Data Source=`, `"Password=`, `"pwd=`, `"ApiKey"`, `"Bearer "`, `"sk-`, `"mongodb://`, `"redis://`
- Violation: Connection strings, API keys, or credentials hardcoded in source code. Must use configuration/secrets management.

### High Severity

**Rule 1 — Dispose All Streams (IDisposable)**
- Search: `new FileStream`, `new MemoryStream`, `new StreamReader`, `new StreamWriter`, `new NetworkStream`, `new BufferedStream`
- Violation: Stream created without a `using` statement or explicit `Dispose()` call.

**Rule 2 — HttpClient Must Be Singleton or Static**
- Search: `new HttpClient()`
- Violation: `HttpClient` instantiated per call or in a transient scope. Must be static, singleton, or use `IHttpClientFactory`.

**Rule 6 — Dispose SQL Objects**
- Search: `new SqlConnection`, `new SqlCommand`, `ExecuteReader`, `BeginTransaction`
- Violation: `SqlConnection`, `SqlCommand`, `SqlDataReader`, or `SqlTransaction` created without `using` statement.

**Rule 9 — Singleton + Lazy for Redis/CosmosDB/Service Bus**
- Search: `ConnectionMultiplexer.Connect`, `new CosmosClient`, `new ServiceBusClient`, `new ServiceBusSender`
- Violation: Connection created per request instead of as a singleton or `Lazy<T>`.

**Rule 11 — Static Collections (Memory Leak)**
- Search: `static List<`, `static Dictionary<`, `static ArrayList`, `static HashSet<`, `static ConcurrentDictionary<`
- Violation: Static collection that grows without bounds (no eviction/cleanup). Should use `IMemoryCache` or bounded collection.

**Rule 12 — Unsubscribe Events / Dispose Timers**
- Search: `+=` (on event handlers), `new Timer(`
- Violation: Event subscription without corresponding `-=` unsubscription, or `Timer` created without `Dispose`.

**Rule 15 — Connection Strings from Configuration**
- Search: `new SqlConnection("`, `ConnectionMultiplexer.Connect("`, `new CosmosClient("` with inline strings
- Violation: Connection string hardcoded instead of coming from `IConfiguration`.

**Rule 16 — No async void**
- Search: `async void`
- Violation: `async void` method that is not an event handler. Must return `async Task`.

**Rule 18 — Concurrent Collections for Shared State**
- Search: `static Dictionary<`, `static List<`, `static HashSet<`
- Violation: Non-thread-safe collection accessed from multiple threads (e.g., in ASP.NET controllers or background services). Must use `ConcurrentDictionary`, `ConcurrentBag`, etc.

### Medium Severity

**Rule 4 — SerialPort Timeouts**
- Search: `new SerialPort`
- Violation: `SerialPort` without `ReadTimeout` and `WriteTimeout` set.

**Rule 7 — C# Code Conventions**
- Check: Naming follows Microsoft conventions (PascalCase for public members, `_camelCase` for private fields), `.editorconfig` exists.
- Violation: Inconsistent naming or missing `.editorconfig`.

**Rule 10 — Use Serilog, Not Custom Logging**
- Search: `Console.WriteLine` (as logging), custom `Logger` or `LogHelper` classes, `File.AppendAllText` for logging
- Violation: Hand-rolled logging instead of structured logging with Serilog/ILogger.

**Rule 14 — StringBuilder in Loops**
- Search: `+=` on a string variable inside `for`, `foreach`, `while`
- Violation: String concatenation in a loop. Must use `StringBuilder`.

**Rule 17 — Use throw; Not throw ex;**
- Search: `throw ex;`
- Violation: `throw ex;` resets the stack trace. Must use `throw;` or wrap in a new exception.

**Rule 19 — No Task.Run() in ASP.NET Handlers**
- Search: `Task.Run(`, `Task.Factory.StartNew(` inside controllers
- Violation: Wrapping sync code in `Task.Run()` inside an ASP.NET request handler. Call directly or use true async.

**Rule 20 — Avoid dynamic Keyword**
- Search: `dynamic `
- Violation: Use of `dynamic` without justification. Must use strong types or generics.

**Rule 21 — ThreadPool.SetMinThreads for .NET 5+ on Linux**
- Search: `ThreadPool.SetMinThreads` in `Program.cs` or startup entry point
- Violation: .NET 5+ project targeting Linux deployment without `ThreadPool.SetMinThreads(200, 200)` configured at startup. Default thread pool ramp-up on Linux is too slow and causes request queuing under burst load.
- Note: Must be called **before** `WebApplication.CreateBuilder()`. Recommended minimum: 200 worker threads, 200 I/O threads.

**Rule 22 — Avoid ConfigureAwait(false) and ConfigureAwait(true)**
- Search: `ConfigureAwait(false)`, `ConfigureAwait(true)`
- Violation: `ConfigureAwait(false)` used in ASP.NET Core or .NET 5+ application code where there is no `SynchronizationContext`. `ConfigureAwait(true)` is always redundant (it's the default).
- Exception: Shared library/NuGet packages consumed by WinForms, WPF, or legacy ASP.NET Framework may still need `ConfigureAwait(false)`.

### Configuration Check

**Rule 8 — Visual Studio .gitignore**
- Check: `.gitignore` exists in repo root with standard VisualStudio template entries.

## Output Format

After scanning, produce a report in this format:

### Violations Found

| Severity | Rule # | Rule | File | Line | Issue |
|----------|--------|------|------|------|-------|
| Critical | 5 | SqlParameter | src/Data/UserRepo.cs | 42 | SQL query built with string concatenation |
| High | 2 | HttpClient | src/Services/ApiClient.cs | 15 | `new HttpClient()` created per method call |
| ... | ... | ... | ... | ... | ... |

### Summary

- **Critical:** X violations
- **High:** X violations
- **Medium:** X violations
- **Total:** X violations

### Recommendations

List the top 3 most impactful fixes to prioritize.
