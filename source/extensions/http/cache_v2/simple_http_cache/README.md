# Simple HTTP Cache (v2)

In-memory reference backend for the v2 HTTP cache filter
(`source/extensions/filters/http/cache_v2`). Stores entries in a global
`flat_hash_map` and never evicts. Unlike the v1 simple cache, each entry
holds its body in a mutex-guarded `string` that can be appended to while
readers are live, so streaming inserts can serve concurrent readers. Not
intended for production (header comment on `simple_http_cache.h:15`).

Proto: `envoy.extensions.http.cache_v2.simple_http_cache.v3.SimpleHttpCacheV2Config`.

## Files
- `simple_http_cache.h` - `SimpleHttpCache` and nested `Entry`.
- `simple_http_cache.cc` - async lookup / insert / update, factory
  registration.

## Interface
- Implements `Envoy::Extensions::HttpFilters::CacheV2::HttpCache`:
  `cacheInfo`, `lookup`, `evict`, `touch` (noop here), `updateHeaders`,
  `insert`.
- Factory is an `HttpCacheFactory` registered as a singleton.

## Logic
- `entries_` maps `Key` to `shared_ptr<Entry>`. Each `Entry` owns the
  body string, response headers, metadata, trailers, and an `end_stream`
  flag, all guarded by the entry's own mutex so readers and a streaming
  writer can coexist.
- `lookup` takes the map mutex only for the hash lookup, returns an
  `HttpSource` that views into the `Entry` buffer. `getBody` copies the
  requested byte range under the entry lock.
- `insert` creates a new `Entry`, installs it in the map, and drains the
  provided `HttpSource` chunk-by-chunk into `Entry::appendBody`, notifying
  the `CacheProgressReceiver` as new bytes become available so waiting
  readers can advance.
- `updateHeaders` swaps the entry's headers + metadata under the entry
  lock.
- `evict` removes the key from `entries_` (the shared entry lives on for
  any outstanding readers via `shared_ptr`).

## Key decision points
- `simple_http_cache.h:33` - per-entry mutex lets writers append while
  readers are mid-stream.
- `simple_http_cache.h:39` - `end_stream_after_body_` is the signal that
  readers have received everything.
- `simple_http_cache.cc` - `insert` drives the source->entry copy loop on
  the provided dispatcher, notifying progress between chunks.

## Configuration
`SimpleHttpCacheV2Config` is empty; no tunables.

## Stats / errors
No dedicated stats. Errors surface via the standard callbacks (`lookup`
callback receives empty result on miss; `insert` reports source errors
through the progress receiver).
