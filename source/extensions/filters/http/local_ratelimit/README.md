# Local Rate Limit Filter (`envoy.filters.http.local_ratelimit`)

Per-Envoy token-bucket rate limiter. No RPC, no latency tax. Use when you need
cheap backpressure inside a single node (or per downstream connection).

Proto: `envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit`.

## Token buckets

- **Default bucket** — `token_bucket` (`max_tokens`, `tokens_per_fill`,
  `fill_interval ≥ 50 ms`). Fill rate = `tokens_per_fill / fill_interval`.
- **Per-descriptor buckets** — `descriptors[]`, each with its own
  `token_bucket`. Keys may be literal or wildcard.
- Implemented by `AtomicTokenBucketImpl` inside `RateLimitTokenBucket`
  (`local_ratelimit_impl.cc:68–92`).
- When multiple descriptor buckets match a request, they are consumed in
  **ascending fill-rate order** so the tightest limit fails fast
  (`local_ratelimit_impl.cc:168–170`).
- Dynamic descriptors (wildcard matches) are LRU-tracked up to
  `max_dynamic_descriptors` (default 20)
  (`local_ratelimit_impl.cc:261–292`).

## Request lifecycle (`local_ratelimit.cc`)

1. **`decodeHeaders()` (line 138)**
   - Skip if `!enabled()`.
   - `populateDescriptors()` (line 258) gathers descriptors from the route
     / virtual host `rate_limits` actions, respecting `stage`.
2. **`requestAllowed()` (line 230)**
   - Picks the per-connection limiter (if
     `local_rate_limit_per_downstream_connection=true`) or the shared one.
   - Calls `LocalRateLimiterImpl::requestAllowed()`
     (`local_ratelimit_impl.cc:148`), which consumes from the matching
     descriptor buckets then the default bucket.
   - Returns `{allowed, token_bucket_context, x_ratelimit_option}`.
3. **Enforce vs shadow**
   - Shadow descriptor → increment `shadow_mode_` and continue
     (`local_ratelimit.cc:184–186`).
   - Enforced and `!allowed` → `sendLocalReply()` with
     `status` (default 429) and apply `response_headers_to_add`
     (`local_ratelimit.cc:196–202`).
   - Allowed → apply `request_headers_to_add` (`local_ratelimit.cc:190`) and
     continue.
4. **`encodeHeaders()` (line 208)** — if `enable_x_ratelimit_headers`, stamps
   `x-ratelimit-limit` / `-remaining` / `-reset` from
   `token_bucket_context_` (lines 214–224).

## Virtual host behaviour

`vh_rate_limits` ∈ {`OVERRIDE`, `INCLUDE`, `IGNORE`}
(`local_ratelimit.h:304–316`).

## Per-connection mode

`local_rate_limit_per_downstream_connection=true` stores
`PerConnectionRateLimiter` in `FilterState` with `LifeSpan::Connection`
(`local_ratelimit.cc:236–256`); buckets are shared across all requests on the
connection.

## Stats

- `enabled` — request evaluated.
- `enforced` — request was rate-limited and enforced.
- `rate_limited` — request denied.
- `ok` — request allowed.
- `shadow_mode` — would-have-been-denied in shadow mode.

## Files

- `local_ratelimit.{h,cc}` — filter.
- Shared bucket logic at
  `source/extensions/filters/common/local_ratelimit/`.
