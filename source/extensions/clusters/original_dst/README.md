# Original Destination Cluster (`envoy.cluster.original_dst`)

A cluster that *discovers hosts lazily from incoming requests*. On each `chooseHost()`, the load balancer extracts a destination address from (in priority order) filter state, dynamic metadata, an HTTP header override, or - most commonly - the downstream connection's `localAddress()` when `SO_ORIGINAL_DST` has been restored. If a host for that address already exists in the map it is reused; otherwise a new `HostImpl` is synthesized inline on the worker thread and also posted to the main thread to be merged into the authoritative host map. A periodic cleanup timer evicts hosts that have not been used since the previous tick.

Proto: driven by the core `envoy::config::cluster::v3::Cluster` message (fields read: `cleanup_interval`, `original_dst_lb_config.use_http_header`, `original_dst_lb_config.http_header_name`, `original_dst_lb_config.upstream_port_override`, `original_dst_lb_config.metadata_key`).

## Files
- `original_dst_cluster.h` / `original_dst_cluster.cc` - `OriginalDstCluster`, `OriginalDstClusterHandle`, inner `LoadBalancer`, `LoadBalancerFactory`, `ThreadAwareLoadBalancer`, `OriginalDstClusterFactory`, and helper structs `HostsForAddress`, `HostMultiMap`.

## Cluster type
- `OriginalDstCluster` extends `Upstream::ClusterImplBase` (`original_dst_cluster.h:68`).
- `initializePhase()` returns `Primary` (`original_dst_cluster.h:76`) - no external dependency; initialization is synchronous via an empty `startPreInit()` that immediately calls `onPreInitComplete()` (`original_dst_cluster.h:175`).
- Wrapped in an `OriginalDstClusterHandle` so the destructor always runs on the main thread: `OriginalDstClusterHandle::~OriginalDstClusterHandle()` posts a final `reset()` onto `dispatcher_` (`original_dst_cluster.cc:25-30`).

## Initialization / host set
- The host map is a `HostMultiMap = absl::flat_hash_map<std::string, HostsForAddressSharedPtr>` keyed by the address string (`original_dst_cluster.h:41-43`). `HostsForAddress` holds a canonical `host_` (set by the first worker that posted) plus any concurrent duplicates in `hosts_` (read/write on main thread only), and an atomic `used_` flag for GC (`original_dst_cluster.h:25-38`).
- Readers (workers) grab a snapshot via `getCurrentHostMap()` under a reader lock (`original_dst_cluster.h:161-164`); writers (main thread) swap in a new map via `setHostMap(...)` under the writer lock (`original_dst_cluster.h:166-169`). This is copy-on-write: `addHost` builds a new `HostMultiMap` then publishes it.
- `cleanup_timer_` fires every `cleanup_interval_ms_` and calls `cleanup()` which walks the current map, keeps any entry whose `used_` flag is true (resetting it to false), and drops the rest. Workers keep their snapshot alive so in-flight connections are unaffected.

## Load balancing hooks
- `ThreadAwareLoadBalancer` -> `LoadBalancerFactory` -> `LoadBalancer` (one per worker, `original_dst_cluster.h:138-159`).
- `LoadBalancer::chooseHost(context)` (`original_dst_cluster.cc:32+`):
  1. `filterStateOverrideHost(context)` - read from `envoy.network.transport_socket.original_dst_address` filter state if present.
  2. Else `metadataOverrideHost(context)` - reads `metadata_key_`.
  3. Else `requestOverrideHost(context)` - reads the configured HTTP header.
  4. Else `connection->connectionInfoProvider().localAddress()` when `localAddressRestored()` is true.
  5. If `port_override_` is set, replace the port via `Network::Utility::getAddressWithPort(...)`.
  6. Look up `dst_addr.asString()` in the worker's snapshot `host_map_`; on hit return the cached host and set `used_=true`. On miss, synthesize a new `HostImpl` (weight 1, default locality, UNKNOWN health), post it to the main thread through `parent_->cluster_->dispatcher_.post(...)`, and return it immediately so the request does not block.
- `peekAnotherHost`, `selectExistingConnection`, `lifetimeCallbacks` are all no-ops (`original_dst_cluster.h:100-111`).

## Key decision points
- Host-source priority chain (filter state -> metadata -> header -> restored local address) - `original_dst_cluster.cc:33-54`.
- Port override precedence before lookup - `original_dst_cluster.cc:55-57`.
- Cross-thread host registration via `dispatcher_.post(...)` with a `weak_ptr<OriginalDstClusterHandle>` guard against cluster disappearance - `original_dst_cluster.cc:87-94`.
- Used-flag-based GC pass keeps any host seen in the last interval - see `cleanup()`.
- Main-thread-only destruction contract enforced by `OriginalDstClusterHandle` - `original_dst_cluster.cc:25-30`.

## Configuration
- `cleanup_interval` (default 5000ms).
- `original_dst_lb_config.use_http_header` + `http_header_name` (lower-case string stored in `http_header_name_`).
- `original_dst_lb_config.metadata_key` - optional metadata path providing the destination address.
- `original_dst_lb_config.upstream_port_override` - overrides the port regardless of source.

## Stats
- Standard `ClusterImplBase` membership and request stats. Host add/remove activity is observable via the standard `membership_*` gauges.
