# Dynamic Modules Network Filter (`envoy.filters.network.dynamic_modules`)

Host a dynamically-loaded user module (typically Rust/C compiled via the Envoy Dynamic Modules SDK) as a full `Network::Filter` — both read and write directions. The filter forwards every L4 event into ABI callbacks exported by the module; the module can read/write the TCP buffers, close the connection, issue cross-thread scheduled events, make async HTTP callouts, record custom metrics, and attach socket options.

Proto: `envoy.extensions.filters.network.dynamic_modules.v3.DynamicModuleNetworkFilter`.

## Files
- `factory.h` / `factory.cc` — `DynamicModuleNetworkFilterConfigFactory`, an `ExceptionFreeFactoryBase` that loads the `.so`, resolves symbols, builds the shared `DynamicModuleNetworkFilterConfig`, and registers a per-connection `DynamicModuleNetworkFilter`.
- `filter_config.h` / `filter_config.cc` — `DynamicModuleNetworkFilterConfig`: the shared holder of resolved ABI function pointers, cluster manager handle, stats scope, and counter/gauge/histogram registries allocated by the module during config-time init. Also `DynamicModuleNetworkFilterConfigScheduler` for posting config-level events onto the main dispatcher.
- `filter.h` / `filter.cc` — `DynamicModuleNetworkFilter`: the per-connection C++ shell that marshals Envoy events into ABI calls. Also `DynamicModuleNetworkFilterScheduler` (filter.h:220) for cross-thread per-filter events.
- `abi_impl.cc` — C implementations of the `envoy_dynamic_module_callback_*` symbols the module calls back into (buffer ops, socket options, metrics, HTTP callouts, etc.).

## Factory
`DynamicModuleNetworkFilterConfigFactory::createFilterFactoryFromProtoTyped` (factory.cc:14):
1. Calls `DynamicModules::newDynamicModuleByName` with `do_not_close` and `load_globally` flags from the proto (factory.cc:19). Returns `InvalidArgumentError` on load failure.
2. If `filter_config` Any is set, serializes via `MessageUtil::knownAnyToBytes` (factory.cc:28).
3. Picks the stats namespace: `module_config.metrics_namespace()` or `"dynamicmodulescustom"` (factory.cc:34, default at filter_config.h:35).
4. Calls `newDynamicModuleNetworkFilterConfig` (filter_config.cc:36) which resolves every required ABI symbol (`envoy_dynamic_module_on_network_filter_config_new`, `..._destroy`, `..._new`, `..._new_connection`, `..._read`, `..._write`, `..._event`, `..._destroy`, filter_config.cc:43-75) plus optional ones (`http_callout_done`, `scheduled`, `above/below_watermark`). Failure at any `RETURN_IF_NOT_OK_REF` propagates as `Status`.
5. When runtime flag `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix` is on, registers the namespace as a custom stat namespace so `/stats/prometheus` strips the prefix (factory.cc:56).
6. Returns a factory callback that creates a `DynamicModuleNetworkFilter` (a `shared_ptr`, since it uses `enable_shared_from_this`) and adds it via `addFilter` (not `addReadFilter`) — installing it as both read and write filter (factory.cc:62-66).

Termination: `isTerminalFilterByProtoTyped` returns `proto_config.terminal_filter()` (factory.h:28), letting the module author pick.

## Per-connection filter lifecycle
`DynamicModuleNetworkFilter` implements `Network::Filter`, `Network::ConnectionCallbacks`, and holds a `shared_ptr` to the config (filter.h:26). A weak-self pattern (`enable_shared_from_this`) lets schedulers and HTTP callouts safely outlive the filter.

