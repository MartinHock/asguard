# Dashboard & Web API Reference

AsGuard ships a built-in Razor dashboard plus a secure REST API and Server-Sent Events (SSE) stream. This page covers the UI, authentication, and the API surface behind it.

---

## 1. Dashboard UI

Loads at your configured `DashboardRoute`, no external assets required:

- **Request Logs** — HTTP requests with status, timing, user, and expandable detail (headers, query params, redacted bodies). Drill into a correlation ID's full timeline, or copy a request as `curl`.
- **Exceptions & Events** — unhandled crashes, manually pushed errors, and logger events with interactive stack traces.
- **Endpoint Leaderboard** — ranks endpoints by `p50`/`p95`/`p99`, error rate, and volume.
- **Top Users Leaderboard** — ranks users (by `UserId`) by volume, error rate, avg duration, and p95.
- **Slow Traces** — slowest DB/HTTP spans in a selected time range, plus outbound dependency stats by host.
- **Alert History** — fired queue-pressure, exception-spike, and failure alerts.
- **Host Metrics & Health** — process metrics, health-check status cards, and a health history trend chart.
- **Filtering** — search, level/method/status filters, correlation ID, user ID, UTC date range.
- **Data Controls** — clear or selectively delete logs, export the current view as CSV/JSON.

---

## 2. Authentication

