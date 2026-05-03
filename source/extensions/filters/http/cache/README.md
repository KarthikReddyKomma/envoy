# HTTP Cache Filter (`envoy.filters.http.cache`)

RFC 7234 HTTP response cache. Serves cache hits locally, validates stale
entries against the origin, and inserts cacheable upstream responses into a
pluggable backend.

Proto: `envoy.extensions.filters.http.cache.v3.CacheConfig`.

## Lifecycle

Bidirectional filter.

- **`decodeHeaders()` (`cache_filter.cc:101`)**
  - Build a `LookupRequest` (URL + selected `Vary` headers).
  - `cache_->makeLookupContext()` starts an async lookup.
  - Stops iteration (line 133) waiting for headers.
- **`onHeaders()` callback (`cache_filter.cc:91`)** — dispatches on
  `CacheEntryStatus` (`cache_entry_utils.h:28–42`):
  - `Ok` → serve from cache directly.
  - `RequiresValidation` → inject `If-Modified-Since` / `If-None-Match` and
    forward upstream; on 304, merge cached headers with `applyHeaderUpdate`.
  - `Unusable` → cache miss, forward upstream and (if eligible) insert on the
    encode path.
- **`encodeHeaders()` (`cache_filter.cc:148`)** — if the response is
  cacheable, queues it for insertion via `cache_insert_queue`.

## Range requests

`createRangeDetails()` (`range_utils.h:113`) parses `Range`, validates
against `Content-Length`, returns `AdjustedByteRange`s (lines 65–89). Cache
hits with a satisfiable `Range` stream only the requested byte ranges
(`handleCacheHitWithRangeRequest`, `cache_filter.h:100`).

## Vary

Responses with `Vary` have their selected headers folded into the cache key,
but only for headers listed in `allowed_vary_headers`
(`cache_filter_config.cc:44`).

## Configuration

- `typed_config` — pluggable cache backend (e.g., `SimpleHttpCache`).
  Factory lookup at `config.cc:18–27`.
- `allowed_vary_headers` — whitelist of headers considered for `Vary`.
- `ignore_request_cache_control_header` — ignore client `Cache-Control`.

## Observability

Cache hit/miss/insert outcomes are recorded in `CacheFilterLoggingInfo` on
`StreamInfo` (`cache_filter.cc:92–98`) and available to access logs.

## Files

- `cache_filter.{h,cc}` — filter.
- `http_cache.{h,cc}`, `cache_policy.{h,cc}`, `cache_entry_utils.{h,cc}` —
  abstractions for pluggable caches.
- `range_utils.{h,cc}` — Range parsing.
- `config.{h,cc}`, `cache_filter_config.{h,cc}` — factory.
