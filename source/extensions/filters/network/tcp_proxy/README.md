# TCP Proxy (`envoy.filters.network.tcp_proxy`)

L3/L4 bidirectional byte proxy. Given a downstream TCP connection, it picks a route and cluster, opens an upstream connection via a connection pool, and shuttles bytes between the two. Supports weighted cluster selection, on-demand cluster discovery (ODCDS), idle/max-duration timeouts, HTTP CONNECT/POST tunneling, proxy-protocol TLV propagation/merging, and delayed connection-establishment modes that let preceding filters read data (e.g., SNI) before the upstream is dialed.

Proto: `envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy`.

## Files
This extension folder is a thin factory layer; the implementation lives in `source/common/tcp_proxy/`.
- `config.h/cc` — `ConfigFactory`, registered as `envoy.filters.network.tcp_proxy` (`config.h:20`). `createFilterFactoryFromProtoTyped` builds an `Envoy::TcpProxy::Config` and installs an `Envoy::TcpProxy::Filter` as a read filter (`config.cc:14`). Registered via `LEGACY_REGISTER_FACTORY` with the deprecated alias `envoy.tcp_proxy` (`config.cc:30`).

The rest of this document describes the code under `source/common/tcp_proxy/`:
- `tcp_proxy.h/cc` — `Config` / `Config::SharedConfig` (route table, stats, tunneling, ODCDS, idle/duration timeouts, proxy-protocol TLVs), `Filter` (the `Network::ReadFilter`), `Drainer`, `UpstreamDrainManager`.
- `upstream.h/cc` — `TcpConnPool`, `HttpConnPool`, `TcpUpstream`, `HttpUpstream`, `CombinedUpstream`, `RouterUpstreamRequest`; implementations of `GenericConnPool` / `GenericUpstream` used for plain-TCP, HTTP CONNECT tunneling, and the HTTP-router-filter path.

## Major components

### `Config` / `SharedConfig`
`Config` (`tcp_proxy.h:235`) holds the route set, `max_connect_attempts_`, backoff, access logs, random/regex, `upstream_connect_mode_`, `max_early_data_bytes_`, and a TLS slot used by `UpstreamDrainManager`. `SharedConfig` (`tcp_proxy.h:240`) is the subset safe to share with `Drainer`s that outlive the listener: stats, idle/duration timeouts, flush flags, tunneling helper, ODCDS, backoff, proxy-protocol TLVs and merge policy. `getRouteFromEntries` chooses between `default_route_` and `weighted_clusters_`, with optional `PerConnectionCluster` filter-state override (`tcp_proxy.h:324`, `:443`).

### `Filter` (the L4 proxy)
`Filter` (`tcp_proxy.h:459`) is both a `Network::ReadFilter` and `GenericConnectionPoolCallbacks`; it also implements `Upstream::LoadBalancerContextBase` so its hash policy and metadata match propagate into LB.

### Upstream types (`upstream.h`)
- `TcpConnPool` (`upstream.h:32`): wraps an `Upstream::TcpPoolData`, returns a `TcpUpstream` on `onPoolReady` and calls `GenericConnectionPoolCallbacks::onGenericPoolReady/Failure`.
- `HttpConnPool` (`upstream.h:70`): creates either a CONNECT/POST tunnel (`HttpUpstream`) or a `CombinedUpstream` that routes via upstream HTTP filters when `envoy.restart_features.upstream_http_filters_with_tcp_proxy` is enabled (`upstream.h:98`).
- `GenericUpstream` implementations forward `encodeData`, `readDisable`, and downstream events back to the connection/stream.

### Drain path
When the downstream closes but the upstream still has buffered bytes, `Filter::onDownstreamEvent` hands the upstream off to `UpstreamDrainManager::add` (`tcp_proxy.cc:985`) which owns a `Drainer` (`tcp_proxy.h:768`). The drainer keeps the upstream alive until the buffer flushes, an idle timeout fires (`Drainer::onIdleTimeout`), or the upstream closes.

## Lifecycle

