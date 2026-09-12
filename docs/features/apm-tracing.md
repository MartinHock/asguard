# APM & Trace Timelines

AsGuard includes a zero-configuration Application Performance Monitoring (APM) engine. It automatically tracks database calls and outbound HTTP calls made during an HTTP request, and renders them as a Gantt chart timeline in the dashboard.

---

## 1. What Gets Traced

AsGuard hooks into .NET's native `DiagnosticListener` framework at startup — no interceptors, delegating handlers, or code changes required:

- **Entity Framework Core** — every SQL query, migration, or LINQ-translated command.
- **HttpClient** — every outgoing call (Stripe, Auth0, internal services, etc).

Spans are collected in a request-scoped, lock-free collector (`IApmSpanCollector`) safe for concurrent async work like `Task.WhenAll`. Because it's `Scoped`, collected spans are garbage-collected as soon as the request ends — nothing leaks across requests.

Spans are queued into a background channel at request completion and persisted in bulk by a hosted worker (`ApmSpanPersistenceWorker`), so tracing adds no latency to the request itself.

---

## 2. Per-Request Gantt Timeline

Any request detail panel that produced DB or HTTP spans shows a **Trace Spans Timeline**:

```text
Trace Spans Timeline
Total Duration: 450 ms | Spans Count: 3

[SQL] SELECT * FROM [Users] WHERE [Id] = @p0
=== [50ms (offset 10ms)]

[HTTP] POST https://api.stripe.com/v1/charges
======================== [300ms (offset 70ms)]

[SQL] UPDATE [Orders] SET [Status] = @p0 WHERE [Id] = @p1
== [20ms (offset 380ms)]
```

- Database spans render as **purple** bars, HTTP spans as **amber** bars.
- Failed spans (thrown exceptions, or 4xx/5xx responses) get a `⚠️` marker and a rose border.
- Click any bar to expand the full SQL text or request URI, plus exception details on failure.

---

## 3. Slow Traces (Cross-Request View)

The Gantt timeline above is scoped to one correlation ID. The dashboard's **Slow Traces** tab ranks the slowest DB/HttpClient spans across *all* requests in a time window (`15m`/`1h`/`6h`/`24h`/`7d`). Clicking a row jumps to that span's correlation ID in the log view.

```http
GET /request-logs-api/apm/slow-traces?range=1h&limit=20
```

This is a relative top-N ranking, not an absolute "slow" threshold — there's no `LogRequestsSlowerThan` equivalent for APM spans, so every span in the window is a candidate regardless of actual duration.

A companion endpoint aggregates outbound HTTP calls by target host instead of by span:

```http
GET /request-logs-api/apm/dependencies?range=1h
```
Returns calls, failures, and avg/max/total duration per host — useful for spotting a degraded downstream dependency without digging through individual traces.

---

## 4. Correlation Timeline

APM spans aren't only visible in the per-request panel — `GET /request-logs-api/correlation/{correlationId}` returns request logs, exception/host-log events, *and* APM spans sharing one correlation id, all in chronological order. This is what powers the dashboard's Correlation Timeline view when a request fans out across multiple log types.

---

## 5. Configuration

APM tracing is enabled by default. Disable it to skip all DB/HttpClient telemetry entirely:

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.EnableApmTracing = false;
});
```

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| **EnableApmTracing** | `bool` | `true` | Enables EF Core and HttpClient tracing. |
| **ApmSpanRetentionDays** | `int?` | `7` | Days to retain APM spans before the retention worker deletes them. `null` keeps them forever. |
| **MaxInMemoryEntries** | `int` | `10_000` | For `InMemory` storage, spans are bounded to roughly 10x this value. |

When disabled, the diagnostics subscriber skips listener registration entirely (zero hookup overhead), and the middleware skips span collection and queuing.

---

## 6. Gotchas

- **Table**: AsGuard creates an `AsGuardApmSpans` table automatically on startup — no manual migration needed. Each span links back to its parent request via `CorrelationId`.
- **Sensitive data**: raw SQL command text and full request URIs are logged as intercepted, to maximize debugging value. Make sure `SensitiveHeaders` covers things like `Authorization` or `X-Api-Key` so they don't leak into logged URLs, and be aware that enabling database-level sensitive logging can put parameter values into captured SQL text.
- **Storage growth**: for persistent providers, spans flush in bulk batches like everything else; for `InMemory`, they're capped relative to `MaxInMemoryEntries` to protect RAM. `ApmSpanRetentionDays` keeps long-term growth in check on persistent providers too.

See **[Configuration Reference](../configuration.md)** for the full options list, and **[Dashboard & Web API](dashboard-api.md)** for the `slow-traces`, `dependencies`, and `correlation` endpoints.
