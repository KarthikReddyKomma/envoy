# Dynamic Forward Proxy Cluster (`envoy.clusters.dynamic_forward_proxy`)

Produces hosts on-demand from the HTTP filter chain: whenever the DFP HTTP filter sees a new `host:port`, it asks the shared DNS cache to resolve it; resolution results flow back to this cluster as `onDnsHostAddOrUpdate` callbacks and the cluster materializes one `LogicalHost` per cache entry inside priority 0. Optionally, a "sub-cluster" mode promotes each `host:port` into a full child `STRICT_DNS` cluster managed in the cluster manager so that per-origin load-balancing / circuit-breaking / stats are available.

Proto: `envoy.extensions.clusters.dynamic_forward_proxy.v3.ClusterConfig` (see `api/envoy/extensions/clusters/dynamic_forward_proxy/v3/cluster.proto`).

## Files
- `cluster.h` / `cluster.cc` - `Cluster`, inner `LoadBalancer`, `DFPHostSelectionHandle`, `LoadBalancerFactory`, `ThreadAwareLoadBalancer`, `ClusterFactory`, plus `StreamInfo::FilterState::ObjectFactory` factories for `envoy.upstream.dynamic_host` and `envoy.upstream.dynamic_port`.

## Cluster type
- `Cluster` extends `Upstream::BaseDynamicClusterImpl` and implements two extra interfaces (`cluster.h:22-24`):
  - `Common::DynamicForwardProxy::DfpCluster` - contract consumed by the DFP HTTP filter (`findHostByName`, `createSubClusterConfig`, `touch`, `enableSubCluster`).
  - `Common::DynamicForwardProxy::DnsCache::UpdateCallbacks` - receives `onDnsHostAddOrUpdate`, `onDnsHostRemove`, `onDnsResolutionComplete`.
- `initializePhase()` returns `Primary` (`cluster.h:29`).

## Initialization / host set
- Constructor (`cluster.cc:64-91`):
  - Resolves a shared `DnsCache` from the supplied `DnsCacheManager` and calls `addUpdateCallbacks(*this)` to register as a cache observer (`cluster.cc:73`).
  - Captures `orig_cluster_config_` verbatim so sub-cluster mode can clone the transport socket, TLS, etc.
  - If `sub_clusters_config` is set, arms `idle_timer_` on the main-thread dispatcher to periodically call `checkIdleSubCluster()` (`cluster.cc:87-90`).
- `startPreInit()` (`cluster.cc:112-126`): iterates the pre-populated cache via `dns_cache_->iterateHostMap(...)`, calls `addOrUpdateHost` for each entry, then `updatePriorityState(*hosts_added, {})` and finally `onPreInitComplete()`.
- `addOrUpdateHost()` (`cluster.cc:256-314`): under `host_map_lock_`, either (a) calls `LogicalHost::setNewAddresses(...)` on an existing entry for an in-place IP swap (no priority-set rebuild, see `cluster.cc:272-293`) or (b) constructs a new `Upstream::LogicalHost` with the `dummy_locality_lb_endpoint_` / `dummy_lb_endpoint_` and inserts it into `host_map_`.
- `onDnsHostRemove` (`cluster.cc:346+`) erases the entry and calls `updatePriorityState({}, hosts_removed)`.
- `updatePriorityState` (`cluster.cc:330-344`) rebuilds priority 0 via `PriorityStateManager`, registering every current logical host.

## Load balancing hooks
- `ThreadAwareLoadBalancer::factory()` returns a `LoadBalancerFactory` that creates one `LoadBalancer` per worker (`cluster.h:186-193`).
- `LoadBalancer::chooseHost` reads a hostname from either a `DynamicHostFilterStateKey` filter-state string, the HTTP `Host` header, or authority metadata (see `cluster.cc:chooseHost` path), then calls `cluster.findHostByName(hostname)` to resolve against `host_map_`. If not found, it returns a `DFPHostSelectionHandle` (`cluster.h:149-180`) that holds a DNS cache load handle; when the async DNS resolution finishes, `onLoadDnsCacheComplete` re-runs `findHostByName` and calls `context_->onAsyncHostSelection(host, details)`.
- `LoadBalancer` also acts as a `Http::ConnectionPool::ConnectionLifetimeCallbacks` so that H2/H3 connections can be reused across origins when `allow_coalesced_connections_` is true (tracked in `connection_info_map_`, `cluster.h:122-142`).
- Sub-cluster mode: when `enable_sub_cluster_` is true, `createSubClusterConfig` (`cluster.cc:158-200`) clones `orig_cluster_config_`, forces `type=STRICT_DNS`, sets the single `host:port` endpoint, and pushes the new `Cluster` into the cluster manager. `chooseHost(host, context)` (`cluster.cc:202-226`) looks up that child cluster via `cm_.getThreadLocalCluster(cluster_name)` and delegates to its LB.

## Key decision points
- In-place address swap vs. new host creation - `cluster.cc:270-305`.
- Sub-cluster TTL eviction - `cluster.cc:139-156`, `cluster.cc:240-254` (`ClusterInfo::checkIdle`).
- Sub-cluster cap - `max_sub_clusters_` (default 1024) enforced at `cluster.cc:168-172`.
- Connection coalescing behavior under `allow_coalesced_connections_` - `cluster.h:45`, `LoadBalancer::onConnectionOpen/Draining`.
- Destruction removes every sub-cluster from the cluster manager to prevent leaks - `cluster.cc:93-110`.

## Configuration
- `dns_cache_config` (resolved into a shared `DnsCache`).
- `allow_coalesced_connections` (bool).
- `sub_clusters_config`: `{ max_sub_clusters, sub_cluster_ttl (default 300s), lb_policy }`.

## Stats
- Beyond the `ClusterImplBase` defaults, the shared `DnsCache` emits its own stats (cache size, resolve attempts/success/failure). When sub-cluster mode is on, each per-origin `STRICT_DNS` child cluster carries its own full statistics namespace.