### `onNewConnection()` (`tcp_proxy.cc:897`)
1. Start `connection_duration_timer_` if `max_downstream_connection_duration` is set (with optional jitter) (`:898`).
2. Start `access_log_flush_timer_` when configured (`:904`).
3. Resolve idle timeout (config or per-connection filter-state override via `PerConnectionIdleTimeoutMs`) and arm `idle_timer_` (`:909`).
4. Assign stream UUID and emit start-of-connection access log if `flush_access_log_on_start_` (`:926`).
5. Branch on `connect_mode_`:
   - `IMMEDIATE`: pick route and call `establishUpstreamConnection()` right away (`:934`).
   - `ON_DOWNSTREAM_DATA`: pre-pick route, return `Continue`/`StopIteration` depending on `receive_before_connect_` so later filters or `onData` can drive connection (`:945`).
   - `ON_DOWNSTREAM_TLS_HANDSHAKE`: wait for the `Connected` event reported by the TLS transport socket (see `onDownstreamEvent`).

### `initializeReadFilterCallbacks` / `initialize` (`tcp_proxy.cc:335`)
- Register `downstream_callbacks_`, enable half-close (`:341`).
- Resolve `connect_mode_`; detect legacy `ReceiveBeforeConnectKey` filter state or new `max_early_data_bytes` and set `receive_before_connect_`/`max_buffered_bytes_` (`:349`).
- If not receiving-before-connect, `readDisable(true)` so we do not drain downstream into a missing upstream (`:383`).
- Initialize byte meters, set upstream info, increment `downstream_cx_total`, and wire downstream byte stats (`:390`).

### `onData(data, end_stream)` (`tcp_proxy.cc:839`)
- If `upstream_` is already established: `upstream_->encodeData(data, end_stream)` and `resetIdleTimer()` (`:846`).
- Else if `receive_before_connect_`: buffer into `early_data_buffer_`, record `end_stream`, possibly trigger `establishUpstreamConnection()` when the first real byte arrives in `ON_DOWNSTREAM_DATA` mode, and `readDisable(true)` when buffered bytes exceed `max_buffered_bytes_` (`:850`).
- Always returns `StopIteration`: the filter owns both ends of the byte stream.

### `establishUpstreamConnection()` (`tcp_proxy.cc:525`)
1. Look up the `ThreadLocalCluster` by `route_->clusterName()`; if missing, either start ODCDS (`:540`) or fail with `NoRoute` (`:536`).
2. Check connection-resource budget via `resourceManager(...).connections().canCreate()`; fail with `ResourceLimitExceeded` if over (`:559`).
3. Populate/merge a `ProxyProtocolFilterState` using `proxyProtocolTlvMergePolicy` (`ADD_IF_ABSENT`, `OVERWRITE_BY_TYPE_IF_EXISTS_OR_ADD`, `APPEND_IF_EXISTS_OR_ADD`, `:569`-`:613`).
4. Derive transport-socket options and upstream socket options from filter state (`:615`).
5. Call `maybeTunnel(cluster)` which picks a `GenericConnPoolFactory` (`envoy.filters.connection_pools.tcp.generic` by default) and kicks off `newStream` on the resulting `GenericConnPool`; failure translates to `NoHealthyUpstream` (`:625`). On success the pool will call `onGenericPoolReady` or `onGenericPoolFailure`.

### `onGenericPoolReady` / `onGenericPoolFailure` (`tcp_proxy.cc:729`, `:698`)
Ready: adopt the `GenericUpstream`, expose `UpstreamCallbacks`, re-enable downstream reads, flush any `early_data_buffer_`, record `upstream_cx_connect_ms`, and transition to the steady-state. Failure: `ConnectFailed`, retry via `onRetryTimer` until `max_connect_attempts_`, then `onInitFailure(ConnectFailed)` (`:1068`, `:1193`).

### Steady state
`UpstreamCallbacks::onUpstreamData` (`tcp_proxy.cc:495`) writes upstream bytes to the downstream connection and resets the idle timer. Watermark callbacks push pause/resume signals across via `readDisableUpstream`/`readDisableDownstream` (`:412`, `:427`). `onDownstreamEvent`/`onUpstreamEvent` (`:964`, `:1010`) propagate half-close and remote RST. Idle timeout (`:1132`) and max-duration timer (`:1140`) close the downstream with dedicated response-flag reasons.

