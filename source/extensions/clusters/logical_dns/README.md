# Logical DNS Cluster (`envoy.cluster.logical_dns`)

A cluster with *exactly one logical host* whose backing IP is periodically refreshed by DNS. The host identity never changes, so connection pools that bind to it survive DNS rotations, yet new connections pick up whatever address the resolver just returned. Useful for fronting round-robin DNS endpoints where preserving long-lived keep-alive connections matters more than fanning out across every resolved IP.

Proto: `envoy.extensions.clusters.dns.v3.DnsCluster` (shared with `dns` / `strict_dns`; see `api/envoy/extensions/clusters/dns/v3/dns_cluster.proto`).

This class is the *legacy* implementation; when `envoy.reloadable_features.enable_new_dns_implementation` is on, the `dns` factory delegates to the unified `DnsClusterImpl` (see `../dns/`) instead.

## Files
- `logical_dns_cluster.h` / `logical_dns_cluster.cc` - `LogicalDnsCluster`.

## Cluster type
- `LogicalDnsCluster` extends `Upstream::ClusterImplBase` (`logical_dns_cluster.h:39`) rather than `BaseDynamicClusterImpl` because membership never really changes - the same single `LogicalHost` sits in priority 0 for the life of the cluster; only its address is swapped.
- `initializePhase()` returns `Primary` (`logical_dns_cluster.h:44`).

## Initialization / host set
- `create(...)` (`logical_dns_cluster.cc:50-80`) enforces:
  - exactly one `LocalityLbEndpoints` and exactly one `LbEndpoint` - otherwise `InvalidArgumentError` (`logical_dns_cluster.cc:57-64`).
  - no custom DNS resolver name on the endpoint - `InvalidArgumentError` at `logical_dns_cluster.cc:68-70`.
- Ctor (`logical_dns_cluster.cc:82+`):
  - Snapshots the `ClusterLoadAssignment` via `convertPriority(...)` which rewrites every endpoint priority to 0 (`logical_dns_cluster.cc:30-47`). This protects zone-aware routing, which requires priority 0 across the board.
  - Copies DNS knobs: `dns_refresh_rate` (default 5000ms), `dns_jitter` (default 0), `respect_dns_ttl`, `dns_lookup_family`, failure-backoff strategy.
  - Creates `resolve_timer_` on the main-thread dispatcher; its callback is `startResolve()`.
- `startPreInit()` (`logical_dns_cluster.h:76`, impl in `.cc`) kicks the first `startResolve()`.
- `startResolve()` issues `dns_resolver_->resolve(dns_address_, dns_lookup_family_, cb)`. On success:
  - The first address (`current_resolved_address_`) becomes the host's primary address.
  - The full address list (`current_resolved_address_list_`) is used for happy-eyeballs and is compared against the previous result. If either the primary address or the list changed, `logical_host_->setNewAddresses(...)` is called (from `Upstream::LogicalHost`, see `../common/logical_host.*`).
  - If the logical host does not yet exist, it is created once via `LogicalHost::create(...)` and registered into priority 0 via `PriorityStateManager`.
- `onPreInitComplete()` fires after the first successful resolve (or, with `wait_for_warm_on_init_` disabled, immediately).

## Load balancing hooks
- None specific - the factory returns `nullptr` for `ThreadAwareLoadBalancer`, so the top-level `lb_policy` runs against a priority set that always contains a single host. Load balancer host selection is effectively a no-op; the interesting behavior is inside `LogicalHost::createConnection`, which snapshots the current address under its `address_lock_`.

## Key decision points
- Single-host enforcement - `logical_dns_cluster.cc:55-64`.
- Priority-to-zero rewrite for zone-aware routing - `logical_dns_cluster.cc:30-47`.
- Inline address swap via `LogicalHost::setNewAddresses` (no priority set rebuild) keeps connection pools intact across DNS rotations - see `../common/logical_host.cc:51-68`.
- TTL-driven vs. fixed refresh rate based on `respect_dns_ttl_`.
- Custom DNS resolver names are rejected - use `typed_dns_resolver_config` on the factory instead.

## Configuration
- `dns_refresh_rate`, `dns_jitter`, `dns_failure_refresh_rate`, `respect_dns_ttl`, `dns_lookup_family` (all on the `DnsCluster` proto).
- The single `load_assignment` endpoint's `socket_address.address` is treated as the hostname; `socket_address.port_value` is the port attached to every resolved IP.

## Stats
- Inherits `ClusterImplBase` config-update stats. `update_attempt` / `update_success` / `update_no_rebuild` / `update_failure` fire per DNS roundtrip; `update_no_rebuild` in particular is common for this cluster since the logical host usually does not change identity.