- `initializeReadFilterCallbacks(callbacks)` (filter.cc:64): stores `read_callbacks_`, parses the worker index from the dispatcher name `worker_N` (filter.cc:68-72), then calls `initializeInModuleFilter()` which invokes `on_network_filter_new_(in_module_config_, thisAsVoidPtr)` producing `in_module_filter_` (filter.cc:44). Finally registers itself as `ConnectionCallbacks`. Per-filter init is delayed until this point so the module can see `workerIndex()` during creation.
- `initializeWriteFilterCallbacks(callbacks)` (filter.cc:83): only stores `write_callbacks_`; no module call.
- `onNewConnection()` (filter.cc:88): if the module failed to build a filter handle (`in_module_filter_ == nullptr`), closes the connection with `NoFlush` and returns `StopIteration` (filter.cc:89-94). Otherwise calls `on_network_filter_new_connection_` and translates the return via `toEnvoyFilterStatus` (filter.cc:10).
- `onData(data, end_stream)` (filter.cc:99): stores `current_read_buffer_ = &data` so ABI helpers (invoked later from schedulers, HTTP callout completions, etc.) can still read/mutate the buffer outside the hook. Calls `on_network_filter_read_` with `data.length()` and `end_stream`, returns the translated status.
- `onWrite(data, end_stream)` (filter.cc:113): symmetric; stashes `current_write_buffer_` and invokes `on_network_filter_write_`.
- `onEvent(event)` (filter.cc:127): translates via `toAbiConnectionEvent` (filter.cc:21-34: `RemoteClose`, `LocalClose`, `Connected`, `ConnectedZeroRtt`) and forwards to the module.
- `onAboveWriteBufferHighWatermark` / `onBelowWriteBufferLowWatermark` (filter.cc:141, 149): optional — only invoked if the module exported the corresponding symbol.
- `onScheduled(event_id)` (filter.cc:135): entry point for cross-thread events posted by a `DynamicModuleNetworkFilterScheduler` (filter.h:220). The scheduler locks the weak_ptr to the filter, fetches its worker dispatcher, and posts a lambda that re-locks and calls `onScheduled` — safely no-ops if the filter is gone.
- Destruction: `~DynamicModuleNetworkFilter` calls `destroy()` (filter.cc:42) which cancels all pending `HttpCalloutCallback` requests (filter.cc:50-55), calls `on_network_filter_destroy_`, and flips `destroyed_=true`.

## HTTP callouts
`sendHttpCallout(callout_id_out, cluster_name, message, timeout_ms)` (filter.cc:241) looks up a thread-local cluster, sends via `Http::AsyncClient` with a `HttpCalloutCallback` that holds a `weak_ptr<DynamicModuleNetworkFilter>` and a per-filter `callout_id_` (filter.h:179). On completion, if the filter is still alive, the module's `on_network_filter_http_callout_done_` is invoked. `destroy()` cancels every outstanding request (filter.cc:50).

## Socket options
Modules can persist socket options keyed by `(level, name, state)` via `storeSocketOptionInt`/`storeSocketOptionBytes` (filter.cc:175, 182). Envoy surfaces these to the listener/upstream connection via `socketOptionCount`/`copySocketOptions` (filter.cc:214). Retrieval by key: `tryGetSocketOptionInt`/`tryGetSocketOptionBytes` (filter.cc:190, 202).

## Decision points
- Module load failure -> factory-time `InvalidArgumentError` (factory.cc:21).
- Module filter creation failure -> per-connection close with `NoFlush` (filter.cc:89-94).
- Missing optional symbols -> early return in the watermark / scheduled hooks (filter.cc:136, 142, 150).
- Cross-thread safety -> `DynamicModuleNetworkFilterScheduler::commit` (filter.h:224) locks `weak_ptr` before touching dispatcher.

## Configuration fields
- `dynamic_module_config.name` — name of the `.so` to load.
- `dynamic_module_config.do_not_close` / `load_globally` — `dlopen` flags propagated to `newDynamicModuleByName`.
- `dynamic_module_config.metrics_namespace` — prefix prepended to each metric the module declares (factory.cc:34).
- `filter_name` — passed to `on_network_filter_config_new_` so the module can distinguish multiple config instances.
- `filter_config` (google.protobuf.Any, optional) — raw bytes delivered to the module.
- `terminal_filter` — controls whether the chain may contain additional filters after this one.

## Stats
The filter itself emits no built-in counters. The module declares its own counters/gauges/histograms during config init via the ABI; those are stored in `counters_`/`gauges_`/`histograms_` on the `DynamicModuleNetworkFilterConfig` (filter_config.h:192-194) and surfaced in `stats_scope_`, which lives under `<metrics_namespace>.` (filter_config.cc:20). Increments happen inside the module via `ModuleCounterHandle::add` / `ModuleGaugeHandle::{add,sub,set}` / `ModuleHistogramHandle::recordValue` (filter_config.h:100-127).
