# Strict DNS Cluster (`envoy.cluster.strict_dns`)

Periodically resolves each configured hostname and maintains *one host per returned IP*. If the resolver returns three addresses, the cluster exposes three hosts; if a later query drops one address, that host is removed and its connection pool drained. Unlike `logical_dns`, connection pools are not preserved across IP changes.

Proto: `envoy.extensions.clusters.dns.v3.DnsCluster` (shared with the `dns` / `logical_dns` factories; see `api/envoy/extensions/clusters/dns/v3/dns_cluster.proto`).

This class is the *legacy* implementation; when runtime flag `envoy.reloadable_features.enable_new_dns_implementation` is on, the `envoy.cluster.strict_dns` factory name is serviced by the unified `DnsClusterImpl` instead (see `../dns/README.md`).

## Files
- `strict_dns_cluster.h` / `strict_dns_cluster.cc` - `StrictDnsClusterImpl` and its nested `ResolveTarget`.

## Cluster type
- `StrictDnsClusterImpl` extends `Upstream::BaseDynamicClusterImpl` (`strict_dns_cluster.h:18`) because the host set is built asynchronously and mutates on every refresh.
- `initializePhase()` returns `Primary` (`strict_dns_cluster.h:21`).
- No dedicated factory class in this file - it is instantiated by `LegacyDnsClusterFactory` (`envoy.cluster.strict_dns`) defined in `../dns/dns_cluster.cc`, which first calls `createDnsClusterFromLegacyFields(...)` (see `../common/dns_cluster_backcompat.cc`) to translate the old inline fields into a `DnsCluster` proto.

## Initialization / host set
- `create(...)` (`strict_dns_cluster.cc:18-29`) wraps the ctor and funnels status errors.
- Ctor (`strict_dns_cluster.cc:31-71`):
  - Snapshots the `load_assignment_` and `LocalInfo`.
  - Pulls DNS knobs from the `DnsCluster` proto: `dns_refresh_rate` (default 5000ms), `dns_jitter` (default 0), `respect_dns_ttl`, `dns_lookup_family`.
  - Builds the `failure_backoff_strategy_` via `Config::Utility::prepareDnsRefreshStrategy`.
  - Validates endpoints via `validateEndpoints(locality_lb_endpoints, {})` (`strict_dns_cluster.cc:53`).
  - Rejects per-endpoint `resolver_name` (`strict_dns_cluster.cc:56-59`).
  - Creates one `ResolveTarget` per `LbEndpoint`, each with its own DNS query, refresh timer, and host vector (`strict_dns_cluster.cc:61-64`).
  - Reads `overprovisioning_factor` and `weighted_priority_health` from `load_assignment_.policy()` (`strict_dns_cluster.cc:68-70`).
- `startPreInit()` (`strict_dns_cluster.cc:73-82`) kicks every target's `startResolve()`. If there are no targets (or `wait_for_warm_on_init_` is disabled) the cluster completes immediately.
- `ResolveTarget::startResolve()` (`strict_dns_cluster.cc:128-236`):
  - Increments `update_attempt_`.
  - On success (`Completed` status): builds `new_hosts` by materializing one `HostImpl` per unique returned address (deduped via `all_new_hosts`) - `strict_dns_cluster.cc:141-174`. `ttl_refresh_rate` is the minimum of the returned TTLs.
  - Runs `parent_.updateDynamicHostList(new_hosts, hosts_, hosts_added, hosts_removed, all_hosts_, all_new_hosts)` (`strict_dns_cluster.cc:178-179`). If this returns true, the diff is applied to the target's `all_hosts_` map and pushed into the priority set via `parent_.updateAllHosts(...)`; otherwise `update_no_rebuild_` is incremented.
  - Resets the failure backoff on success; on failure, the next timer interval comes from `failure_backoff_strategy_->nextBackOffMs()`.
  - With `respect_dns_ttl_`, the refresh interval follows the response TTL (clamped > 0); with `dns_jitter_ms_`, a pseudo-random jitter of `random_.random() % dns_jitter_ms_.count()` is added.
  - Calls `parent_.onPreInitComplete()` so the first successful (or failed) resolution is enough to complete cluster warming - even if other targets are still pending.
- `updateAllHosts(added, removed, current_priority)` (`strict_dns_cluster.cc:84-108`): rebuilds the target priority via `PriorityStateManager::initializePriorityFor` + `registerHostForPriority` for each current host that belongs to that priority, then `updateClusterPrioritySet(...)` with `weighted_priority_health_` and `overprovisioning_factor_`.

## Load balancing hooks
- None custom. Top-level `Cluster::lb_policy` runs against the strict priority set.

## Key decision points
- Deduplication by `address->asString()` within a single DNS response - `strict_dns_cluster.cc:155-157`.
- Per-priority ASSERT after diff to catch host-priority drift - `strict_dns_cluster.cc:181-183`.
- Completion fires on the *first* target to finish, so multi-hostname clusters do not block init on the slowest DNS - `strict_dns_cluster.cc:229-233`.
- `active_query_->cancel(QueryAbandoned)` in `~ResolveTarget` prevents callbacks from firing after cluster destruction - `strict_dns_cluster.cc:122-126`.

## Configuration
- DNS knobs: `dns_refresh_rate`, `dns_jitter`, `dns_failure_refresh_rate`, `respect_dns_ttl`, `dns_lookup_family`.
- `load_assignment.endpoints[*].lb_endpoints[*].endpoint.address.socket_address.{address, port_value}`: `address` is treated as a hostname; `port_value` is applied to every resolved IP.

## Stats
- Inherits `ClusterImplBase` config-update stats: `update_attempt`, `update_success`, `update_no_rebuild`, `update_failure` (incremented at `strict_dns_cluster.cc:130`, `:142`, `:195`, `:221`).
