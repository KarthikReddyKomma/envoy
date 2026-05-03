# Cache v2 Filter (`envoy.filters.http.cache_v2`)

A reverse-proxy HTTP cache that attempts to satisfy each request from a pluggable `HttpCache` backend before falling back to an upstream fetch. The filter runs the cache lookup fully async, streams body bytes from the cache source (or upstream) back to the downstream client in bounded chunks, honors Range responses, and coalesces concurrent requests via `CacheSessions`.

Proto: `envoy.extensions.filters.http.cache_v2.v3.CacheV2Config`.

## Files
- `cache_filter.h/cc` — `CacheFilterConfig` and the `CacheFilter` stream filter itself.
- `cache_sessions.{h,cc}` / `cache_sessions_impl.{h,cc}` — request coalescing and subscriber bookkeeping.
- `http_cache.{h,cc}`, `http_source.h`, `cache_entry_utils.{h,cc}` — backend interface + entry model.
- `upstream_request.h` / `upstream_request_impl.{h,cc}` — wraps `Http::AsyncClient` for cache-managed upstream fetches.
- `cacheability_utils.{h,cc}`, `cache_headers_utils.{h,cc}`, `cache_custom_headers.{h,cc}` — RFC 7234 cacheability rules and header helpers.
- `range_utils.{h,cc}` — byte-range math (`AdjustedByteRange`).
- `cache_policy.h`, `cache_progress_receiver.h` — extension points.
- `stats.{h,cc}` — `CacheFilterStats` interface + generator.
- `key.proto` — cache key proto.
- `config.h/cc` — `CacheFilterFactory` and registration.

## Lifecycle
`CacheFilter` extends `Http::PassThroughFilter` and `Http::DownstreamWatermarkCallbacks` (`cache_filter.h:55-57`). It owns both decode and encode sides.

Overrides:
- `setDecoderFilterCallbacks` (`cache_filter.cc:60`): registers itself for downstream watermark events before delegating to the pass-through base.
- `decodeHeaders(headers, end_stream)` (`cache_filter.cc:103`):
  - If the proto was `disabled` (no cache backend) → `Continue` (pass through).
  - If request has a body/trailers (`!end_stream`) → records `Uncacheable` and `Continue`.
  - Calls `CacheabilityUtils::canServeRequestFromCache(headers)` — on failure records `Uncacheable` and continues.
  - Resolves the upstream cluster (route cluster or `override_upstream_cluster`). Missing route → `sendNoRouteResponse` (404, `cache_no_route`). Missing cluster in thread-local CM → `sendNoClusterResponse` (503, `cache_no_cluster`).
  - Builds an `UpstreamRequestImplFactory` + `ActiveLookupRequest`, kicks off `cacheSessions().lookup(...)` with a `cancelWrapped` callback into `onLookupResult`, and returns `StopIteration` to hold the filter chain.
- `encodeHeaders(headers, _)` (`cache_filter.cc:211`): when a lookup result arrived and the filter is feeding cached data out via `decoder_callbacks_->encodeHeaders`, this observes the response on the way out and is a no-op (`Continue`). If an external local reply is generated while a lookup is outstanding (e.g., downstream idle timeout produces a 408) it `cancel_in_flight_callback_()`s the lookup and fires an `ENVOY_BUG`.
- `onDestroy` (`cache_filter.cc:65`): sets `is_destroyed_`, invokes the in-flight cancel callback if any, and resets `lookup_result_`. Every async callback re-checks `is_destroyed_` after filter-chain calls that can reenter.
- `onAboveWriteBufferHighWatermark` / `onBelowWriteBufferLowWatermark` (`cache_filter.cc:452`, `454`): maintain `downstream_watermarked_` refcount. While blocked, body fetches are paused; when the counter returns to zero and `get_body_on_unblocked_` was set, `getBody()` resumes.

