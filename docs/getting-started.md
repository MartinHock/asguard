# Getting Started with AsGuard

Setting up AsGuard in your ASP.NET Core application takes less than 5 minutes. This guide will walk you through the package installation, basic setup, middleware placement, dashboard verification, and credential security.

---

## 1. Prerequisites & Installation

AsGuard supports **.NET 8, .NET 9, and .NET 10**.

Install the package in your ASP.NET Core Web API, Minimal API, or MVC project using the .NET CLI or NuGet Package Manager:

```bash
dotnet add package AsGuard
```

Or via Package Manager Console in Visual Studio:

```powershell
Install-Package AsGuard
```

---

## 2. Minimal Setup Example

For quick local development, you can configure AsGuard with a self-contained local **SQLite** database (which requires zero database server setup):

Open your application's `Program.cs` and configure the services and middleware:

```csharp
using AsGuard.Domain.RequestLogging;
using AsGuard.Extensions;
using Microsoft.Extensions.Logging;

var builder = WebApplication.CreateBuilder(args);

// Add standard application services
builder.Services.AddControllers();

// Add AsGuard monitoring services
builder.Services.AddRequestLogging(options =>
{
    // Configure SQLite storage (tables and indexes are auto-created on startup)
    options.DatabaseProvider = LoggingDatabaseProvider.Sqlite;
    options.ConnectionString = "Data Source=asguard_monitoring.db";

    // Set up secure credentials for the dashboard
    // NOTE: Default 'admin'/'admin' credentials are explicitly blocked at startup.
    options.DashboardRoute = "/request-logs-ui";
    options.DashboardUsername = builder.Configuration["AsGuard:Username"] ?? "asguard_admin";
    options.DashboardPassword = builder.Configuration["AsGuard:Password"] ?? "superSecretPassword123!";

    // Enable detailed error and host log capture
    options.EnableExceptionLogging = true;
    options.CaptureHostLogs = true;
    options.HostLogMinimumLevel = LogLevel.Warning;

    // Enable automatic APM database & HttpClient tracing (default: true)
    options.EnableApmTracing = true;
});

var app = builder.Build();

// Configure standard ASP.NET Core pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();

// REGISTER ASGUARD MIDDLEWARE
// It is critical to place UseRequestLogging() after routing/exception handlers
// but before authorization and endpoint execution.
app.UseRequestLogging();

app.UseAuthorization();

// Sample Endpoint to generate normal HTTP logs
app.MapGet("/api/greet", (string? name) => 
    Results.Ok(new { message = $"Hello, {name ?? "Guest"}!" }));

// Sample Endpoint to trigger host ILogger Warnings
app.MapGet("/api/warn", (ILogger<Program> logger) =>
{
    logger.LogWarning("This is a warning event captured by AsGuard.");
    return Results.Ok(new { status = "Warning registered" });
});

// Sample Endpoint to trigger Unhandled Exceptions
app.MapGet("/api/error", () =>
{
    throw new InvalidOperationException("This exception is captured and logged automatically by AsGuard!");
});

app.MapControllers();

app.Run();
```

---

## 3. Middleware Ordering Guide

The placement of `app.UseRequestLogging()` in the middleware pipeline is crucial. 

AsGuard sets up several diagnostic and monitoring services under a single method call:
- Correlation ID generation and propagation (`X-Correlation-ID` and W3C `traceparent`).
- Host logging scope mapping.
- Handled and unhandled exception interceptors.
- HTTP Request/Response details and body capture.
- Dashboard UI and REST API route mappings.

To ensure all requests are logged correctly and exceptions are accurately associated with their HTTP contexts, follow this pipeline order:

| Order | Middleware Method | Why it belongs here |
| :---: | :--- | :--- |
| **1** | `app.UseExceptionHandler(...)` | If you use standard exception pages or handlers, register them *first*. This allows AsGuard to intercept the exception, commit it to background queues, and let the outer handler generate the client HTTP response. |
| **2** | `app.UseCors(...)` / `app.UseHsts()` | Security and policy middlewares should execute early before any request logging starts. |
| **3** | `app.UseHttpsRedirection()` | Ensures requests are redirected to secure HTTPS before AsGuard logs them (preventing duplicate logging of redirected HTTP requests). |
| **4** | **`app.UseRequestLogging()`** | **REGISTER ASGUARD HERE**. Places request logging early in the pipeline to capture full duration, body streams, correlation scopes, and errors from all route endpoints. |
| **5** | `app.UseAuthentication()` | Placing Authentication *after* `UseRequestLogging` allows AsGuard to log requests that fail with `401 Unauthorized` responses. |
| **6** | `app.UseAuthorization()` | Captures requests that fail with `403 Forbidden` responses. |
| **7** | `app.MapControllers()` / Endpoints | The actual controller actions or minimal endpoints where business logic runs. |

