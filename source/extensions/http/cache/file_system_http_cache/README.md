# File System HTTP Cache

A persistent, on-disk cache backend consumed by the HTTP `cache` filter
(`source/extensions/filters/http/cache`). The filter calls into any registered
`HttpCache` implementation to store and retrieve cached responses; this
extension stores each entry as a single file under a configured directory and
runs a background eviction thread to enforce size / count limits. It is
registered via an `HttpCacheFactory` and selected by `CacheConfig.typed_config`
in the HCM cache filter.

Proto: `envoy.extensions.http.cache.file_system_http_cache.v3.FileSystemHttpCacheConfig`.

## Files
- `file_system_http_cache.h/cc` - cache instance and `HeaderUpdateContext`.
- `lookup_context.h/cc` - async read / vary / range handling.
- `insert_context.h/cc` - async write of a new entry.
- `cache_file_fixed_block.h/cc` - on-disk fixed header preamble.
- `cache_file_header.proto` + `cache_file_header_proto_util.h/cc` - variable
  header/trailer proto stored in the file.
- `cache_eviction_thread.h/cc` - single shared thread performing LRU eviction.
- `stats.h/cc` - cache-scoped counters / gauges.
- `config.cc` - singleton registry + `HttpCacheFactory`.
- `DESIGN.md` - canonical design doc (referenced from code comments).

## Interface
- Implements `Envoy::Extensions::HttpFilters::Cache::HttpCache`, providing
  `makeLookupContext`, `makeInsertContext`, `updateHeaders`, `cacheInfo`.
- Factory implements `HttpCacheFactory` and registers via
  `Registry::RegisterFactory`.
- Each cache file layout: `CacheFileFixedBlock` (offsets/sizes), then a
  `CacheFileHeader` proto, then body, then optional trailers proto.

## Logic
- `config.cc` keeps a process-wide `CacheSingleton` keyed by normalized
  `cache_path`; configs with the same path must match exactly or a throw
  fires.
- `makeLookupContext` builds a `FileLookupContext` that opens
  `cache-<stableHashKey>` under `cache_path` and streams headers/body through
  `AsyncFileManager`. Vary responses are a two-step lookup: first the base
  key (a vary-only stub file) yields the vary headers, then the varied key is
  fetched.
- `makeInsertContext` refuses to insert if another writer is already writing
  the key (`workInProgress`); otherwise it anonymously-creates a temp file,
  writes block+headers+body, and hard-links it into place.
- `updateHeaders` is a full copy-on-write: unlink original, allocate a new
  anonymous file, write new header block / proto, copy the body in
  128 KiB chunks, and hard-link the new file into the original path. The
  inline `HeaderUpdateContext` class chains the async callbacks.
- Size stats (`size_bytes`, `size_count`) are updated on insert / eviction;
  when either exceeds `max_cache_size_bytes` / `max_cache_entry_count`,
  `trackFileAdded` signals the `CacheEvictionThread`, which rescans the
  directory and LRU-evicts.

## Key decision points
- `file_system_http_cache.cc:22` - 128 KiB copy chunk for header updates.
- `file_system_http_cache.cc:108` - vary stub file is written before varied
  entry is started.
- `file_system_http_cache.cc:166` - `HeaderUpdateContext::unlinkOriginal`
  tolerates ENOENT (another deleter may have raced).
- `file_system_http_cache.cc:329` - `maybeStartWritingEntry` uses a set +
  `Cleanup` to serialize concurrent writers for the same key.
- `file_system_http_cache.cc:409` - `needsEviction` checks configured byte /
  count limits.
- `config.cc:68` - reject mismatched configs for the same path.

## Configuration
- `cache_path` (required) - directory root, normalized to end with `/`.
- `manager_config` - `AsyncFileManager` options (thread pool size, etc.).
- `max_cache_size_bytes`, `max_cache_entry_count` - trigger eviction.
- `cache_subdivisions` - reserved for future directory-tree layout.

## Stats / errors
Hosted under `cache.<cache_path>.*`:
- `size_bytes`, `size_count` (gauges) - current cache occupancy; may drift
  (see `stats.h`).
- `size_limit_bytes`, `size_limit_count` (gauges) - configured limits.
- `eviction_runs` (counter) - eviction sweeps completed.
- `event` with `event_type=hit|miss` (counter) - lookup outcome.

Async file failures are logged at `warn` and surfaced as false completions
through the cache filter; `updateHeaders` failure causes the cache entry to
be deleted (the intermediate unlink).