## Decision / logic
- Cluster resolution (`cache_filter.cc:127-143`): with an `override_upstream_cluster`, the configured cluster is used but the original route cluster is retained for the cache key (fallback `"unknown"` if none).
- Lookup result (`onLookupResult`, `cache_filter.cc:186`): null `http_source_` ⇒ `resetStream()` with details `cache.aborted_lookup`. Otherwise `stats().incForStatus(status_)` and, unless the status is `Uncacheable`, sets the `ResponseFromCacheFilter` response flag.
- Status → details mapping (`responseCodeDetailsFromStatus`, `cache_filter.cc:167`): `Miss`/`FailedValidation` → `cache.insert_via_upstream`; `Hit`/`FoundNotModified`/`Follower`/`Validated`/`ValidatedFree`/`UpstreamReset` → `cache.response_from_cache_filter`; everything else → `ViaUpstream`.
- `onHeaders` (`cache_filter.cc:328`): strips an indiscriminately-added `Age` header for `Miss`/`Validated`/`ValidatedFree`; detects `206 Partial Content` into `is_partial_response_`; parses `Content-Range` via `rangeFromHeaders` (`cache_filter.cc:291`) to seed `remaining_ranges_`. For HEAD requests or truly-empty responses, signals `end_stream` on `encodeHeaders`.
- Body streaming (`getBody`, `cache_filter.cc:237`): fetches at most `encoder_callbacks_->bufferLimit()` bytes per call (or `MaxBytesToFetchFromCachePerRead = 64 MiB` if unlimited). Uses a hand-rolled loop (`GetBodyLoop { InCallback, Again, Idle }`) so that synchronous callbacks iterate instead of recursing — if the callback was posted asynchronously the loop is re-entered via a new `getBody()` call.
- `onBody` (`cache_filter.cc:375`): trims `remaining_ranges_[0]` by the bytes received; `null body` + `end_stream` ends the response, `null body` + more to come transitions to `getTrailers`. Oversized body is treated as `IS_ENVOY_BUG` and resets the stream. For partial-content responses, ends the stream once ranges are consumed (skipping trailers). Returns `true` to request another `getBody` iteration, or sets `get_body_on_unblocked_` when downstream is watermarked.
- Reset handling: every `EndStream::Reset` path sets a distinct response-code-details (`CacheFilterAbortedDuring{Lookup,Headers,Body,Trailers}`) and resets the stream.

## Configuration
`CacheFilterConfig` (`cache_filter.cc:45`) stores:
- `vary_allow_list_` — headers allowed to vary on, built from `allowed_vary_headers`.
- `ignore_request_cache_control_header_` — forwarded into `ActiveLookupRequest`.
- `override_upstream_cluster_` — optional cluster override; used as route and cluster resolution as described above.
- `cluster_manager_` + `upstream_options_` — used by `UpstreamRequestImplFactory`.
- `cache_sessions_` — nullable: if `disabled` in proto, the filter becomes pure pass-through (`cache_filter.cc:106`).

No per-route filter config is exposed here — the factory implements only `createFilterFactoryFromProtoTyped`.

## Stats
Emitted through `CacheFilterStats` (`stats.h:12`). The interface mandates:
- `incForStatus(CacheEntryStatus)` — covers `Hit`, `Miss`, `Validated`, `ValidatedFree`, `FoundNotModified`, `Follower`, `Uncacheable`, `FailedValidation`, `LookupError`, `UpstreamReset`.
- `incCacheSessionsEntries` / `decCacheSessionsEntries` — active session gauge.
- `incCacheSessionsSubscribers` / `subCacheSessionsSubscribers(n)` — coalesced subscriber count.
- `addUpstreamBufferedBytes` / `subUpstreamBufferedBytes` — bytes buffered waiting to stream.
Concrete stat naming comes from `generateStats(scope, label)` (`stats.h:32`); the `CacheSessions` impl owns the generated stats and exposes them through `config.stats()` at `cache_filter.h:39`.

## Factory
`CacheFilterFactory` extends `Common::FactoryBase<CacheV2Config>` (`config.h:13`). `createFilterFactoryFromProtoTyped` (`config.cc:12`):
1. If not `disabled`, resolves the `HttpCacheFactory` by the `typed_config.type_url` and calls `getCache(config, context)`; throws if unknown or if factory returns an error.
2. Constructs a shared `CacheFilterConfig`.
3. Returns a callback doing `addStreamFilter(std::make_shared<CacheFilter>(config))`.

Registered by `REGISTER_FACTORY(CacheFilterFactory, NamedHttpFilterConfigFactory)` at `config.cc:42`.
