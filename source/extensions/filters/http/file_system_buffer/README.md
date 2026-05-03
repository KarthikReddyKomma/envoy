# File System Buffer (`envoy.filters.http.file_system_buffer`)

Buffers the request and/or response body with a two-tier memory + on-disk storage model so large bodies don't pin virtual memory. When the in-memory buffer grows past its limit, the latest chunks spill to an async-files temp file; when memory drains, earlier chunks are paged back in. Also supports injecting or replacing `Content-Length` after the full body is captured and propagating downstream watermarks so upstream/downstream senders slow down appropriately.

Proto: `envoy.extensions.filters.http.file_system_buffer.v3.FileSystemBufferFilterConfig`.

## Files
- `filter.h/cc` — `FileSystemBufferFilter` (full `Http::StreamFilter` + `DownstreamWatermarkCallbacks`) and `BufferedStreamState` (per-direction state).
- `filter_config.h/cc` — `FileSystemBufferFilterConfig` / `FileSystemBufferFilterMergedConfig` (combines listener + per-route configs), plus `BufferBehavior` wrapping the protobuf one-of.
- `fragment.h/cc` — `Fragment`, a buffer chunk that can be resident in memory or in the on-disk file; owns the round-trip between the two.
- `config.h/cc` — `FileSystemBufferFilterFactory`; `REGISTER_FACTORY` at `config.cc:64`.

## Lifecycle
`FileSystemBufferFilter` is a `StreamFilter` (`filter.h:45`) installed once per stream via `addStreamFilter` (`config.cc:33`). The factory holds the shared `FileSystemBufferFilterConfig` and is also registered as a route-specific factory (`config.cc:53-62`).

Overridden callbacks:
- `setDecoderFilterCallbacks` (`filter.cc:125-129`): stores callbacks and registers as a downstream watermark observer.
- `setEncoderFilterCallbacks` (`filter.cc:217-220`): stores encoder callbacks.
- `decodeHeaders` (`filter.cc:47-73`): end-stream → pass through; else `initPerRouteConfig()` merges per-route with base config (`filter.cc:16-40`). If the merge fails to find an `AsyncFileManager` (and behavior is not `bypass`), `onBadConfig()` sends a 500 (`filter.cc:42-45`). Determines `injecting_content_length_header_` from the config's `injectContentLength`/`replaceContentLength` vs. the incoming `Content-Length` (`filter.cc:61-64`), tightens the downstream buffer limit to the memory cap (`filter.cc:70`), and returns `StopIteration` so data flows through the buffer.
- `decodeData` (`filter.cc:138-150`): rejects with `413 PayloadTooLarge` when accumulated bytes would exceed memory+storage limits and the body must be fully buffered (`injectContentLength` or `alwaysFullyBuffer`). Otherwise delegates to `receiveData`.
- `decodeTrailers` (`filter.cc:205-207`): delegates to `receiveTrailers` with the request behavior.
- `encodeHeaders` (`filter.cc:75-101`): mirrors `decodeHeaders` for the response side.
- `encodeData` (`filter.cc:191-203`): mirrors `decodeData`; overflow → `500 InternalServerError`.
- `encodeTrailers` (`filter.cc:187-189`): mirrors `decodeTrailers`.
- `encode1xxHeaders` / `encodeMetadata` — pass-through (`filter.cc:209-215`).
- `onAboveWriteBufferHighWatermark` / `onBelowWriteBufferLowWatermark` (`filter.cc:103-123`): increment/decrement `response_state_.water_level_`. Watermarks sent by the filter itself set `sending_watermark_` so the echo is not double-counted. Reaching `water_level_ == 0` triggers `dispatchStateChanged()`.
- `onDestroy` (`filter.cc:222-229`): sets the shared `is_destroyed_` sentinel, cancels the in-flight async action, and closes both directions.

`receiveData` (`filter.cc:152-169`) splits incoming data into `Fragment`s (each at most `memoryBufferBytesLimit`), appends them to `state.buffer_`, and queues an `onStateChange` via `dispatchStateChanged`. `receiveTrailers` (`filter.cc:171-185`) marks the direction as end-of-stream with trailers pending.

`onStateChange` (declared `filter.h:90`) is the central scheduler; it calls `maybeStorage` to page memory↔disk, `maybeSendWatermarkUpdates` to emit downstream watermarks when disk backlog grows, then `maybeOutputRequest` / `maybeOutputResponse` to inject buffered chunks back into the chain via `injectDecodedDataToFilterChain` (`filter.cc:250`) once watermark pressure allows.

## Decision / logic
- Config resolution: `initPerRouteConfig` builds a chain of per-route configs (most-specific first) plus the base config and constructs a `FileSystemBufferFilterMergedConfig` (`filter.cc:17-40`). If both directions' behavior is `bypass`, absence of a file manager is allowed (`filter.cc:35-38`).
- `injecting_content_length_header_` is set when the config asks to inject and no header is present, or when `replaceContentLength` is true (`filter.cc:61-64` / `filter.cc:89-92`).
- Overflow gates: exceeding `memoryBufferBytesLimit + storageBufferBytesLimit` only aborts when the direction must be fully buffered (`injectContentLength` or `alwaysFullyBuffer`); otherwise streaming continues. Request side → `413` (`filter.cc:142-148`); response side → `500` (`filter.cc:195-200`).
- `bypass` short-circuits both header decisions and data reception (`filter.cc:56-57`, `filter.cc:155-156`, `filter.cc:84-85`, `filter.cc:178-179`).
- Post-destroy safety: `is_destroyed_` is a `shared_ptr<bool>` captured by dispatched callbacks so async-files completions that land after `onDestroy` no-op (`filter.h:77`).
- Watermark echoes are suppressed via `sending_watermark_` (`filter.cc:104-108`) to avoid counting our own high/low watermark signals.
- `setBufferLimit(memoryBufferBytesLimit)` on the stream (`filter.cc:70` / `filter.cc:98`) keeps the outgoing buffer from ballooning beyond configured memory.

## Configuration
- `manager_config` — feeds `AsyncFileManagerFactory::getAsyncFileManager` (`config.cc:27-28`); optional when both directions bypass.
- `request` / `response` → `StreamConfig { BufferBehavior, memory_buffer_bytes_limit, storage_buffer_bytes_limit, storage_buffer_queue_high_watermark_bytes }`. Behavior is one of stream-on, fully-buffer, bypass, inject-content-length, replace-content-length (enum handled by `BufferBehavior` in `filter_config.h`).
- Per-route / per-virtual-host config supported (`config.cc:53-62`). `initPerRouteConfig` composes them (`filter.cc:17-32`) via `route->perFilterConfigs(filterName())`.

## Stats
No stats are declared by this filter; it relies on Envoy's response-code-based visibility and on the `AsyncFileManager` stats. Error paths use `sendLocalReply` with rc-details strings `"buffer filter error"` (`filter.cc:134`), `"buffer limit exceeded"` (`filter.cc:144`, `filter.cc:197`).
