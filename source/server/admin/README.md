# Admin Console

> Documentation for Envoy's admin / management HTTP interface.
> Source lives in `source/server/admin/`. The public contract is
> `envoy/server/admin.h`; the out-of-network async variant is
> `source/exe/admin_response.{h,cc}`.

The admin console is a small HTTP server that runs **inside** the Envoy process on a
dedicated listener. It exposes operational endpoints — `/stats`, `/clusters`,
`/config_dump`, `/ready`, `/server_info`, `/logging`, `/quitquitquit`, and many more — for
inspecting and controlling a running Envoy.

## The big idea: reuse, don't reinvent

Envoy does **not** write a bespoke HTTP server for admin. Instead, `AdminImpl` impersonates
a full HTTP Connection Manager (HCM) configuration and stands up the *standard* Envoy HTTP
stack on a synthetic listener. This is why `AdminImpl` implements so many interfaces at once
(`admin.h`):

```cpp
class AdminImpl : public Admin,
                  public Network::FilterChainManager,
                  public Network::FilterChainFactory,
                  public Http::FilterChainFactory,
                  public Http::ConnectionManagerConfig,
                  public std::enable_shared_from_this<AdminImpl>,
                  Logger::Loggable<Logger::Id::admin> { ... };
```

Most of the `ConnectionManagerConfig` overrides return null/fixed values (no tracing, no
routing, fixed 100ms drain, server header `OVERWRITE`). The payoff is that the admin server
reuses Envoy's battle-tested codec, filters, and listener machinery for free.

## How a request flows (one sentence each)

1. A connection hits the admin listener; `AdminImpl` supplies a single TCP filter chain
   whose network filter is a `Http::ConnectionManagerImpl`.
2. The HCM installs one terminal HTTP filter, `AdminFilter`, which also implements
   `AdminStream` (the per-request context handlers receive).
3. On end-of-stream, `AdminFilter::onComplete()` calls `AdminImpl::makeRequest()` to resolve
   the URL prefix to a handler, then drives the streaming `start()`/`nextChunk()` protocol,
   encoding each chunk back to the client.

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | Architecture: the synthetic HCM, handler registry, the streaming `Request` model, the three execution paths (network / in-process / out-of-network), and design patterns. |
| `CLASS_HIERARCHY.md` | UML diagrams for `AdminImpl`, the `Request`/`AdminStream` types, handler classes, stats streaming, and `AdminResponse`. |
| `admin.md` | (existing) Deep dive on `AdminImpl`. |
| `stats_handler.md` | (existing) The `/stats` endpoint and `StatsRequest`. |
| `config_dump_handler.md` | (existing) The `/config_dump` endpoint. |
| `prometheus_stats.md` | (existing) Prometheus exposition format. |

## Two registration APIs

| API | For |
|-----|-----|
| `addHandler(prefix, help, HandlerCb, removable, mutates_state, params)` | One-shot handlers that build their whole response into a buffer. |
| `addStreamingHandler(prefix, help, GenRequestFn, ...)` | Streaming handlers that produce a `Request` and emit chunk-by-chunk. |

`addHandler` is just sugar: it wraps the `HandlerCb` in a `RequestGasket` and calls
`addStreamingHandler`, so there is a single code path internally.

## Safety properties worth knowing

- **State-mutating endpoints are POST-only** (flagged `mutates_server_state_` at
  registration, enforced centrally in `makeRequest`). Examples: `/quitquitquit`,
  `/runtime_modify`, `/drain_listeners`.
- **Prefixes are validated against XSS** — registration rejects prefixes containing
  `&"'<>?:` because the admin page renders HTML.
- **Admin survives overload** — it uses the *null* overload manager and bypasses the global
  connection limit, so you can still reach `/stats` and `/quitquitquit` when the data plane
  is shedding load.
