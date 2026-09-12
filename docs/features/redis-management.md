# Redis Management

AsGuard can add a **Redis** tab to the dashboard: a key browser, a value inspector/editor, and live server stats. No extra service, no separate UI — and usually no configuration beyond one flag.

It's the only feature flag that defaults to `false`. Every other `Enable*` option defaults to `true` because it's observability data; this one reads and writes your *application* data, so it stays opt-in even across upgrades.

---

## 1. Enabling It

```csharp
builder.Services.AddRequestLogging(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("Logging")!;
    options.DashboardUsername = "ops";
    options.DashboardPassword = builder.Configuration["AsGuard:Password"]!;

    options.EnableRedisManagement = true;
});
```

That's usually the whole setup — AsGuard discovers your Redis connection itself.

---

## 2. Connection Discovery

AsGuard prefers to **reuse a connection your app already built**, and treats a connection string as a last resort. That matters for **Azure Cache for Redis under Microsoft Entra ID**, where there's no password at all — just a refreshing access token set on the `ConfigurationOptions` your app already constructed. A connection string can't reproduce that, so reusing the app's own multiplexer is the only thing that works (same story for any managed Redis using token/certificate auth).

Discovery runs once, lazily, on first tab open. Each rung is fault-isolated — one failing source doesn't suppress the others:

| # | Source | Typical setup |
| :-- | :--- | :--- |
| 1 | `IAsGuardRedisConnectionProvider` in DI | Explicit escape hatch; always wins |
| 2 | `IConnectionMultiplexer` in DI | `AddSingleton<IConnectionMultiplexer>`, Aspire `AddRedisClient` — **including Entra ID** |
| 3 | Keyed `IConnectionMultiplexer` | Aspire `AddKeyedRedisClient("cache")`, multi-instance hosts |
| 4 | `AddStackExchangeRedisCache` config | Hosts that only wired `IDistributedCache` |
| 5 | `RequestLoggingOptions.RedisConnections` | No Redis client registered at all |

