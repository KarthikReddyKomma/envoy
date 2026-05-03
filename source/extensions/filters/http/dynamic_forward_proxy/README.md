# Dynamic Forward Proxy Filter (`envoy.filters.http.dynamic_forward_proxy`)

Lets Envoy forward to arbitrary upstream hostnames that are **not** known at
configuration time. Works in tandem with the `dynamic_forward_proxy` cluster
type: the filter resolves the target hostname into the cluster's DNS cache and
hands the request to the cluster.

Proto: `envoy.extensions.filters.http.dynamic_forward_proxy.v3.FilterConfig`.

## Lifecycle

`decodeHeaders()` (`proxy_filter.cc:194–234`):

1. Resolve the active route and verify its cluster is a
   `dynamic_forward_proxy` cluster.
2. Determine the target host:
   - Per-route `host_rewrite_literal` — static override.
   - Per-route `host_rewrite_header` — use the value of the named request
     header as the host.
   - Otherwise, the request's `:authority`.
3. Determine the default port from the cluster's transport socket (443 for
   secure, 80 otherwise).
4. Call `loadDynamicCluster()` (`proxy_filter.cc:376+`). This starts an async
   DNS resolution against the cluster's DNS cache and, if needed, a cluster
   bootstrap.
5. Filter returns `StopIteration`; callbacks resume decoding when the host is
   ready.
6. Optionally stores the resolved upstream address in `FilterState` when
   `save_upstream_address=true`.

## Configuration

- `dns_cache_config` — shared DNS cache; controls resolver, TTL, eviction,
  host count.
- `sub_cluster_config` — enable on-demand cluster creation per host.
- `save_upstream_address` — record resolved address for logging / other
  filters.
- Per-route `PerRouteConfig`:
  - `host_rewrite` — literal.
  - `host_rewrite_header` — pick host from header.

## Stats

DNS cache stats (`dns_query_attempt`, `dns_query_success`, `dns_query_failure`,
`host_added`, `host_removed`, …) plus cluster init stats.

## Files

- `proxy_filter.{h,cc}` — filter, async DNS + cluster load.
- `config.{h,cc}` — factory, DNS cache manager wiring.
