# Configuration & Options Reference

Every AsGuard feature, queue, and security setting is configured through `RequestLoggingOptions` in the `AddRequestLogging` call.

---

## Complete Options Reference

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| **DatabaseProvider** | `LoggingDatabaseProvider` | `SqlServer` | `SqlServer`, `PostgreSql`, `Sqlite`, or `InMemory`. |
| **ConnectionString** | `string` | `""` | Connection string for SQL Server, PostgreSQL, or SQLite. Unused for `InMemory`. |
| **QueueCapacity** | `int` | `10_000` | Max in-memory request log queue size. |
| **QueueOverflowPolicy** | `QueueOverflowPolicy` | `DropNewest` | `DropNewest` or `DropOldest` when the request queue is full. |
| **BatchSize** | `int` | `100` | Log entries flushed per database batch. |
| **EnableExceptionLogging** | `bool` | `true` | Enables exception and log event persistence. |
| **EnableApmTracing** | `bool` | `true` | Enables automatic APM tracing for EF Core queries and HttpClient calls. |
| **EnableW3CPropagation** | `bool` | `true` | Enables W3C `traceparent` header propagation across HTTP boundaries. |
| **ExceptionQueueCapacity** | `int` | `5_000` | Max in-memory exception queue size. |
| **ExceptionQueueOverflowPolicy** | `QueueOverflowPolicy` | `DropNewest` | Overflow behavior for the exception queue. |
| **ExceptionBatchSize** | `int` | `50` | Exception/log entries flushed per database batch. |
| **MaxInMemoryEntries** | `int` | `10_000` | Max rows retained by `InMemory` storage before eviction. |
| **BroadcastQueueCapacity** | `int` | `2_048` | Live broadcast queue capacity for the SSE stream. |
| **BroadcastOverflowPolicy** | `BroadcastOverflowPolicy` | `DropOldest` | Overflow behavior for the broadcast queue. |
| **BroadcastDetailMode** | `BroadcastDetailMode` | `SummaryOnly` | `SummaryOnly` or `Full` detail pushed over SSE. |
| **EnableDetailedLogEndpoint** | `bool` | `true` | Enables endpoints that fetch headers, bodies, and stack traces on demand. |
| **CaptureHostLogs** | `bool` | `true` | Captures host `ILogger` events into the exceptions dashboard. |
| **HostLogMinimumLevel** | `LogLevel` | `Warning` | Minimum host `ILogger` level captured. |
| **ExcludedHostLogCategoryPrefixes** | `HashSet<string>` | `["AsGuard."]` | Logger categories excluded from capture (case-insensitive prefixes). |
| **CorrelationHeaderName** | `string` | `"X-Correlation-ID"` | HTTP header used to propagate correlation IDs. |
| **RequestRetentionDays** | `int?` | `null` | Retention period for request data. Disabled when null. |
| **ExceptionRetentionDays** | `int?` | `null` | Retention period for exception data. Disabled when null. |
| **RetentionCleanupInterval** | `TimeSpan` | `6 hours` | How often the retention worker deletes expired logs. |
| **QueuePressureAlertThreshold** | `double?` | `0.8` | Queue depth ratio that triggers a pressure alert. Disabled when null. |
| **ExceptionSpikeAlertThreshold** | `int?` | `null` | Exceptions per rolling window that trigger an alert. Disabled when null. |
| **ExceptionSpikeAlertWindow** | `TimeSpan` | `5 minutes` | Rolling window for exception spike alerts. |
| **AlertCooldown** | `TimeSpan` | `5 minutes` | Minimum time between repeated alerts of the same type. |
| **MaxAlertHistoryStored** | `int` | `200` | Max fired alerts retained (`GET /request-logs-api/alerts`). |
| **SamplingRate** | `double?` | `null` | Fraction of requests logged (`0.0`–`1.0`). `null` = 100%. |
| **LogRequestsSlowerThan** | `TimeSpan?` | `null` | Only log requests slower than this (exceptions always log). |
| **LoggableStatusCodes** | `HashSet<int>` | `[]` | If set, only these status codes are logged (exceptions always log). |
| **MaxCapturedBodyBytes** | `int` | `int.MaxValue` | Byte cap on captured body length before truncation. |
| **LogRequestBody** | `bool` | `false` | Captures and persists request bodies. |
| **LogResponseBody** | `bool` | `false` | Captures and persists response bodies. |
| **ExcludedPathPrefixes** | `HashSet<string>` | `["/health", "/swagger", "/metrics"]` | Path prefixes skipped by request logging. |
| **ExcludedExactPaths** | `HashSet<string>` | `["/health", "/metrics", "/swagger", "/favicon.ico"]` | Exact paths skipped by request logging. |
| **LoggableContentTypes** | `string[]` | `[]` | Content types eligible for body capture. Empty = all types. |
| **SensitiveHeaders** | `HashSet<string>` | `[]` | Headers redacted from logs. `Set-Cookie` is always redacted. |
| **SensitiveBodyKeys** | `HashSet<string>` | `[]` | JSON keys redacted in request/response bodies. |
| **EnrichLog** | `Action<HttpContext, RequestLog>?` | `null` | Custom delegate to enrich the captured log with extra data or tags. |
| **DashboardRoute** | `string` | `"/request-logs-ui"` | Route prefix for the dashboard. Empty disables it. |
| **DashboardUsername** | `string` | `""` | Basic Auth username for the dashboard, API, and stream. |
| **DashboardPassword** | `string` | `""` | Basic Auth password for the dashboard, API, and stream. |
| **EnableHostMetrics** | `bool` | `true` | Enables process CPU/memory/ThreadPool sampling and graphs. |
| **HostMetricsSampleInterval** | `TimeSpan` | `15 seconds` | Sampling interval for host resource metrics. |
| **MaxHostMetricsStored** | `int` | `120` | Samples retained in the metrics ring buffer (30 min at the default interval). |
| **EnableLiveLogConsole** | `bool` | `true` | Streams live `ILogger` output to the dashboard. |
| **LiveLogConsoleMinLevel** | `LogLevel` | `Information` | Minimum level forwarded to the live console. |
| **EnableHealthChecksVisualizer** | `bool` | `true` | Enables status cards for registered `HealthCheckService` checks. |
| **HealthHistorySampleInterval** | `TimeSpan` | `30 seconds` | Sampling interval for health check history. |
| **MaxHealthHistoryStored** | `int` | `120` | Samples retained in the health history buffer (1 hour at the default interval). |
| **EnableRedisManagement** | `bool` | `false` | Enables the Redis tab: key browser, inspector/editor, live stats. Opt-in. |
| **RedisManagementReadOnly** | `bool` | `false` | Makes the Redis tab browse-only; edits, TTL changes, deletes, and flushes are rejected server-side. |
| **AllowRedisFlushDatabase** | `bool` | `false` | Allows `FLUSHDB` from the dashboard. Also requires `allowAdmin=true` on the connection and a typed confirmation. |
| **RedisConnections** | `IList<RedisConnectionDescriptor>` | empty | Explicit Redis endpoints, used only when none is discoverable from DI. |
| **RedisScanPageSize** | `int` | `100` | Keys requested per `SCAN` round trip. AsGuard never issues `KEYS`. |
| **MaxRedisValueBytes** | `int` | `262144` | Byte cap on an inspected Redis value; larger values truncate and become read-only. |
| **RedisStatsSampleInterval** | `TimeSpan` | `5 seconds` | Sampling interval for Redis `INFO` stats pushed over SSE. |
| **RedisSlowlogIncludeArguments** | `bool` | `false` | Shows full command arguments in the Redis slow log. Off by default. |

