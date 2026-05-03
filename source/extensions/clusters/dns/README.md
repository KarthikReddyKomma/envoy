# DNS Cluster (`envoy.cluster.dns`)

Unified DNS-based cluster implementation that can produce either *strict* (one host per resolved IP) or *logical* (a single `LogicalHost` with swappable IPs) host sets. It is the new single code path that replaces the historical `envoy.cluster.strict_dns` and `envoy.cluster.logical_dns` factories; those legacy names are preserved as thin `LegacyDnsClusterFactory` subclasses that convert old fields into a `DnsCluster` proto and then dispatch to the same core implementation.

Proto: `envoy.extensions.clusters.dns.v3.DnsCluster` (see `api/envoy/extensions/clusters/dns/v3/dns_cluster.proto`).

## Files
- `dns_cluster.h` / `dns_cluster.cc` - `DnsClusterImpl`, `DnsClusterFactory`, plus in-file `LegacyDnsClusterFactory`, `LogicalDNSFactory` (`envoy.cluster.logical_dns`), `StrictDNSFactory` (`envoy.cluster.strict_dns`).

## Cluster type
- `DnsClusterImpl` extends `Upstream::BaseDynamicClusterImpl` (`dns_cluster.h:34`) because hosts are not known at config time; they arrive asynchronously from DNS.
- `initializePhase()` returns `Primary` (`dns_cluster.h:37`) - DNS lookups start immediately and do not depend on another cluster.
- Three registered factory names:
  - `envoy.cluster.dns` (`DnsClusterFactory`, `dns_cluster.cc:49`) - new, typed, takes `DnsCluster` proto.
  - `envoy.cluster.strict_dns` (`StrictDNSFactory`, `dns_cluster.cc:109`).
  - `envoy.cluster.logical_dns` (`LogicalDNSFactory`, `dns_cluster.cc:98`).

## Initialization / host set
- `DnsClusterImpl` ctor (`dns_cluster.cc:149-213`):
  - `extractAndProcessLoadAssignment` (`dns_cluster.cc:111-131`): when `all_addresses_in_single_endpoint` is set, forces every locality priority to 0 (logical DNS only supports a single host and zone-aware routing requires priority 0).
  - In logical mode, validates that there is exactly one `LocalityLbEndpoints` with exactly one `LbEndpoint` (`dns_cluster.cc:172-184`).
  - In strict mode, validates endpoints via `validateEndpoints(...)` (`dns_cluster.cc:189`).
  - Rejects custom `resolver_name` on any endpoint (`dns_cluster.cc:195-201`).
  - Builds one `ResolveTarget` per `LbEndpoint` (`dns_cluster.cc:203-207`), each holding its own `active_query_`, `resolve_timer_`, and cached host list.
- `startPreInit()` (`dns_cluster.cc:215-224`) kicks each `ResolveTarget::startResolve()`; if there are no targets it completes immediately via `onPreInitComplete()`.
- `ResolveTarget::startResolve()` (`dns_cluster.cc:381+`) calls `dns_resolver_->resolve(...)`. On completion:
  - Strict mode: `createStrictDnsHosts()` (`dns_cluster.cc:301-335`) produces one `HostImpl` per unique resolved address; `updateStrictDnsHosts()` (`dns_cluster.cc:357-379`) calls `updateDynamicHostList(...)` and, on change, `parent_.updateAllHosts(...)`.
  - Logical mode: `createLogicalDnsHosts()` (`dns_cluster.cc:281-299`) builds one `Upstream::LogicalHost` per query; `updateLogicalDnsHosts()` (`dns_cluster.cc:337-355`) compares the cached address + address list against the new primary address and address list, and only rebuilds the host vector on change.
  - `respect_dns_ttl_` makes `final_refresh_rate` follow the DNS TTL; otherwise `dns_refresh_rate_ms_` (default 5000ms) is used. `failure_backoff_strategy_` handles failed resolutions.

## Load balancing hooks
- The factory returns `nullptr` as the `ThreadAwareLoadBalancer` (`dns_cluster.cc:46`, `dns_cluster.cc:82`), which means the cluster manager uses the *default* LB policy configured on the top-level `Cluster` (ROUND_ROBIN, LEAST_REQUEST, etc.) against the dynamically refreshed `PrioritySet`.

## Key decision points
- Runtime feature gate `envoy.reloadable_features.enable_new_dns_implementation` picks between `DnsClusterImpl` and the older `LogicalDnsCluster` / `StrictDnsClusterImpl` - `dns_cluster.cc:34-43` (typed path) and `dns_cluster.cc:70-79` (legacy path).
- Logical vs strict branch inside `DnsClusterImpl` is driven by `all_addresses_in_single_endpoint_` - `dns_cluster.cc:172`, `dns_cluster.cc:272-278`, `dns_cluster.cc:397-403`.
- Empty response handling differs per mode - `isSuccessfulResponse()` at `dns_cluster.cc:269-279`.
- Legacy-field conversion happens in `createDnsClusterFromLegacyFields` (`../common/dns_cluster_backcompat.cc`), called at `dns_cluster.cc:64`.

## Configuration
- `dns_refresh_rate` (default 5s), `dns_jitter`, `dns_failure_refresh_rate`, `respect_dns_ttl`, `dns_lookup_family`, `typed_dns_resolver_config`.
- `all_addresses_in_single_endpoint` (bool): selects logical vs strict semantics.

## Stats
- Inherits `ClusterImplBase` stats plus the standard `configUpdateStats()` counters: `update_attempt`, `update_success`, `update_no_rebuild`, `update_failure` (incremented at `dns_cluster.cc:353`, `dns_cluster.cc:377`, `dns_cluster.cc:383`, `dns_cluster.cc:395`).
