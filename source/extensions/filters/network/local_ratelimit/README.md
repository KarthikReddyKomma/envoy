# Local Rate Limit (`envoy.filters.network.local_ratelimit`)

A connection-oriented L4 rate limiter that uses an in-process token bucket to decide, at `onNewConnection()` time, whether to accept or forcibly close the downstream connection. It is a `Network::ReadFilter` only (no `onWrite`); a single token is consumed per new connection. Multiple listeners / filter instances can optionally share a common bucket via the `share_key` field, which is resolved through a server-wide singleton so that token bucket state is shared across all `Config` objects with the same `(share_key, token_bucket)` pair.

Proto: `envoy.extensions.filters.network.local_ratelimit.v3.LocalRateLimit`.

## Files
- `config.h` / `config.cc` — `LocalRateLimitConfigFactory`. `FactoryBase`-derived factory that builds the per-filter `Config` using main-thread dispatcher, the listener `Stats::Scope`, `Runtime::Loader`, and `Singleton::Manager` from the `FactoryContext` (`config.cc:13-22`). Registers via `REGISTER_FACTORY` (`config.cc:27-28`).
- `local_ratelimit.h` / `local_ratelimit.cc` — `Filter`, `Config`, and `SharedRateLimitSingleton`. `Filter` is a `Network::ReadFilter`; `Config` owns the rate limiter and stats; the singleton maps `(share_key, token_bucket)` to a shared `LocalRateLimiterImpl` so multiple `Config`s can coalesce onto one bucket.

## Lifecycle
- `Filter::onNewConnection()` (`local_ratelimit.cc:99-117`) — the sole decision point. If `Config::enabled()` returns false (runtime flag off), return `Continue` without touching the bucket. Otherwise call `Config::canCreateConnection()` (`local_ratelimit.cc:97`) which forwards to `LocalRateLimiterImpl::requestAllowed({}).allowed`. On denial: increment `rate_limited_` counter, set `CoreResponseFlag::UpstreamRetryLimitExceeded` on `StreamInfo`, close the connection with `ConnectionCloseType::NoFlush` and local-close detail `"local_ratelimit_close_over_limit"`, then return `StopIteration`.
- `Filter::onData()` (`local_ratelimit.h:104-106`) — no-op returning `Continue`. All enforcement happens at connection creation.
- `initializeReadFilterCallbacks()` (`local_ratelimit.h:108-110`) — stashes the callbacks pointer used later to access the connection and its `StreamInfo`.

## Decision / logic
- Shared bucket resolution: `SharedRateLimitSingleton::get()` (`local_ratelimit.cc:20-49`). If `share_key` is empty, create a per-filter bucket and return a `nullptr` key (never shared). Otherwise look up `Key{share_key, token_bucket}`; on hit, lock the `weak_ptr` and reuse (`local_ratelimit.cc:41-47`); on miss, call `create_fn` and insert (`local_ratelimit.cc:34-39`). The `Key` uses `MessageDifferencer::Equivalent` for token-bucket equality so that semantically identical buckets share state (`local_ratelimit.h:48-51`).
- `Config` ctor (`local_ratelimit.cc:62-82`) fetches / creates the `SharedRateLimitSingleton`, then calls `get()` with a creator lambda that constructs `Filters::Common::LocalRateLimit::LocalRateLimiterImpl` with `fill_interval`, `max_tokens`, and `tokens_per_fill` (defaulting to 1) from the proto token bucket.
- `Config::~Config` (`local_ratelimit.cc:84-90`) drops its `rate_limiter_` reference first so the singleton's `weak_ptr` observes expiration, then calls `removeIfUnused()` (`local_ratelimit.cc:51-60`) which erases the map entry only if the weak reference has expired.
- Filter factory (`config.cc:13-22`) wraps a single `ConfigSharedPtr` in a lambda that adds one `Filter` to every `FilterManager`; all filter instances in one listener thus share the same `Config` (and therefore the same underlying bucket when `share_key` is configured).

## Configuration
- `stat_prefix` (required) — used to build the stats namespace `local_rate_limit.<stat_prefix>` (`local_ratelimit.cc:92-95`).
- `token_bucket` (required) — `envoy.type.v3.TokenBucket`. `fill_interval` is fetched with `PROTOBUF_GET_MS_REQUIRED`; `tokens_per_fill` defaults to 1.
- `runtime_enabled` — fractional percent feature flag wrapped by `Runtime::FeatureFlag enabled_` (`local_ratelimit.h:86`). When disabled, the filter is a pass-through.
- `share_key` — optional string. When set, causes this filter to share a bucket with every other network `LocalRateLimit` whose `(share_key, token_bucket)` matches via proto `Equivalent`.

## Stats
Emitted under `local_rate_limit.<stat_prefix>.`:
- `rate_limited` (counter) — incremented each time a connection is closed because the bucket had no tokens (`local_ratelimit.cc:106`). The macro `ALL_LOCAL_RATE_LIMIT_STATS` in `local_ratelimit.h:22` defines the single counter.