---

## Storage Providers

Table structures are created and validated automatically at startup — no EF migrations to run.

| Provider | Best for | Notes |
| :--- | :--- | :--- |
| `SqlServer` | Enterprise production | Standard connection string. |
| `PostgreSql` | Open-source production | Standard connection string. |
| `Sqlite` | Local dev, single-container, small apps | `"Data Source=asguard_logs.db;Cache=Shared;"` |
| `InMemory` | Tests, ephemeral diagnostics | **Not for production.** Restarting the process clears all logs. |

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.DatabaseProvider = LoggingDatabaseProvider.PostgreSql;
    options.ConnectionString = builder.Configuration.GetConnectionString("PostgresConnection")!;

    // InMemory only: cap memory usage
    options.MaxInMemoryEntries = 5_000;
});
```

---

## Queue Tuning

Logs persist asynchronously through bounded queues, so tune capacity and overflow behavior for your load:

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.QueueCapacity = 20_000;
    options.QueueOverflowPolicy = QueueOverflowPolicy.DropOldest;
    options.BatchSize = 250;

    options.ExceptionQueueCapacity = 2_000;
    options.ExceptionQueueOverflowPolicy = QueueOverflowPolicy.DropNewest;
    options.ExceptionBatchSize = 10;
});
```

- **`DropNewest`** — discard new entries when full. Safest default; protects memory during a database outage.
- **`DropOldest`** — evict the oldest entries to make room. Use when only the most recent diagnostics matter.

