# Local Rate Limit Listener Filter (`envoy.filters.listener.local_ratelimit`)

Applies a per-listener token-bucket rate limit to new connections before they reach the network filter chain. Backed by the shared `Filters::Common::LocalRateLimit::LocalRateLimiterImpl`, which refills tokens on a dispatcher timer. Connections that exceed the bucket are closed immediately at `onAccept`; connections are also permitted to bypass when the filter is runtime-disabled. Does not inspect bytes and does not modify `ConnectionSocket` state besides closing the IO handle.

Proto: `envoy.extensions.filters.listener.local_ratelimit.v3.LocalRateLimit`.

## Files
- `local_ratelimit.h` / `local_ratelimit.cc` — `FilterConfig` (shared across workers, holds runtime flag, stats, and the token bucket) and `Filter` (per-connection listener filter). `onData` and `maxReadBytes` are inline no-ops (`local_ratelimit.h:66`, `local_ratelimit.h:69`).
- `config.cc` — `LocalRateLimitConfigFactory`: validates the proto, builds one shared `FilterConfig` on the main-thread dispatcher, and installs the filter via `filter_manager.addAcceptFilter` (`config.cc:24`-`config.cc:37`); registers under `envoy.filters.listener.local_ratelimit` (`config.cc:45`).

## Lifecycle
- `onAccept(cb)` — if `config_->enabled()` (runtime feature flag) returns false, returns `Continue` immediately (`local_ratelimit.cc:32`). Otherwise consults `rate_limiter_.requestAllowed({})` via `FilterConfig::canCreateConnection` (`local_ratelimit.cc:24`). On denial, increments `rate_limited`, closes the socket with `cb.socket().ioHandle().close()`, and returns `StopIteration` (`local_ratelimit.cc:38`-`local_ratelimit.cc:45`). On allow, returns `Continue`.
- `onData(buffer)` — returns `Continue` unconditionally (`local_ratelimit.h:66`).
- `maxReadBytes()` — returns 0 (filter does not peek bytes) (`local_ratelimit.h:69`).

## Decision / logic
- Token bucket configuration is taken from the `token_bucket` message: `fill_interval` (required ms, `PROTOBUF_GET_MS_REQUIRED`), `max_tokens`, and `tokens_per_fill` (default 1) (`local_ratelimit.cc:15`-`local_ratelimit.cc:21`). No descriptors are supplied, so every connection consumes one token from the single global bucket.
- Runtime gating uses `proto_config.runtime_enabled()` wrapped in `Runtime::FeatureFlag` (`local_ratelimit.cc:13`).
- The filter closes the raw IO handle directly; it does not mutate SNI/ALPN/restored addresses.

## Configuration
- `stat_prefix` — appended after the hard-coded `listener_local_ratelimit.` prefix when generating stats (`local_ratelimit.cc:27`).
- `token_bucket.fill_interval` — required, refill period.
- `token_bucket.max_tokens` — bucket capacity.
- `token_bucket.tokens_per_fill` — tokens added per interval (default 1).
- `runtime_enabled` — `RuntimeFeatureFlag` controlling whether rate limiting applies (`local_ratelimit.cc:13`).

## Stats
Rooted under `listener_local_ratelimit.<stat_prefix>.` in the listener scope (`local_ratelimit.cc:27`):
- `listener_local_ratelimit.<stat_prefix>.rate_limited` — incremented each time a connection is dropped because the bucket is empty (`local_ratelimit.cc:42`).
