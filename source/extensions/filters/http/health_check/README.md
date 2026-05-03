# Health Check Filter (`envoy.filters.http.health_check`)

Answers health-check probes (matched by request headers) with server health
status, optionally without ever reaching the upstream. Supports pass-through
and cached pass-through modes.

Proto: `envoy.extensions.filters.http.health_check.v3.HealthCheck`.

## Lifecycle

Decode-only. `decodeHeaders()` (`health_check.cc:53–98`):

1. Match the request against `header_match_data_` via
   `HeaderUtility::matchHeaders`. No match → pass through.
2. Mark the stream as a health check in `StreamInfo`.
3. Decision:
   - `pass_through_mode=false` → always answer locally with the server's
     draining / healthy state.
   - `pass_through_mode=true` →
     - If `cache_time_ms` is set and the cache is fresh, answer from the
       cache.
     - Otherwise forward upstream and (on response) refresh the cache with
       the upstream result.
4. Response code computed from `cluster_min_healthy_percentages` and current
   health; sets `x-envoy-upstream-health-checked-cluster`.

## Cache

`HealthCheckCacheManager` (`health_check.h:56–78`) is an LRU with periodic
invalidation. Stores response code and degraded status so repeat probes
don't walk through the full filter chain.

## Configuration

- `pass_through_mode` — forward to upstream vs answer locally.
- `headers` — match rules identifying the probe.
- `cache_time` — only valid with `pass_through_mode=true`.
- `cluster_min_healthy_percentages` — override health evaluation threshold
  per cluster.

## Files

- `health_check.{h,cc}` — filter and cache manager.
- `config.{h,cc}` — factory.
