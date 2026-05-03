# Reverse Connection Cluster (`envoy.clusters.reverse_connection`)

A cluster for reverse-tunnel topologies: upstream agents dial *into* this Envoy, and the `envoy.bootstrap.reverse_tunnel.upstream_socket_interface` extension parks those sockets in a per-worker `UpstreamSocketManager`. This cluster synthesizes one `HostImpl` per logical node on demand so that downstream requests can be multiplexed back over the parked socket. The "host address" is a synthetic `UpstreamReverseConnectionAddress` (`127.0.0.1:0`) whose `socketInterface()` returns the reverse-tunnel acceptor, so when Envoy tries to "connect" to it, the tunnel interface hands back an already-open parked socket instead.

Proto: `envoy.extensions.clusters.reverse_connection.v3.ReverseConnectionClusterConfig` (see `api/envoy/extensions/clusters/reverse_connection/v3/reverse_connection.proto`).

## Files
- `reverse_connection.h` / `reverse_connection.cc` - `UpstreamReverseConnectionAddress`, `RevConCluster`, inner `LoadBalancer`, `LoadBalancerFactory`, `ThreadAwareLoadBalancer`, `RevConClusterFactory`.

## Cluster type
- `RevConCluster` extends `Upstream::ClusterImplBase` (`reverse_connection.h:135`).
- `initializePhase()` returns `Primary` (`reverse_connection.h:148`); `startPreInit()` immediately calls `onPreInitComplete()` (`reverse_connection.h:212`).
- The factory *requires* `lb_policy == CLUSTER_PROVIDED` (`reverse_connection.cc:225-230`) because the cluster supplies its own LB.

## Initialization / host set
- Ctor (`reverse_connection.cc:191-217`):
  - Resolves `cleanup_interval` (default 60s) and arms `cleanup_timer_` on the main thread.
  - Compiles the mandatory `host_id_format` substitution string into a `Formatter::FormatterPtr` (`reverse_connection.cc:202-205`). This decides the "node identity key" extracted from request context.
  - Optionally compiles `tenant_id_format` for tenant isolation (`reverse_connection.cc:208-213`).
- Hosts are not pre-populated. They are created lazily inside `checkAndCreateHost(host_id)` (`reverse_connection.cc:100-150`):
  - Ask the upstream `UpstreamSocketManager` to map the host key to an actual node id via `getNodeWithSocket(...)` (`reverse_connection.cc:108`). This is where the cluster intersects with parked reverse-tunnel sockets.
  - Double-checked locking against `host_map_` (`reverse_connection.cc:111-130`).
  - Build a `UpstreamReverseConnectionAddress` with the node id (the address's `socketInterface()` (`reverse_connection.h:88-101`) returns the upstream reverse-tunnel acceptor so that connections go through the parked socket).
  - Wrap it in a standard `HostImpl` at priority 0, weight 1, UNKNOWN health, stored in `host_map_[node_id]`.
- `cleanup()` (`reverse_connection.cc:152-169`) is called by `cleanup_timer_`; it erases hosts whose `used()` flag is false (i.e. no conn-pool container is holding the host), preventing leaks as tenants come and go.

## Load balancing hooks
- `ThreadAwareLoadBalancer` -> `LoadBalancerFactory` -> `LoadBalancer` (`reverse_connection.h:180-200`).
- `LoadBalancer::chooseHost(context)` (`reverse_connection.cc:31-98`):
  1. Reject `context == nullptr` (`reverse_connection.cc:33-36`) and missing downstream headers (`reverse_connection.cc:39-42`).
  2. Run `host_id_formatter_` over `{downstream_headers, stream_info}` to obtain the host identifier; treat `""` and the formatter-default `"-"` as missing (`reverse_connection.cc:45-61`).
  3. If tenant isolation is on, run `tenant_id_formatter_` and join with `ReverseConnectionUtility::buildTenantScopedIdentifier(tenant_id, host_id)` (`reverse_connection.cc:65-94`). Tenant isolation requires `tenant_id_format` to be configured, otherwise selection fails.
  4. Call `parent_->checkAndCreateHost(final_host_id)` which returns the existing or newly-minted host.
- `peekAnotherHost`, `selectExistingConnection`, `lifetimeCallbacks` are all no-ops (`reverse_connection.h:160-173`).

## Key decision points
- Host identifier is *always* formatter-derived. There is no fallback to headers, SNI, or authority.
- The address's `socketInterface()` override (`reverse_connection.h:88-101`) is the magic that makes "connections" to `127.0.0.1:0` actually ride the reverse tunnel. The synthetic sockaddr is chosen to match the catch-all filter chain.
- Fallback to the default socket interface when the reverse-tunnel extension is absent (`reverse_connection.h:98-100`), though in practice the factory requires it.
- `getUpstreamSocketManager()` asserts the bootstrap extension + TLS registry are present; both are validated earlier in startup (`reverse_connection.cc:171-189`).
- `CLUSTER_PROVIDED` LB policy is enforced at factory time (`reverse_connection.cc:225-230`).

## Configuration
- `host_id_format` (mandatory): Envoy formatter string used to compute the host key from downstream headers + stream info.
- `tenant_id_format` (optional): formatter string; required when the upstream socket manager reports tenant isolation enabled.
- `cleanup_interval` (default 60s).

## Stats
- Standard `ClusterImplBase` membership and request stats. The `envoy.bootstrap.reverse_tunnel.upstream_socket_interface` extension publishes its own counters for parked sockets / acceptances.
