# Dynamic Modules Listener Filter (`envoy.filters.listener.dynamic_modules`)

Runs a listener filter whose logic is implemented in an externally loaded dynamic module (shared library). Envoy resolves a fixed set of C ABI symbols from the module, then delegates every filter hook (`onAccept`, `onData`, `onClose`, `maxReadBytes`, HTTP callout completion, scheduled events) to the module through those entry points. A rich ABI surface in `abi_impl.cc` lets the module inspect and mutate `ConnectionSocket` state (SNI, ALPN, detected transport, JA3/JA4, restored local and remote addresses, socket options), read/write filter state and dynamic metadata, drain the listener filter buffer, send HTTP callouts, and define/emit Envoy stats.

Proto: `envoy.extensions.filters.listener.dynamic_modules.v3.DynamicModuleListenerFilter`.

## Files
- `factory.h` / `factory.cc` — `DynamicModuleListenerFilterConfigFactory`: loads the dynamic module, resolves symbols, builds the shared `DynamicModuleListenerFilterConfig`, and returns the factory callback; registers under `envoy.filters.listener.dynamic_modules` (`factory.cc:26`, `factory.cc:79`).
- `filter_config.h` / `filter_config.cc` — `DynamicModuleListenerFilterConfig` holds the resolved ABI function pointers, the opaque in-module config handle, a dedicated `Stats::Scope` (prefixed with the metrics namespace), and the metric handle tables used by counter/gauge/histogram ABI callbacks (`filter_config.cc:14`, `filter_config.cc:36`).
- `filter.h` / `filter.cc` — `DynamicModuleListenerFilter` implements `Network::ListenerFilter`, owns the in-module filter handle, caches the original destination for ABI lifetime, and drives HTTP callouts via `HttpCalloutCallback`.
- `abi_impl.cc` — C ABI callbacks the module invokes on the filter (set/get SNI, transport protocol, application protocols, JA3/JA4, addresses, original dst, socket options, filter state, dynamic metadata, metrics, HTTP callout, scheduler).

## Lifecycle
- `onAccept(cb)` — records callbacks, parses the worker index from `dispatcher().name()`, lazily constructs the in-module filter via `on_listener_filter_new_`, and if construction fails closes the socket and returns `StopIteration` (`filter.cc:49`-`filter.cc:69`). Otherwise calls `on_listener_filter_on_accept_` and converts the returned module status (`toEnvoyFilterStatus` at `filter.cc:10`).
- `onData(buffer)` — caches `current_buffer_` so ABI callbacks like `..._get_buffer_chunk` / `..._drain_buffer` (`abi_impl.cc:24`, `abi_impl.cc:40`) work, then invokes `on_listener_filter_on_data_` with the current slice length; `current_buffer_` is cleared after the call (`filter.cc:72`-`filter.cc:85`).
- `onClose()` — forwards to `on_listener_filter_on_close_` (`filter.cc:87`).
- `maxReadBytes()` — asks the module via `on_listener_filter_get_max_read_bytes_`; returns 0 if the in-module filter failed to initialize (`filter.cc:94`).

## Decision / logic
- `destroy()` cancels all outstanding HTTP callouts before invoking `on_listener_filter_destroy_` and sets `destroyed_ = true` (`filter.cc:33`).
- `onScheduled(event_id)` is posted by `DynamicModuleListenerFilterScheduler::commit` from any thread, and re-enters the module only if the filter is still alive (`filter.h:141`, `filter.cc:102`).
- ABI mutators on the socket: `setDetectedTransportProtocol` (`abi_impl.cc:52`), `setRequestedServerName` (`abi_impl.cc:60`), `setRequestedApplicationProtocols` (`abi_impl.cc:68`), `setJA3Hash` / `setJA4Hash` (`abi_impl.cc:85`, `abi_impl.cc:93`), `setRemoteAddress` (`abi_impl.cc:507`), `restoreLocalAddress` (`abi_impl.cc:532`), `useOriginalDst` (`abi_impl.cc:565`).
- Accessors: `getRequestedServerName` / `getDetectedTransportProtocol` / `getRequestedApplicationProtocols*` / `getJA3Hash` / `getJA4Hash` / `is_ssl` / `getSslUriSans` / `getSslDnsSans` / `getSslSubject` / remote/local/direct/original addresses (`abi_impl.cc:101`-`abi_impl.cc:473`).
- Original destination retrieval uses `Network::Utility::getOriginalDst` and caches the shared pointer on the filter so address memory survives the ABI call (`abi_impl.cc:432`, `filter.h:37`).
- Filter-chain control: `envoy_dynamic_module_callback_listener_filter_continue_filter_chain` invokes `callbacks->continueFilterChain(success)` (`abi_impl.cc:557`), allowing asynchronous completion after `onAccept`/`onData` returned `StopIteration`.
- HTTP callout: `sendHttpCallout` resolves the cluster, posts via `Http::AsyncClient`, and records the pending request in `http_callouts_` keyed by a monotonically-increasing id (`filter.cc:109`). On success/failure the module is notified via `on_listener_filter_http_callout_done_` (`filter.cc:139`, `filter.cc:181`).

## Configuration
- `dynamic_module_config.name` — library name passed to `Extensions::DynamicModules::newDynamicModuleByName` (`factory.cc:24`).
- `dynamic_module_config.do_not_close`, `load_globally` — load flags (`factory.cc:25`).
- `dynamic_module_config.metrics_namespace` — prefix for module-defined stats; when the runtime flag `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix` is enabled, it is registered as a custom stat namespace (`factory.cc:42`, `factory.cc:61`). Defaults to `dynamicmodulescustom` (`filter_config.h:33`).
- `filter_name`, `filter_config` — opaque string/bytes passed to the module via `on_listener_filter_config_new` (`factory.cc:31`, `filter_config.cc:112`).

## Stats
All counter/gauge/histogram names are defined by the module at runtime via `envoy_dynamic_module_callback_listener_filter_config_define_{counter,gauge,histogram}` (`abi_impl.cc:871`, `abi_impl.cc:889`, `abi_impl.cc:927`) and emitted under the configured `metrics_namespace` scope (`filter_config.cc:20`). Envoy itself emits no fixed stats for this filter.
