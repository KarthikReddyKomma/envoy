# Common Cluster Helpers

This directory is a shared library, not a cluster implementation. It hosts helpers reused by the DNS-based clusters (`dns`, `strict_dns`, `logical_dns`) and by `dynamic_forward_proxy`.

## Files
- `logical_host.h` / `logical_host.cc` - `Upstream::LogicalHost` and `Upstream::RealHostDescription`.
- `dns_cluster_backcompat.h` / `dns_cluster_backcompat.cc` - `Upstream::createDnsClusterFromLegacyFields`: converts the legacy DNS fields on `envoy::config::cluster::v3::Cluster` into the new `envoy.extensions.clusters.dns.v3.DnsCluster` message so that `DnsClusterImpl` is the single implementation path.

## `LogicalHost` (`logical_host.h:18`)
- Extends `HostImplBase` + `HostDescriptionImplBase`.
- Models a *single stable host identity* whose backing IP addresses can be swapped out at runtime. Used by `logical_dns` (one logical host per cluster) and by `dynamic_forward_proxy` (one logical host per DNS cache entry).
- `setNewAddresses(address, address_list, lb_endpoint)` (`logical_host.h:37`, `logical_host.cc:51-68`) mutates `address_`, `address_list_or_null_`, and `health_check_address_` under `address_lock_` (an `absl::Mutex`, declared `ABSL_GUARDED_BY(address_lock_)` in the header). This means existing connection pools keep the same host identity across DNS refresh, but new connections go to the freshly resolved IP.
- `createConnection()` (`logical_host.h:40`, `logical_host.cc:80+`) snapshots the current address under the lock, then constructs a connection. When the transport socket matcher uses filter state and filter-state objects are present, it re-resolves the transport socket per connection (see the `needs_per_connection_resolution` branch, `logical_host.cc:97-100`).
- `override_transport_socket_options_` (`logical_host.h:52`) allows a user of `LogicalHost` to pin a specific TLS/ALPN configuration regardless of what the caller of `createConnection()` requests.

## `RealHostDescription` (`logical_host.h:66`)
- Wraps a `LogicalHost` plus a *snapped* address. All description getters forward to the logical host (`canary`, `metadata`, `stats`, `healthChecker`, etc.) but `address()` returns the address captured at wrap time.
- All mutator overrides are no-ops (`logical_host.h:109-115`) because the underlying `LogicalHost` is shared and must not mutate through this view.
- Used when callbacks need to report the actual remote endpoint (e.g. outlier detection) while still treating the host as a single logical entity.

## `createDnsClusterFromLegacyFields` (`dns_cluster_backcompat.cc:10`)
- Shim for pre-existing `Cluster` configs that used top-level `dns_refresh_rate`, `dns_failure_refresh_rate`, `respect_dns_ttl`, `dns_jitter`, `dns_lookup_family`. Copies those into a new `DnsCluster` proto (`dns_cluster_backcompat.cc:16-55`).
- Explicitly does NOT copy `typed_dns_resolver_config` - that is handled at the factory level because resolver selection is factory-specific (see doc comment at `dns_cluster_backcompat.h:9-13`).
- `PANIC_ON_PROTO_ENUM_SENTINEL_VALUES` (`dns_cluster_backcompat.cc:39`) guards the `DnsLookupFamily` switch against future proto enum additions.

## Who uses what
- `logical_host.*` - `logical_dns/logical_dns_cluster.cc`, `dns/dns_cluster.cc` (logical path), `dynamic_forward_proxy/cluster.cc`.
- `dns_cluster_backcompat.*` - `dns/dns_cluster.cc` (`LegacyDnsClusterFactory`), `logical_dns/logical_dns_cluster.cc`, `strict_dns/strict_dns_cluster.cc`.

## Cluster type
N/A - there is no cluster class here. The helpers plug into clusters whose base class is `BaseDynamicClusterImpl` (strict_dns, dynamic_forward_proxy, dns) or `ClusterImplBase` (logical_dns).
