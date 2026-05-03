# Dynamic Modules (`envoy.extensions.filters.http.dynamic_modules`)

Loads a native shared library ("dynamic module") at config time and delegates all HTTP filter event hooks to the library through a stable C ABI. Supports three module sources — by registered name, a local filesystem path, or a remote HTTP fetch with SHA256-addressed on-disk cache — and works as both a downstream and an upstream HTTP filter (`UpstreamDynamicModuleConfigFactory`, `factory.h:57`). Per-route filter config is also routed through the module.

Proto: `envoy.extensions.filters.http.dynamic_modules.v3.DynamicModuleFilter` and `DynamicModuleFilterPerRoute`.

## Files
- `factory.h/cc` — `DynamicModuleConfigFactory` (`DualFactoryBase`); handles module loading, remote fetch/cache, and per-route config.
- `factory_registration.cc` — `REGISTER_FACTORY` calls for the downstream and upstream variants.
- `filter.h/cc` — `DynamicModuleHttpFilter` (per-stream `Http::StreamFilter` + `DownstreamWatermarkCallbacks`) and `DynamicModuleHttpFilterScheduler` (cross-thread event posting).
- `filter_config.h/cc` — `DynamicModuleHttpFilterConfig`, `DynamicModuleHttpPerRouteFilterConfig`, the module symbol table, HTTP callout callbacks, and the module-owned stats registry (`counters_`, `gauge_vecs_`, etc.).
- `abi_impl.cc` — C entry points exported to modules (not covered in detail here).

## Lifecycle
### Factory
`DynamicModuleConfigFactory::createFilterFactory` (`factory.cc:67`) resolves the module source:
- `module.remote`: checks `moduleTempPath(sha256)` (`factory.cc:80-81`). If cached, loads synchronously and wipes any background-fetch entry (`factory.cc:82-86`). If not and `nack_on_cache_miss` is set, returns an error while kicking off a background fetch (`factory.cc:94-97`). Otherwise uses `RemoteAsyncDataProvider` through `createFilterFactoryFromRemoteSource` (`factory.cc:100-103`, `factory.cc:122`).
- `module.local.filename`: `newDynamicModule(...)` on the given path (`factory.cc:105-108`).
- Otherwise `module.name`: `newDynamicModuleByName(...)` (`factory.cc:110-113`).
`buildFilterFactoryCallback` (`factory.cc:19`) then builds a `DynamicModuleHttpFilterConfig` via `newDynamicModuleHttpFilterConfig`, registers a custom stat namespace when `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix` is on (`factory.cc:42-44`), and returns a lambda that per-stream: parses the worker index from the dispatcher name (`factory.cc:47-53`), creates the shared `DynamicModuleHttpFilter`, adds it with `addStreamFilter`, then calls `initializeInModuleFilter()` so the module sees both decoder and encoder callbacks at construction time (`factory.cc:54-62`). Remote-fetched factories are fail-open: if the fetch fails the stored `filter_factory_cb` stays empty and the per-stream lambda is a no-op (`factory.cc:166-173`).

`createRouteSpecificFilterConfigTyped` (`factory.cc:182`) loads a module by name, packs `filter_config` to bytes, falls back to `per_route_config_name` when `filter_name` is empty (`factory.cc:197-200`), and returns a `DynamicModuleHttpPerRouteFilterConfig`.

`isTerminalFilterByProtoTyped` returns `proto_config.terminal_filter()` (`factory.h:45-48`).

