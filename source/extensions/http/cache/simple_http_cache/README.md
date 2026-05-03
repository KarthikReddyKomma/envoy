# Simple HTTP Cache

An in-memory reference cache backend for the HTTP `cache` filter. It stores
each response (headers + body + trailers) in a process-global map keyed by
request key, with no eviction. It is intended for demos and integration
tests - not for production use (stated on `simple_http_cache.h:15`). The
`cache` filter (`source/extensions/filters/http/cache`) selects it by
`CacheConfig.typed_config` pointing at this extension's proto.

Proto: `envoy.extensions.http.cache.simple_http_cache.v3.SimpleHttpCacheConfig`.

## Files
- `simple_http_cache.h` - `SimpleHttpCache` class, map entries.
- `simple_http_cache.cc` - lookup / insert / vary logic, factory registration.

## Interface
- `SimpleHttpCache` is an `HttpCache` and a `Singleton::Instance`. It exposes
  `makeLookupContext`, `makeInsertContext`, `updateHeaders`, `cacheInfo`.
- `SimpleHttpCacheFactory` is an `HttpCacheFactory` registered through
  `Registry::RegisterFactory`. One singleton cache is shared across all
  filter instances, retrieved through `SingletonManager::getTyped`.

## Logic
- Storage is a single `absl::flat_hash_map<Key, Entry>` guarded by
  `absl::Mutex`, with `Entry` holding response headers, metadata, body
  string, and trailers.
- `SimpleLookupContext::getHeaders` does a synchronous map lookup, then posts
  the result back on the filter's dispatcher. `getBody` slices the cached
  string and `getTrailers` returns the stored map.
- `SimpleInsertContext` buffers chunks into an `OwnedImpl` and commits
  everything on the end-of-stream call via `insert` or `varyInsert`.
- `varyInsert` writes two entries: the varied entry at
  `key + vary_identifier` and a vary-only stub at the base key that records
  which `Vary` headers to consult. `varyLookup` reads the stub, computes the
  vary identifier from the request, then fetches the varied entry.
- `updateHeaders` applies `applyHeaderUpdate` on the existing entry in
  place (under the writer lock); vary entries are resolved first.

## Key decision points
- `simple_http_cache.cc:146` - commit either `varyInsert` or plain `insert`
  based on whether the response has `Vary`.
- `simple_http_cache.cc:19` - `variedRequestKey` refuses vary identifiers
  outside the `VaryAllowList`.
- `simple_http_cache.cc:189` - `updateHeaders` returns `false` if the key
  was never inserted, to signal the filter.
- `simple_http_cache.cc:307` - stub entry stores vary header list in the
  body field so `varyLookup` can rebuild the vary key.

## Configuration
`SimpleHttpCacheConfig` is an empty message; there are no tunables.

## Stats / errors
No dedicated counters. Errors: `updateHeaders` posts `false` for missing /
vary-mismatched keys; `varyInsert` returns `false` if the vary identifier
cannot be built from the allow-list.