Connections found on rungs 1–4 are **borrowed**: AsGuard never disposes or mutates them (disposing one would take your app's cache down with the dashboard). Only rung 5 connections are owned by AsGuard. The UI shows which rung each connection came from.

```csharp
// Rung 1: supply connections yourself (per-tenant multiplexers, etc.)
public sealed class TenantRedisProvider(TenantRegistry tenants) : IAsGuardRedisConnectionProvider
{
    public IEnumerable<AsGuardRedisConnection> GetConnections() =>
        tenants.All.Select(t => new AsGuardRedisConnection(t.Name, t.Multiplexer));
}
builder.Services.AddSingleton<IAsGuardRedisConnectionProvider, TenantRedisProvider>();

// Rung 5: only needed when nothing is registered in DI
options.RedisConnections.Add(new RedisConnectionDescriptor
{
    Name = "sessions",
    ConnectionString = "localhost:6379,ssl=False"
});
```

Rung-5 connections get `abortConnect=false` (an unreachable server shows as *offline* instead of throwing at startup) and a client name of `AsGuard-Dashboard`. They do **not** get `allowAdmin` — see [Flushing](#5-flushing-a-database). If nothing is found, the tab explains what it looked for rather than failing — same as the health-check visualizer with no health checks registered.

---

## 3. Key Browser

Keys are enumerated with **`SCAN`**, never `KEYS` — `KEYS` blocks the whole Redis instance, which is an outage on a large production cache. This is structural: the only type that issues Redis commands never references `KEYS`, and AsGuard checks the server version and refuses to enumerate rather than let StackExchange.Redis silently fall back to `KEYS` on servers older than Redis 2.8.

Paging is **cursor-based**, not page-numbered — a `SCAN` cursor is opaque and a filtered traversal has no honest total, so the UI offers *Next*/*Restart* instead of a fake page count. Each row shows key, type, and remaining TTL, pipelined in a handful of round trips per page.

**Binary keys** (not valid UTF-8) are shown base64-encoded and tagged `base64` rather than mangled — so they stay addressable.

---

## 4. Inspector & Editor

| Type | View | Edit |
| :--- | :--- | :--- |
| `string` | Full value, truncated at `MaxRedisValueBytes` | Replace value (TTL preserved via `keepTtl`) |
| `hash` | Fields/values via `HSCAN` | Set or delete a field |
| `list` | Elements with index via `LRANGE` | Set by index; remove by value |
| `set` | Members via `SSCAN` | Add or remove a member |
| `zset` | Members with scores via `ZRANGE` | Add/update a score, or remove a member |

TTL can be set to any positive number of seconds or cleared to make a key persistent.

Two safeguards:
- **A truncated value can't be saved.** If a string exceeds `MaxRedisValueBytes`, only part loads, and writing it back would destroy the rest — the editor goes read-only instead.
- **Type changes are detected.** If a key was retyped since the inspector loaded it, the write is refused with an explanation.

Collections load a window at a time (`EntryCount`, default 100, max 500) so a million-element list can't exhaust memory; hash fields and set members accept a `MATCH` filter.

---

## 5. Flushing a Database

`FLUSHDB` requires **all four** gates:

1. `EnableRedisManagement` is `true`.
2. `AllowRedisFlushDatabase` is `true` — a separate opt-in, not implied by enabling the module.
3. The connection was built with `allowAdmin=true`. AsGuard doesn't work around this — it's your application's own decision, and the UI hides the control (with an explanation) when the flag is absent.
4. The caller types the exact confirmation phrase for the target database (e.g. `FLUSH db0`) in the request body — a client-side dialog alone can't stop a replayed or hand-crafted request.

The flush endpoint also requires an explicit connection name (other operations fall back to the first discovered connection; flush must not, to avoid hitting the wrong database on a multi-connection host). `FLUSHALL` is never issued, and there's no filtered bulk delete. Every flush is logged at **warning** level.

---

## 6. Live Statistics

The Redis tab streams over the **dashboard's existing SSE connection** — no second stream, no polling — and only samples `INFO`/`PING` when someone is actually subscribed (a dashboard tab open on the group), so idle apps don't hit Redis on a timer for nobody.

| Tile | Shows |
| :--- | :--- |
| Latency | `PING` round trip (client-observed) |
| Memory | `used_memory`, % of `maxmemory` or peak, RSS |
| Ops/sec | `instantaneous_ops_per_sec`, network in/out |
| Hit rate | Interval + lifetime hit rate |
| Clients | Connected/max clients, blocked, new connections/sec |
| Keys | Total keys, expiry/eviction **rates** (not lifetime counters) |

A health strip below the tiles surfaces version, uptime, replication status, RDB/AOF status, fragmentation, and rejected connections — only when something needs attention. Four sparklines (latency, throughput, memory, hit rate) chart the last 120 samples client-side per session (history resets on page reload, since sampling is demand-driven).

An expandable **Command & Latency Diagnostics** panel (fetched on demand, not on the sampling tick — too expensive to run every few seconds) adds:
- `INFO commandstats` — calls, mean time, rejected/failed counts per command.
- `INFO latencystats` — p50/p99/p99.9 per command (Redis 7+ only).
- `SLOWLOG` — commands that breached the slow-log threshold. Requires `allowAdmin=true`, same as flush.

> Slow log entries record command arguments verbatim, so by default AsGuard shows only the command, key, and argument count. Set `RedisSlowlogIncludeArguments = true` to see full arguments.

---

## 7. Security

The module inherits the dashboard's protections rather than inventing new ones — every route lives under the already-authenticated `/request-logs-api` prefix:

- Same Basic Auth / session cookie, login rate limiting, and `X-AsGuard-Request` CSRF header on mutating calls.
- `/request-logs-api` is excluded from AsGuard's own request logging, so inspected cache values never end up persisted (or broadcast) via `LogResponseBody`.
- Routes are only mapped when `EnableRedisManagement` is true — a default install has no Redis route at all, not a 403.
- `RedisManagementReadOnly = true` is enforced server-side in one shared path every mutating call passes through, so it can't be bypassed by a forgotten check.

---

## 8. Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| **EnableRedisManagement** | `bool` | `false` | Enables the Redis tab, key browser, inspector, and live stats. |
| **RedisManagementReadOnly** | `bool` | `false` | Browse-only: edits, TTL changes, deletes, and flushes rejected server-side. |
| **AllowRedisFlushDatabase** | `bool` | `false` | Allows `FLUSHDB`. Also requires `allowAdmin` and typed confirmation. |
| **RedisConnections** | `IList<RedisConnectionDescriptor>` | empty | Explicit endpoints, used only when nothing is discoverable from DI. |
| **RedisScanPageSize** | `int` | `100` | Keys per `SCAN` round trip. Capped at 1000/request. |
| **MaxRedisValueBytes** | `int` | `262144` | Byte cap on an inspected value before truncation. |
| **RedisStatsSampleInterval** | `TimeSpan` | `5 seconds` | Interval for the `INFO` sample pushed over SSE. |
| **RedisSlowlogIncludeArguments** | `bool` | `false` | Shows full slow-log arguments. Off by default. |

---

## 9. REST API

All routes sit under the authenticated `/request-logs-api` prefix and are excluded from the host's OpenAPI document.

| Method | Route | Purpose |
| :--- | :--- | :--- |
| GET | `/request-logs-api/redis/connections` | Discovered connections, databases, capability flags |
| GET | `/request-logs-api/redis/keys` | One `SCAN` page |
| GET | `/request-logs-api/redis/keys/value` | Read one key |
| PUT | `/request-logs-api/redis/keys/value` | Write/remove a value, field, element, or member |
| PUT | `/request-logs-api/redis/keys/ttl` | Set or clear an expiry |
| DELETE | `/request-logs-api/redis/keys` | Delete one key |
| POST | `/request-logs-api/redis/flush` | Flush one database, with confirmation |
| GET | `/request-logs-api/redis/stats` | One-shot `INFO` snapshot |
| GET | `/request-logs-api/redis/diagnostics` | Command stats, latency percentiles, slow log |

Keys travel as a query parameter or body field, never a route segment (Redis keys routinely contain `/`, `%`, `#`). Pass `keyEncoding=base64` for non-UTF-8 keys.

Status codes: `400` malformed/unsupported server, `403` refused action (read-only, flush disabled, missing confirmation), `404` unknown connection, `503` reachable-but-failing server.

---

## 10. Dependency Note

Enabling this module pulls in `StackExchange.Redis`, floor-pinned at **2.8.58** on purpose — NuGet resolves to the highest version across your dependency graph, so an app already on a newer version keeps it, and one on 2.8 isn't force-upgraded. If a host unifies to an incompatible major version, the module disables itself with a warning instead of faulting the app at startup.