---

## Retention & Cleanup

A background worker deletes logs older than your configured threshold:

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.RequestRetentionDays = 14;
    options.ExceptionRetentionDays = 60;
    options.RetentionCleanupInterval = TimeSpan.FromHours(12); // Default: 6 hours
});
```

`RequestRetentionDays` and `ExceptionRetentionDays` default to `null` (disabled) — leave them null to manage retention yourself (partitioning, custom cron jobs, etc).

---

## Host Metrics, Live Console & Health Checks

All three are in-memory, zero-migration features enabled by default:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Process CPU/memory/ThreadPool graphs
    options.EnableHostMetrics = true;
    options.HostMetricsSampleInterval = TimeSpan.FromSeconds(30);
    options.MaxHostMetricsStored = 240; // 2 hours at 30s

    // Live ILogger streaming to the dashboard
    options.EnableLiveLogConsole = true;
    options.LiveLogConsoleMinLevel = LogLevel.Debug;

    // Status cards + trend chart for registered health checks
    options.EnableHealthChecksVisualizer = true;
    options.HealthHistorySampleInterval = TimeSpan.FromSeconds(15);
    options.MaxHealthHistoryStored = 240; // 1 hour at 15s
});
```

The health visualizer hooks into standard ASP.NET Core health checks — register them normally:

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!);
```

Raw history is queryable at `GET /request-logs-api/host-metrics` and `GET /request-logs-api/health/history`.

---

## W3C Trace Context

`EnableW3CPropagation` (default `true`) propagates standard `traceparent` headers across service boundaries:

- Format: `00-{traceId}-{spanId}-01`, per the W3C spec.
- If an incoming correlation header isn't a valid 32-character hex `traceId`, AsGuard deterministically MD5-hashes it into one, so tracing still works without altering your original correlation ID.

See **[Enrichment & Correlation](features/enrichment-correlation.md)** for the full correlation ID and tracing flow.

---

## Redis Management

Opt-in key browser, value editor, and live `INFO` stats — the only feature flag here that defaults to `false`, since it touches application data, not observability data.

```csharp
options.EnableRedisManagement = true;
```

In most apps that's the only setting needed: AsGuard discovers your Redis connection from the host's service provider instead of asking for a connection string, which is what makes it work with token-based auth (e.g. Azure Cache for Redis under Microsoft Entra ID). Keys are always enumerated with `SCAN`, never `KEYS`, and destructive actions are gated behind `RedisManagementReadOnly` and `AllowRedisFlushDatabase`.

See **[Redis Management](features/redis-management.md)** for the connection discovery ladder, editing surface, safety gates, and REST API.

---

## Alert History

Every alert AsGuard fires (queue pressure, exception spike, persistence/broadcast failures) is kept in an in-memory ring buffer in addition to being handed to your `IAsGuardAlertSink`:

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.MaxAlertHistoryStored = 500; // Default: 200
});
```

Query the raw history via `GET /request-logs-api/alerts`. See **[Alerts & Production Operations](features/alerts-production.md)** for wiring up Slack/Teams webhooks.
