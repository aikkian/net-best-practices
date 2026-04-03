# .NET Best Practices — Project Rules

Follow these rules when writing or modifying C# / .NET code in this project.

## Critical — Must Never Violate

- **Never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`** on Tasks. Always use `await` with `async` methods all the way up the call chain.
- **Never concatenate strings into SQL queries.** Always use `SqlParameter` or parameterized queries (Dapper, EF Core). No `$"SELECT ... {variable}"` or `"SELECT ... " + variable`.
- **Never hardcode secrets** (connection strings, API keys, passwords) in source code. Use `appsettings.json`, User Secrets, environment variables, or Azure Key Vault.

## High — Must Follow

- **All streams must be disposed.** Use `using var` or `using () { }` for `FileStream`, `MemoryStream`, `StreamReader`, `StreamWriter`, `NetworkStream`, etc.
- **HttpClient must be singleton or static.** Never `new HttpClient()` per request. Use a `static readonly` field or `IHttpClientFactory`.
- **Dispose all SQL objects.** `SqlConnection`, `SqlCommand`, `SqlDataReader`, and `SqlTransaction` must be wrapped in `using` statements.
- **Redis, CosmosDB, Service Bus connections must be singleton.** Use `Lazy<T>` for thread-safe initialization or register as singleton in DI.
- **No `async void`.** Return `async Task` instead. Only exception: UI event handlers (WinForms/WPF).
- **Unsubscribe event handlers (`-=`)** when the subscriber is disposed. Dispose `Timer` objects.
- **Static collections must be bounded.** If using `static List<T>`, `static Dictionary<K,V>`, etc., ensure items are evicted. Prefer `IMemoryCache` with expiration.
- **Use concurrent collections** (`ConcurrentDictionary`, `ConcurrentQueue`) when data is accessed from multiple threads. Regular `Dictionary`/`List` are not thread-safe.
- **Connection strings must come from `IConfiguration`**, not hardcoded inline.

## Medium — Should Follow

- **SerialPort must have `ReadTimeout` and `WriteTimeout` set.** Default is infinite timeout.
- **Follow Microsoft C# naming conventions.** PascalCase for public members, `_camelCase` for private fields. Use `.editorconfig` + `dotnet format`.
- **Use Serilog / `ILogger<T>`** for logging. No `Console.WriteLine` or custom log classes.
- **Use `StringBuilder`** for string concatenation inside loops.
- **Use `throw;` not `throw ex;`** to preserve stack traces.
- **Avoid `Task.Run()` in ASP.NET request handlers.** ASP.NET already runs on thread pool threads.
- **Avoid the `dynamic` keyword.** Use strong types or generics instead.

## Configuration

- **Repository must have a VisualStudio `.gitignore`** from the standard GitHub template.
- **Include an `.editorconfig`** to enforce code style automatically.