### Visual Pipeline Flow:
```text
  [HTTP Request In]
         │
         ▼
  ┌──────────────┐
  │ Exception    │  ◄── Catches unhandled errors re-thrown by AsGuard
  │ Handler      │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Https / CORS │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ AsGuard      │  ◄── Correlation ID generated, Scope set up,
  │ Middleware   │      request/response streaming opened, APM initialized.
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Auth & Authz │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Endpoints /  │  ◄── Your API controllers execute. 
  │ Controllers  │      EF Core queries and outgoing HttpClients auto-traced.
  └──────────────┘
```

---

## 4. Verifying Your Setup

Run your application and test the monitoring pipeline:

### Step A: Accessing the Dashboard
Open your browser and navigate to your configured dashboard route:
```text
https://localhost:<port>/request-logs-ui
```
*   You will be redirected (`302`) to the dedicated **AsGuard Login Page** (`/request-logs-login`) — no annoying browser HTTP Basic Auth popups.
*   Enter your configured `DashboardUsername` (`asguard_admin`) and `DashboardPassword` (`superSecretPassword123!`).
*   Upon successful authentication, an encrypted session cookie (`AsGuardDashboardAuth`) is issued with sliding expiration, and you are redirected directly to the dashboard.

### Step B: Generating HTTP Traffic
Open another tab or use curl/Postman to trigger test requests:
1.  **Normal Request**: Navigate to `https://localhost:<port>/api/greet?name=Developer`. 
2.  **Warning Event**: Navigate to `https://localhost:<port>/api/warn`. This writes a `Warning` log to `ILogger`.
3.  **Unhandled Exception**: Navigate to `https://localhost:<port>/api/error`. This displays your standard error response to the client, but AsGuard captures the full unhandled stack trace behind the scenes.

### Step C: Inspecting the Live Dashboard
Return to the Dashboard tab:
*   **Request Logs**: The requests appear in real time over the live Server-Sent Events (SSE) connection without page refreshing.
*   **Exceptions**: Navigate to the **Exceptions** tab to see both the `Warning` from `/api/warn` and the unhandled `InvalidOperationException` from `/api/error`.
*   **Request Details**: Click on any request row to open the modal inspector. You can view headers, response status, execution duration, correlation timeline, APM trace Gantt charts, and copy the request as a `curl` command.

---

## 5. Security Best Practices

> [!IMPORTANT]
> **Startup Security Validation**:
> 1.  **Change Default Credentials**: AsGuard's startup validation actively blocks execution if credentials are set to the legacy default (`admin`/`admin`).
> 2.  **Configuration Storage**: Never hardcode credentials in source code. Store them in user secrets (`dotnet user-secrets`) during development, and environment variables or Azure Key Vault in production:
>     ```json
>     {
>       "AsGuard": {
>         "Username": "production_ops",
>         "Password": "YourVeryLongStrongPassword123!"
>       }
>     }
>     ```
> 3.  **Session Cookies & HTTPS**: The dashboard session cookie defaults to `CookieSecurePolicy.Always`. In production behind a TLS-terminating reverse proxy (like Nginx, Traefik, or AWS ALB), ensure you configure `app.UseForwardedHeaders()` so ASP.NET Core recognizes HTTPS requests correctly.

---

## Next Steps

*   **[Configuration & Options Reference](configuration.md)**: Tune database providers, queue capacities, retention rules, and alerting.
*   **[HTTP Traffic Monitoring](features/request-logging.md)**: Configure payload body capture, allowlists, and sensitive data masking.
*   **[APM & Trace Timelines](features/apm-tracing.md)**: Explore how AsGuard traces database queries and downstream HTTP calls.
