# File Server (`envoy.filters.http.file_server`)

Serves static files from the local filesystem. Matches the request path against a radix tree of configured `PathMapping`s, computes a normalized filesystem path, and streams the file through the async-files subsystem back onto the encode path using `encodeHeaders`/`encodeData`. Supports `HEAD`, byte-range requests (single range only), directory defaults / listings, content-type lookup, and downstream flow control via watermark callbacks.

Proto: `envoy.extensions.filters.http.file_server.v3.FileServerConfig`.

## Files
- `filter.h/cc` — `FileServerFilter` (decoder-only), plus the `FileStreamerClient` callbacks.
- `config.h/cc` — `FileServerFilterFactory` (`DualFactoryBase`) with cross-cutting proto validation in `validateProto` (`config.cc:21-56`); `REGISTER_FACTORY` at `config.cc:88`.
- `filter_config.h/cc` — `FileServerConfig` (the `RouteSpecificFilterConfig`), radix-tree path matching, content-type table, directory behavior list.
- `file_streamer.h/cc` — `FileStreamer` that drives `AsyncFileManager` and pushes chunks back into the filter via `FileStreamerClient`.
- `absl_status_to_http_status.h/cc` — error-code mapping helper.

## Lifecycle
`FileServerFilter` extends `Http::PassThroughDecoderFilter`, `FileStreamerClient`, and `Http::DownstreamWatermarkCallbacks` (`filter.h:21-23`). The factory builds a shared `FileServerConfig` from the proto via `FileServerConfig::create` and installs one filter per stream (`config.cc:70-73`). The same `FileServerConfig` type also acts as a per-route config (`createRouteSpecificFilterConfigTyped`, `config.cc:76-86`).

Overridden callbacks:
- `setDecoderFilterCallbacks` (`filter.cc:63-66`): registers `*this` as downstream watermark callbacks, then forwards to the base.
- `decodeHeaders` (`filter.cc:72`):
  1. If no route or no `:path`, pass through (`filter.cc:74-76`).
  2. Prefer the per-route `FileServerConfig` via `resolveMostSpecificPerFilterConfig`, fall back to the listener-level one (`filter.cc:77-81`).
  3. Percent-decode the path (`filter.cc:82`) and look up a `PathMapping` in the config's radix tree. A miss → continue (bypass); this filter is not terminal (`filter.cc:84-87`).
  4. `applyPathMapping` returns `nullopt` for non-normalized paths (`..`, `./`) → `400` with rc-details `file_server_rejected_non_normalized_path` (`filter.cc:88-94`).
  5. Missing `:method` → `400 file_server_rejected_missing_method` (`filter.cc:95-100`).
  6. Method not `GET`/`HEAD` → `405 file_server_rejected_method` (`filter.cc:101-107`).
  7. Request body present (`!end_stream`) → `400 file_server_rejected_not_end_stream` (`filter.cc:108-113`).
  8. Parse the `Range:` header (`filter.cc:28-56`); single `bytes=start-end` only; malformed headers degrade to `{0, 0}` (full file).
  9. Record `is_head_`, kick off `file_streamer_.begin(...)` with the dispatcher and resolved path, and return `StopIteration` (`filter.cc:115-119`).
- `FileStreamerClient` hooks:
  - `headersFromFile` (`filter.cc:122-127`): emits the response headers via `decoder_callbacks_->encodeHeaders(...)`. `is_head_` or `content-length: 0` short-circuits to end-of-stream; returns whether the streamer should send the body.
  - `bodyChunkFromFile` (`filter.cc:129-131`): forwards each chunk via `decoder_callbacks_->encodeData`.
  - `errorFromFile` (`filter.cc:133-141`): before any headers are sent → `sendLocalReply` with the given code; after headers sent → `resetStream(LocalReset, ...)` since the response is mid-flight.
- `onAboveWriteBufferHighWatermark` / `onBelowWriteBufferLowWatermark` (`filter.cc:68-70`): pause/unpause the streamer to match downstream backpressure.
- `onDestroy` (`filter.cc:143-146`): `file_streamer_.abort()` and releases the config shared_ptr.

No encode-side callbacks are overridden; all response data is fed back through `decoder_callbacks_->encodeHeaders` / `encodeData`.

## Decision / logic
- Path match uses `RadixTree<PathMapping>` (`filter_config.h:53`) for longest-prefix semantics on `request_path_prefix`.
- Range parsing rejects multi-ranges, suffix ranges, and malformed numbers by returning `{0, 0}` (`filter.cc:38-52`).
- Proto validation (`config.cc:21-56`) rejects duplicate `request_path_prefix`, requires exactly one of `default_file`/`list` per `directory_behavior`, rejects duplicate `default_file`s and multiple `list` directives.
- End-of-response detection for `HEAD` / zero-length in `headersFromFile` (`filter.cc:123`).
- rc-details strings: `file_server_rejected_non_normalized_path`, `file_server_rejected_missing_method`, `file_server_rejected_method`, `file_server_rejected_not_end_stream`.

## Configuration
- `path_mappings[]` — `request_path_prefix` to a filesystem base; validated unique.
- `directory_behaviors[]` — one of `default_file` or `list` (mutually exclusive per entry, `config.cc:33-54`).
- `content_types` and the default content type (`filter_config.h:54-55`).
- `manager_config` on the async-file side (handled inside `FileStreamer`/`AsyncFileManager`).
Per-route: the filter resolves the most specific per-filter `FileServerConfig` via `resolveMostSpecificPerFilterConfig` (`filter.cc:78`), allowing routes / virtual hosts to supply a different mapping set and content-type table.

## Stats
No counters or gauges are defined in this filter; errors surface through the response code / rc-details plumbing and the async-files subsystem.
