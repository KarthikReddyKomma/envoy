# Bandwidth Limit Filter (`envoy.filters.http.bandwidth_limit`)

Throttles request and/or response byte-rate on an HTTP stream by metering bytes through a shared token bucket. The filter buffers data internally, releases it in fill-interval slices based on available tokens, applies downstream/upstream watermarks when the internal buffer grows, and optionally reports the imposed delay via response trailers.

Proto: `envoy.extensions.filters.http.bandwidth_limit.v3.BandwidthLimit`.

## Files
- `bandwidth_limit.h/cc` — `FilterConfig`, stats struct, and the `BandwidthLimiter` stream filter.
- `config.h/cc` — factory (`BandwidthLimitFilterConfig`) wiring proto to filter creation; registers the extension.

## Lifecycle
`BandwidthLimiter` derives from `Http::StreamFilter` (`bandwidth_limit.h:122`) so it sits on both decode and encode paths. Heavy lifting is delegated to `Common::StreamRateLimiter`, instantiated lazily on first header call.

Overridden methods:
- `decodeHeaders` (`bandwidth_limit.cc:62`): checks `config.enabled()` and that `enable_mode_` includes `REQUEST`. If yes, increments `request_enabled_` and constructs `request_limiter_` wired with callbacks for high/low watermark, write-to-chain, continue-decoding, and a stats-writer lambda that updates `request_allowed_*` and accumulates `request_delay_`. Always returns `Continue` — the bucket only kicks in for body bytes.
- `decodeData` (`bandwidth_limit.cc:93`): if a `request_limiter_` exists, lazily starts `request_latency_` timespan, bumps `request_incoming_size_`, pushes bytes into the limiter, and returns `StopIterationNoBuffer` so the filter chain blocks until the limiter re-injects data via `injectDecodedDataToFilterChain`.
- `decodeTrailers` (`bandwidth_limit.cc:111`): if the limiter still has buffered bytes (`onTrailers()` returns true) the trailers wait (`StopIteration`). Otherwise it calls `updateStatsOnDecodeFinish` and returns `Continue`.
- `encodeHeaders` (`bandwidth_limit.cc:123`): mirror of decode side for `RESPONSE` mode.
- `encodeData` (`bandwidth_limit.cc:155`): if `enable_response_trailers()` is set and this is `end_stream`, pre-adds an encoded-trailer map via `addEncodedTrailers` before pushing data to the limiter so the delay trailers can be filled in later.
- `encodeTrailers` (`bandwidth_limit.cc:181`): captures the trailer map so `updateStatsOnEncodeFinish` can write delay headers into it; blocks or continues depending on limiter state.
- `encode1xxHeaders` / `encodeMetadata`: pure pass-through (`bandwidth_limit.h:136`, `bandwidth_limit.h:144`).
- `onDestroy` (`bandwidth_limit.cc:240`): calls `destroy()` on both limiters to cancel their timers.

## Decision / logic
- Per-direction activation decided by bit-and of `enable_mode_` with `REQUEST`/`RESPONSE` (`bandwidth_limit.cc:65`, `bandwidth_limit.cc:126`).
- Per-route override: `getConfig()` (`bandwidth_limit.cc:232`) calls `Http::Utility::resolveMostSpecificPerFilterConfig<FilterConfig>`, so each route can swap in its own limit. A per-route config without `limit_kbps` is rejected in `FilterConfig::FilterConfig` (`bandwidth_limit.cc:42`).
- Delay attribution (`bandwidth_limit.cc:79-86`, `141-148`): the stats callback distinguishes "allowed without buffering" (bytes released in same interval) from "enforced" (bytes buffered), incrementing `*_enforced_` and accumulating `request_delay_`/`response_delay_`.
- Trailer emission (`bandwidth_limit.cc:208-223`): on encode finish, writes `bandwidth-request-delay-ms`, `bandwidth-response-delay-ms`, and the `-filter-delay-ms` variants only when the respective value is non-zero. Trailer names can be prefixed via `response_trailer_prefix` (`bandwidth_limit.cc:36-39`).
- Token bucket: one `SharedTokenBucketImpl` per `FilterConfig` instance (`bandwidth_limit.cc:50-52`), sized to `limit_kbps * 1024` bytes and pre-filled to `max_tokens * fill_interval / 1000`. All streams sharing that config share the bucket.

## Configuration
Important proto fields (consumed in `FilterConfig` ctor, `bandwidth_limit.cc:33`):
- `limit_kbps` — kilobytes/sec cap; required when `per_route` is true.
- `fill_interval` — token refill granularity; defaults to `StreamRateLimiter::DefaultFillInterval` (50ms).
- `enable_mode` — `REQUEST`, `RESPONSE`, or `REQUEST_AND_RESPONSE`.
- `runtime_enabled` — runtime feature flag gating `enabled()`.
- `enable_response_trailers` + `response_trailer_prefix` — control delay-trailer emission and optional prefix.

Per-route: `createRouteSpecificFilterConfigTyped` (`config.cc:28`) builds another `FilterConfig` with `per_route=true`; `BandwidthLimiter::getConfig()` picks the most specific one at runtime.

## Stats
Prefix `<stat_prefix>.http_bandwidth_limit.` (`bandwidth_limit.cc:56`). From `ALL_BANDWIDTH_LIMIT_STATS` (`bandwidth_limit.h:32`):
- Counters: `request_enabled`, `response_enabled`, `request_enforced`, `response_enforced`, `request_incoming_total_size`, `response_incoming_total_size`, `request_allowed_total_size`, `response_allowed_total_size`.
- Gauges (Accumulate): `request_pending`, `response_pending`, `request_incoming_size`, `response_incoming_size`, `request_allowed_size`, `response_allowed_size`.
- Histograms (Milliseconds): `request_transfer_duration`, `response_transfer_duration`.

## Factory
`BandwidthLimitFilterConfig` extends `Common::ExceptionFreeFactoryBase` (`config.h:17`). `createFilterFactoryFromProtoTyped` (`config.cc:15`) builds a shared `FilterConfig` and returns a callback that does `callbacks.addStreamFilter(std::make_shared<BandwidthLimiter>(...))`. Registered via `LEGACY_REGISTER_FACTORY` under both `envoy.filters.http.bandwidth_limit` and legacy alias `envoy.bandwidth_limit` (`config.cc:39`).
