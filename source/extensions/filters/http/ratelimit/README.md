# Global Rate Limit Filter (`envoy.filters.http.ratelimit`)

Enforces global rate limits by calling an external rate-limit service
(Lyft RLS protocol, gRPC). For an in-process, per-node limiter see
`../local_ratelimit/`.

Proto: `envoy.extensions.filters.http.ratelimit.v3.RateLimit`.

## How limits are defined

Limits are keyed by **descriptors** — lists of `{key, value}` entries. Where
descriptors come from:

1. Route / virtual host `rate_limits[*].actions` — the canonical source.
2. `RateLimitPerRoute` (per-route `typed_per_filter_config`) — per-route
   override with the highest precedence.
3. `RateLimitConfig` — global `typed_per_filter_config`.
4. Virtual host limits are included / overridden / ignored based on
   `vh_rate_limits`.

## Request lifecycle (`ratelimit.cc`)

1. **`decodeHeaders()` (line 137)**
   - Exit if `!enabled()`.
   - `initiateCall()` (line 52) builds and fires the RLS request.
2. **`populateRateLimitDescriptors()` (line 71)**
   - Pulls descriptors from the matched route / virtual host,
     skipping entries whose `disable_key` is disabled in runtime
     (`ratelimit.cc:310`).
   - Respects `stage` — only actions with the matching stage run.
3. **`client_->limit(...)` (line 65)** — async gRPC call. State = `Calling`;
   return `StopIteration` (line 144).
4. **`complete()` (line 217)** — dispatch on `LimitStatus`:
   - **OK** — `ok_` stat, `continueDecoding()`.
   - **OverLimit** — set `x-envoy-ratelimited`, apply RLS-provided response
     headers, `sendLocalReply()` with `rate_limited_status` (default 429)
     at lines 276–284.
   - **Error** — `failure_mode_deny=true` → deny with `status_on_error`
     (default 500). Otherwise fail-open and `continueDecoding()`
     (lines 286–291).
5. **`encodeHeaders()` (line 172)** — `populateResponseHeaders()` adds
   `x-envoy-ratelimit-limit` / `-remaining` / `-reset` per
   draft-ietf-httpapi-ratelimit-headers-03.

Async lifetime: `OnStreamDoneCallBack` (`ratelimit.h:296`) keeps the gRPC
client alive via `keepAlive()` even if the filter is destroyed mid-flight.

## Response contract

- `rate_limited_status` — status for 429s. Default 429.
- `status_on_error` — status when the RLS call fails. Default 500.
- `rate_limited_as_resource_exhausted` — gRPC bridge: returns
  `RESOURCE_EXHAUSTED` instead of `UNAVAILABLE`.
- `enable_x_envoy_ratelimited_header` — adds `x-envoy-ratelimited` header on
  overlimit (`ratelimit.cc:255–260`).
- Body + headers from the RLS response are applied to the local reply.

## Timeout / stage / disable

- Timeout configured on the underlying RLS gRPC service.
- `stage` (uint64) selects which route actions to evaluate
  (`ratelimit.cc:309`).
- `disable_key` gated by the runtime flag
  `ratelimit.{disable_key}.http_filter_enabled`.

## Stats

`ok`, `error`, `over_limit`, `failure_mode_allowed`; response-code stats via
`HttpContext::codeStats()` (`ratelimit.cc:254`).

## Files

- `ratelimit.{h,cc}` — filter.
- `config.{h,cc}` — factory, RLS client wiring.
- Shared client: `source/extensions/filters/common/ratelimit/`.