The dashboard, REST API, and SSE stream all sit behind AsGuard's auth middleware:

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.DashboardRoute = "/request-logs-ui";
    options.DashboardUsername = builder.Configuration["AsGuard:Username"]!;
    options.DashboardPassword = builder.Configuration["AsGuard:Password"]!;
});
```

**Login flow:** unauthenticated browser access redirects (`302`) to a built-in login page (`/request-logs-login`), which posts to `POST /request-logs-login-api`:

- Correct credentials → `200 OK`, sets an `HttpOnly`, Data Protection–encrypted `AsGuardDashboardAuth` session cookie (lifetime = `DashboardSessionTimeout`, sliding renewal).
- Incorrect credentials → `401 Unauthorized`.
- Too many failed attempts from one IP (`LoginRateLimitMaxAttempts` / `LoginRateLimitWindow`) → `429` with `Retry-After`.
- Login/logout POSTs and cookie-authenticated `DELETE` calls must send `X-AsGuard-Request: 1` (CSRF proof). Basic-Auth clients are exempt.
- `POST /request-logs-logout-api` clears the session cookie.

The REST API and SSE stream still accept plain `Authorization: Basic` headers (or the session cookie) for every request, so curl/Postman/CI keep working unchanged. Query-string credentials (`?username=&password=`) were **removed** — they leaked into proxy/access logs.

**Startup validation** fails fast rather than exposing diagnostics by accident:
1. If `DashboardRoute` is non-empty, both `DashboardUsername` and `DashboardPassword` must be set, or the app throws at startup.
2. `"admin"` / `"admin"` is explicitly rejected.
3. `ConnectionString` (non-`InMemory` providers), `SamplingRate` (`0.0`–`1.0`), and `BatchSize`/`ExceptionBatchSize` (`>= 1`) are validated at startup too.

---

## 3. REST API

All endpoints require the same Basic Auth / session credentials as the dashboard.

| Method | Route | Description |
| :--- | :--- | :--- |
| GET | `/request-logs-api` | Query and page request logs. |
| GET | `/request-logs-api/{id}` | Full request log detail (headers, bodies). |
| DELETE | `/request-logs-api` | Delete request logs matching a filter (or clear all). |
| GET | `/request-logs-api/stats` | Queue depths and counter metrics. |
| GET | `/request-logs-api/exceptions` | Query and page exceptions/logger events. |
| GET | `/request-logs-api/exceptions/{id}` | Full stack trace and metadata. |
| GET | `/request-logs-api/exceptions/summary` | Counts grouped by severity. |
| GET | `/request-logs-api/exceptions/trends` | Exception timeline (hourly/daily). |
| DELETE | `/request-logs-api/exceptions` | Delete exceptions matching a filter (or clear all). |
| GET | `/request-logs-api/leaderboard` | Endpoint leaderboard (percentiles, error rate). |
| GET | `/request-logs-api/users/leaderboard` | Per-user leaderboard. |
| GET | `/request-logs-api/apm/slow-traces` | Slowest DB/HttpClient spans in a range. |
| GET | `/request-logs-api/exceptions/groups` | Exceptions grouped by type ("issues") in a range. |
| GET | `/request-logs-api/apm/dependencies` | Outbound HTTP dependency stats by host. |
| GET | `/request-logs-api/correlation/{correlationId}` | Everything captured for one correlation id (logs, events, APM spans). |
| GET | `/request-logs-api/alerts` | Fired alert history. |
| GET | `/request-logs-api/host-metrics` | Sampled host/process metrics history. |
| GET | `/request-logs-api/health` | Run registered health checks, return current status. |
| GET | `/request-logs-api/health/history` | Sampled health-check history. |
| GET | `/request-logs-api/host-environment` | Uptime, OS, .NET version, CPU cores. |
| POST | `/request-logs-login-api` | Authenticate and issue a session cookie (not Basic-Auth protected — it *is* the login). |

Redis management endpoints (only mapped when `EnableRedisManagement` is true) are documented in **[Redis Management](redis-management.md)**.

### Key query parameters

**`GET /request-logs-api`**: `PageIndex`, `PageSize` (1–100), `Search`, `Method`, `StatusCode`, `FromUtc`, `ToUtc`, `UserId`, `ExceptionsOnly`, `IncludeDetails`, `SkipTotalCount`.

**`GET /request-logs-api/exceptions`**: same paging/date/search params, plus `Level`, `CorrelationId`, `IncludeDetails`.

**`GET /request-logs-api/leaderboard`**: `Range` (`15m`/`1h`/`6h`/`24h`/`7d`, default `1h`), `PageIndex`, `PageSize`, `SortBy` (`p50`/`p95`/`p99`/`errorRate`/`volume`/`path`/`method`), `SortDir` (`asc`/`desc`).

**`GET /request-logs-api/users/leaderboard`**: same as above with `SortBy` = `volume`/`errorRate`/`avgDuration`/`p95`/`userId`.

**`GET /request-logs-api/apm/slow-traces`**: `Range`, `Limit` (1–100, default 20). A relative top-N ranking within the window, not an absolute "slow" threshold.

**`GET /request-logs-api/apm/dependencies`**: `Range`. Returns calls, failures, avg/max/total duration grouped by target host.

**`GET /request-logs-api/correlation/{correlationId}`**: up to 20 request logs, up to 100 exception/host-log events, and all APM spans for the id, in chronological order. `404` if nothing was captured or `EnableDetailedLogEndpoint` is disabled.

---

## 4. Response Schemas

**Request log detail** (`GET /request-logs-api/123`):
```json
{
  "id": 123,
  "correlationId": "8f2d59ac-b49d-47fe-902e-b615177894a8",
  "method": "POST",
  "path": "/api/users/register",
  "requestHeaders": "Authorization: [REDACTED]\r\n",
  "requestBody": "{\"email\":\"demo@asguard.io\",\"password\":\"[REDACTED]\"}",
  "statusCode": 201,
  "durationInMilliseconds": 42.15,
  "occurredOnUtc": "2026-05-22T13:15:00Z",
  "userId": "user_demo_9912",
  "tags": "public,auth",
  "metadataJson": "{\"tenantId\":\"emea-77\"}"
}
```

**Diagnostic stats** (`GET /request-logs-api/stats`) — useful for Datadog/Prometheus scraping:
```json
{
  "requestQueueDepth": 0,
  "enqueuedRequests": 45102,
  "droppedRequests": 0,
  "persistedRequests": 45102,
  "persistenceFailures": 0,
  "apmSpanPersistenceFailures": 0,
  "broadcastFailures": 0,
  "activeLiveConnections": 2
}
```
*`activeLiveConnections` tracks active SSE client channels (renamed from `activeSignalRConnections` in 0.3.0).*

**Leaderboard** (`GET /request-logs-api/leaderboard`):
```json
{
  "fromUtc": "2026-05-31T09:00:00Z",
  "toUtc": "2026-05-31T10:00:00Z",
  "totalRequests": 421,
  "pageIndex": 1,
  "pageSize": 20,
  "items": [
    { "method": "GET", "path": "/api/orders/{id}", "volume": 87, "errorRate": 4.6, "p50": 12.4, "p95": 93.1, "p99": 140.6 }
  ]
}
```

**Alert history** (`GET /request-logs-api/alerts`) — capped by `MaxAlertHistoryStored` (default 200):
```json
[
  { "type": "queue-pressure:request", "message": "request log queue depth reached 8200/10000.", "occurredOnUtc": "2026-05-31T09:41:02Z" }
]
```

---

## 5. Live SSE Stream

Real-time updates at `/request-logs-stream`. The dashboard connects with `EventSource` and `withCredentials: true` so the session cookie is sent automatically; Basic Auth headers work too.

```text
GET /request-logs-stream?groups=logs,exceptions
```

- `groups` (optional): comma-separated `logs`, `exceptions`. Omit to subscribe to everything (including `redis`, when enabled).
- A `: ping\n\n` heartbeat is sent every 15 seconds to keep proxies/load balancers from closing idle connections.

Messages are JSON envelopes with a `method` and a `payload` array:
```text
data: {"method":"NewLogs","payload":[{"id":124,"method":"GET","path":"/api/greet","statusCode":200,"durationInMilliseconds":2.8}]}
```

Broadcast queue behavior is tunable:
```csharp
builder.Services.AddRequestLogging(options =>
{
    options.BroadcastQueueCapacity = 1_000;              // Default: 2_048
    options.BroadcastOverflowPolicy = BroadcastOverflowPolicy.DropOldest;
    options.BroadcastDetailMode = BroadcastDetailMode.SummaryOnly; // or Full
});
```

---

## 6. Other Diagnostics Endpoints

- **Health status** (`GET /request-logs-api/health`): `overallStatus`, `totalDurationMilliseconds`, `entries[]`, `healthChecksConfigured` (`false` when nothing is registered).
- **Host environment** (`GET /request-logs-api/host-environment`): sanitized system info — `machineName`, `osVersion`, `frameworkDescription`, `processorCount`, `environment`, `uptime`.