## Decision points (selected branches)

- Weighted vs. simple route: `Config::getRouteFromEntries` (`tcp_proxy.h:324`) consults `weighted_clusters_` with `random_generator_`, otherwise returns `default_route_`.
- Tunneling vs. plain TCP: `maybeTunnel` (`tcp_proxy.cc:666`) inspects the cluster's `upstreamConfig()` for a typed `GenericConnPoolFactory`.
- Legacy vs. new early-data: `Filter::initialize` (`tcp_proxy.cc:353`) — `max_early_data_bytes` wins; otherwise falls back to `ReceiveBeforeConnectKey` filter state with `max_buffered_bytes_ = 0` (immediate read-disable).
- ODCDS outcome routing: `onClusterDiscoveryCompletion` (`tcp_proxy.cc:638`) maps `Missing`/`Timeout` to failure and `Available` back into `establishUpstreamConnection()`.
- Drain decision on downstream close: `onDownstreamEvent` (`tcp_proxy.cc:983`) hands the upstream to `drainManager().add()` only if `onDownstreamEvent` returned a live `conn_data`.

## Configuration (highlights)
- `cluster` or `weighted_clusters` — target cluster(s). Per-connection override via `PerConnectionCluster` filter state (`tcp_proxy.h:443`).
- `stat_prefix` — required (asserted in `config.cc:17`).
- `max_connect_attempts`, `backoff_options` — retry budget and delay schedule.
- `idle_timeout`, `max_downstream_connection_duration` (+ jitter), `access_log_flush_interval`, `access_log`, `flush_access_log_on_*`.
- `tunneling_config` — activates HTTP CONNECT/POST tunneling via `TunnelingConfigHelperImpl` (`tcp_proxy.h:163`).
- `on_demand` — enables ODCDS (`OnDemandConfig`, `tcp_proxy.h:210`).
- `hash_policy`, `metadata_match` — propagate into the load balancer context.
- `proxy_protocol_tlvs`, `proxy_protocol_tlv_merge_policy`, `dynamic_tlvs` — forwarded via `ProxyProtocolFilterState`.
- `upstream_connect_mode` — `IMMEDIATE` / `ON_DOWNSTREAM_DATA` / `ON_DOWNSTREAM_TLS_HANDSHAKE`.
- `max_early_data_bytes` — buffer size for receive-before-connect.

## Stats
Prefix is `<stat_prefix>.`. See `ALL_TCP_PROXY_STATS` (`tcp_proxy.h:61`) and `ON_DEMAND_TCP_PROXY_STATS` (`tcp_proxy.h:79`):
- Counters: `downstream_cx_total`, `downstream_cx_no_route`, `downstream_cx_rx_bytes_total`, `downstream_cx_tx_bytes_total`, `downstream_flow_control_paused_reading_total`, `downstream_flow_control_resumed_reading_total`, `early_data_received_count_total`, `idle_timeout`, `max_downstream_connection_duration`, `upstream_flush_total`.
- Gauges: `downstream_cx_rx_bytes_buffered`, `downstream_cx_tx_bytes_buffered`, `upstream_flush_active`.
- ODCDS (only when enabled): `on_demand_cluster_attempt`, `on_demand_cluster_missing`, `on_demand_cluster_timeout`, `on_demand_cluster_success`.
- Cluster-level `upstream_cx_*` and `upstream_rq_*` stats are incremented via the standard connection pool path, not inside this filter.

## Factory
`ConfigFactory::createFilterFactoryFromProtoTyped` (`config.cc:14`) instantiates `Envoy::TcpProxy::Config` and returns a lambda that calls `filter_manager.addReadFilter(std::make_shared<Envoy::TcpProxy::Filter>(...))` with the cluster manager from the server factory context. Registered via `LEGACY_REGISTER_FACTORY` at `config.cc:30`.
