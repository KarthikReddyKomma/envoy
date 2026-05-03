# SNI Cluster (`envoy.filters.network.sni_cluster`)

Non-terminal L4 filter that copies the TLS SNI value from the downstream connection into the per-connection filter state under the key consumed by `tcp_proxy` (`TcpProxy::PerConnectionCluster`). When placed in front of a `tcp_proxy` filter whose cluster is unset or dynamically resolved, this causes the upstream cluster to be selected by the SNI the client presented. The filter does no DNS work; it is a one-line handoff that runs once per connection and then goes quiet.

Proto: `envoy.extensions.filters.network.sni_cluster.v3.SniCluster` (empty message — there is nothing to configure).

## Files
- `sni_cluster.h` — `SniClusterFilter` class: the `onData` override is inlined as a no-op returning `Continue` (sni_cluster.h:19-21); real work is in `onNewConnection`.
- `sni_cluster.cc` — body of `SniClusterFilter::onNewConnection()`.
- `config.h/cc` — `SniClusterNetworkFilterConfigFactory` registration.

## Lifecycle
- `onNewConnection()` (sni_cluster.cc:13) — single-shot. Reads `read_callbacks_->connection().requestedServerName()` (the SNI that `ListenerFilter` previously extracted), logs at trace, and — when non-empty — sets `TcpProxy::PerConnectionCluster` on the filter state. Returns `Continue`.
- `onData()` (sni_cluster.h:19) — always returns `Continue`; does not inspect payload.
- `initializeReadFilterCallbacks()` (sni_cluster.h:23) — stores the callbacks pointer.

No `onWrite()`; pure `ReadFilter`.

## Decision / logic
- Empty-SNI short-circuit (sni_cluster.cc:18): when no SNI is present the filter does nothing and the downstream cluster selection falls back to whatever `tcp_proxy` is otherwise configured with (static cluster, weighted clusters, cluster header, etc.).
- Filter-state write (sni_cluster.cc:21-24): uses the SNI string as the cluster name, installs it `Mutable` and at `LifeSpan::Connection`. Being mutable lets a later filter (e.g. a custom policy filter) overwrite the value before `tcp_proxy` reads it. The key is `TcpProxy::PerConnectionCluster::key()` (sni_cluster.cc:22) which `tcp_proxy` queries in its own `onNewConnection` path.

## Configuration
None. The proto `SniCluster` is empty and accepted as-is by the factory.

## Stats
None.

## Factory
`SniClusterNetworkFilterConfigFactory` (config.h:15) implements `NamedNetworkFilterConfigFactory` directly (not via `FactoryBase`). `createFilterFactoryFromProto` ignores both arguments and returns a lambda that adds a `SniClusterFilter` to the filter manager (config.cc:16-21). `createEmptyConfigProto` returns the empty v3 message (config.cc:23-25). The registered name is `NetworkFilterNames::get().SniCluster`. Registration is via `REGISTER_FACTORY` (config.cc:30). The factory does not override `isTerminalFilter`, so it is non-terminal by default — which is correct since `tcp_proxy` is expected to follow it in the chain.
