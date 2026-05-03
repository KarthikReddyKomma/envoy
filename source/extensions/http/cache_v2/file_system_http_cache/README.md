# File System HTTP Cache (v2)

On-disk persistent cache backend consumed by the v2 HTTP cache filter
(`source/extensions/filters/http/cache_v2`). Functionally analogous to
`source/extensions/http/cache/file_system_http_cache`, but targets the
redesigned `HttpCache` API where lookup, insert, touch, evict and
`updateHeaders` are direct async methods on the cache and reads go through
an `HttpSource` abstraction. Registered as an `HttpCacheFactory` selected by
the v2 filter's `CacheConfig.typed_config`.

Proto: `envoy.extensions.http.cache_v2.file_system_http_cache.v3.FileSystemHttpCacheV2Config`.

## Files
- `file_system_http_cache.h/cc` - cache instance + top-level API.
- `lookup_context.h/cc` - async entry lookup / vary resolution.
- `insert_context.h/cc` - writes new entries to disk.
- `cache_file_reader.h/cc` - shared file-handle reader used by sources.
- `cache_file_fixed_block.h/cc` - fixed-size file preamble.
- `cache_file_header.proto`, `cache_file_header_proto_util.h/cc` - variable
  headers proto.
- `cache_eviction_thread.h/cc` - background LRU eviction thread.
- `stats.h/cc` - cache stats.
- `config.cc` - singleton registry + factory.
- `DESIGN.md` - design notes.

## Interface
- Implements `Envoy::Extensions::HttpFilters::CacheV2::HttpCache`:
  `cacheInfo`, `lookup`, `evict`, `touch`, `updateHeaders`, `insert`.
- Factory implements `HttpCacheFactory` (v2 namespace).
- Uses `AsyncFileManager` for all I/O and shares a single
  `CacheEvictionThread` across cache instances via the singleton.

## Logic
- A `CacheSingleton` keyed by normalized `cache_path` returns (or creates) a
  cache instance per unique config; duplicate paths with different configs
  throw (`config.cc`).
- `lookup` opens the file, reads the fixed block, then the headers proto,
  and invokes the callback with headers + an `HttpSource` backed by
  `cache_file_reader` for body/trailers. Vary entries are resolved by doing
  a second open after the caller provides the varied key.
- `insert` (called by the filter with a live `HttpSource`) writes a temp
  anonymous file, streams body from the source, finalizes with a hard link,
  and reports progress through the provided `CacheProgressReceiver`.
- `updateHeaders` uses the same copy-on-write strategy as v1 but pushes work
  into the file manager from the caller-supplied dispatcher.
- `evict` unlinks a specific key; `touch` updates LRU metadata for
  expiry-ordering decisions made by the eviction thread.
- Stats size tracking plus `needsEviction` signal the eviction thread when
  configured limits are exceeded.

## Key decision points
- `file_system_http_cache.cc` - `insert` pipes the `HttpSource` directly to
  disk chunk by chunk, so large bodies never sit in memory.
- `file_system_http_cache.h:126` - `CacheShared::signal_eviction_` is
  cleared on cache destruction so a live eviction thread cannot keep a dead
  cache's state alive.
- `config.cc` - same path + differing config is an error.

## Configuration
- `cache_path` - directory root.
- `manager_config` - `AsyncFileManager` config.
- `max_cache_size_bytes`, `max_cache_entry_count` - eviction triggers.

## Stats / errors
Hosted under `cache.<cache_path>.*` (see `stats.h`):
- `size_bytes`, `size_count`, `size_limit_bytes`, `size_limit_count`.
- `eviction_runs`.
- `event{event_type=hit|miss}`.

File I/O failures are surfaced via callbacks and logged at `warn`; failed
`updateHeaders` leaves the entry deleted.
