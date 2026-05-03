# Drain-Aware HTTP Connection Manager (`envoy.filters.network.reverse_tunnel.drain_aware_hcm`)

Thin wrapper around the standard `http_connection_manager` network filter that makes the HCM react to the listener-level drain decision. The stock HCM only honours the server-wide drain manager (hot-restart / full shutdown), so reverse-tunnel listeners that are drained via the admin `/drain_listeners` endpoint never emit an HTTP/2 `GOAWAY`. This filter swaps in a `DrainAwareServerConnection` codec that polls `FactoryContext::drainDecision()` and, when drain fires, forwards a single `goAway()` to the inner codec so in-flight streams finish cleanly and new streams are refused.

Proto: `envoy.extensions.filters.network.reverse_tunnel.v3.DrainAwareHttpConnectionManager` (wraps the standard `envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager` under `hcm_config`).

## Files
- `drain_aware_config.h/cc` — `DrainAwareHttpConnectionManagerConfig` (subclass of `HttpConnectionManager::HttpConnectionManagerConfig`) overriding codec creation; factory `DrainAwareHttpConnectionManagerFilterConfigFactory`.
- `drain_aware_server_connection.h` — `DrainAwareServerConnection`, the inline codec wrapper that implements the drain poll and GOAWAY.

## Lifecycle
This filter does not override `onData` / `onWrite` itself; it delegates to `Http::ConnectionManagerImpl` constructed in the factory lambda (drain_aware_config.cc:42-44). The drain-aware behaviour lives in the codec wrapper:
- Construction (`DrainAwareServerConnection::DrainAwareServerConnection`, drain_aware_server_connection.h:22) arms a 100 ms `Event::Timer` on the connection's dispatcher.
- On each tick `onDrainCheckTimer()` (drain_aware_server_connection.h:45) calls `drain_decision_.drainClose(DrainDirection::All)`; once it returns true, it sets `drain_goaway_sent_`, invokes `inner_->goAway()`, and stops re-arming. Otherwise it re-arms the timer for another 100 ms.
- Destructor (drain_aware_server_connection.h:28) disables the timer.
- All codec methods (`dispatch`, `goAway`, `protocol`, `shutdownNotice`, `wantsToWrite`, watermark hooks) forward unmodified to the wrapped inner codec (drain_aware_server_connection.h:34-42).

## Decision / logic
- `DrainAwareHttpConnectionManagerConfig::createCodec` (drain_aware_config.cc:17) delegates to `createBaseCodec` (drain_aware_config.cc:13) — the overridable seam for tests that returns the standard HCM codec — then, if non-null, wraps it with `DrainAwareServerConnection` constructed with `factory_context_.drainDecision()` (drain_aware_config.cc:27). The comment at drain_aware_config.cc:24-26 explains why the listener-level `drainDecision()` is used instead of `serverFactoryContext().drainManager()`.
- Nullptr defensive path: when `createBaseCodec` returns null, the override logs a warning and returns the null pointer unchanged (drain_aware_config.cc:19-22).
- Factory lambda (drain_aware_config.cc:39-45) picks the overload manager based on `context.listenerInfo().shouldBypassOverloadManager()` (drain_aware_config.cc:41) and constructs a plain `Http::ConnectionManagerImpl` as the read filter, passing the custom `filter_config` so that subsequent codec creation goes through the override.

## Configuration
- `hcm_config` — the full inner HttpConnectionManager proto. All HCM options (codec type, route config, HTTP filters, tracing, access logs) apply as usual (drain_aware_config.cc:31).
- Listener-level `drain_type` / admin `/drain_listeners` drives when `drainClose` returns true.
- The 100 ms poll cadence is a hard-coded constant (drain_aware_server_connection.h:25, 55).

## Stats
None emitted directly by this wrapper. All stats are those produced by the wrapped HCM (`http.<stat_prefix>.*`) and by the underlying HTTP/1 or HTTP/2 codec.

## Factory
`DrainAwareHttpConnectionManagerFilterConfigFactory` (drain_aware_config.h:34) extends `Common::ExceptionFreeFactoryBase<DrainAwareHttpConnectionManager>`, is marked terminal (`true` in its base ctor, drain_aware_config.h:36) under the name `NetworkFilterNames::get().ReverseTunnelDrainAwareHcm`, and is registered via `REGISTER_FACTORY` (drain_aware_config.cc:48). `createFilterFactoryFromProtoTyped` (drain_aware_config.cc:30) reuses the HCM extension's shared singletons (`HttpConnectionManager::Utility::createSingletons`, drain_aware_config.cc:32) and builds a `DrainAwareHttpConnectionManagerConfig` with them, propagating creation errors through `creation_status`.