### Per-stream filter
`DynamicModuleHttpFilter` is a shared `StreamFilter` (`filter.h:18`) that forwards each hook to the module function pointer stored on the config:
- `decodeHeaders` / `decodeData` / `decodeTrailers` → `on_http_filter_request_headers_` / `_body_` / `_trailers_` (`filter.cc:79-97`). Each records the returned status into `in_continue_` (used by `continueDecoding`/`continueEncoding` guards) and casts the module enum to Envoy's `FilterXxxStatus`.
- `decodeMetadata` / `encode1xxHeaders` / `encodeMetadata` are no-op `Continue` paths (`filter.cc:99-109`, `filter.cc:140-143`).
- `encodeHeaders` / `encodeData` / `encodeTrailers` (`filter.cc:111-138`) guard on `sent_local_reply_`: once a local reply has been sent, encode callbacks are skipped to avoid reentering the module on the synthetic response.
- `setDecoderFilterCallbacks` (`filter.h:38-44`) caches callbacks and calls `maybeRegisterDownstreamWatermarkCallbacks` (`filter.cc:21-27`); registration is gated on both callbacks and the in-module filter existing (idempotent, so it runs once on whichever entry point completes last).
- `onAboveWriteBufferHighWatermark` / `onBelowWriteBufferLowWatermark` forward to the module.
- `sendLocalReply` (`filter.cc:145-148`) sets `sent_local_reply_` then delegates to `decoder_callbacks_->sendLocalReply`.
- `onStreamComplete` (`filter.cc:29`) and `onDestroy` (`filter.cc:31-40`) call into the module and then `destroy()` (`filter.cc:42-77`), which invokes `on_http_filter_destroy_`, cancels all pending one-shot HTTP callouts (`filter.cc:50-60`), resets all streaming callouts (`filter.cc:62-72`), and nulls the callback pointers.
- `onLocalReply` forwards to the module (see `abi_impl.cc`).

Additional facilities exposed to modules: `sendHttpCallout` / `startHttpStream` / `resetHttpStream` / `sendStreamData` / `sendStreamTrailers` implement async HTTP callouts through `Upstream::ThreadLocalCluster::httpAsyncClient` (`filter.cc:152-176` and continuations for streaming in the rest of the file). Socket options set by the module are stored in `socket_options_` via `storeSocketOptionInt` / `storeSocketOptionBytes` (`filter.h:341-357`) and surfaced back to it.

`DynamicModuleHttpFilterScheduler` (`filter.h:369`) lets module code commit events from another thread: `commit()` locks a weak_ptr to the filter and posts `onScheduled(event_id)` onto the stream's dispatcher (`filter.h:373-391`).

## Decision / logic
- Module-source dispatch order: cached remote → NACK-on-miss branch → async fetch → local → by name (`factory.cc:73-114`).
- Cached remote load failures are **not** re-fetched because SHA256 addressing means the bytes would be identical (`factory.cc:88-90`).
- `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix` controls whether the custom stat namespace prefix is stripped from Prometheus output (`factory.cc:42-44`).
- `terminal_filter` is propagated to both the config and `isTerminalFilterByProtoTyped`.
- Encode hooks short-circuit after `sendLocalReply` (`filter.cc:112-114`, `filter.cc:121-123`, `filter.cc:132-134`) to prevent double-invocation when the filter itself emits a local reply.
- `downstream_watermark_callbacks_registered_` gates both `addDownstreamWatermarkCallbacks` and the paired `remove` in `onDestroy` because `removeDownstreamWatermarkCallbacks` asserts the callback was previously added (`filter.cc:33-38`).

## Configuration
- `dynamic_module_config` — source (`module.remote` / `module.local` / `name`), `do_not_close`, `load_globally`, `metrics_namespace`, and `nack_on_cache_miss`.
- `filter_name` — selected filter within the module.
- `filter_config` — `google.protobuf.Any` passed as bytes to `newDynamicModuleHttpFilterConfig` (`factory.cc:22-27`).
- `terminal_filter` — marks the filter terminal.
- Per-route (`DynamicModuleFilterPerRoute`): `dynamic_module_config` (name-only), `filter_name` or `per_route_config_name`, and a typed `filter_config`. Instantiated as `DynamicModuleHttpPerRouteFilterConfig` (`filter_config.h:412`).

## Stats
Stats are declared by the module itself at config construction. `DynamicModuleHttpFilterConfig` holds registries for counters, counter vectors, gauges, gauge vectors, histograms, and histogram vectors (`filter_config.h:114-277`). All live under the configured `metrics_namespace` (default `"dynamicmodulescustom"`, `filter_config.h:17`). Creation is frozen after `envoy_dynamic_module_on_http_filter_config_new` returns (`stat_creation_frozen_`, `filter_config.h:110`). Handles are keyed by 1-based ABI IDs (`ID_TO_INDEX`, `filter_config.h:205`) and looked up via `getCounterById` / `getGaugeById` / `getHistogramById` (and their `*Vec` variants).
