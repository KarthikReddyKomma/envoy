# Reverse Tunnel Upstream Socket Interface

Acceptor side of Envoy's reverse-tunnel feature. Instead of dialing
upstream hosts, this extension caches inbound TCP connections that were
initiated by downstream peers, then hands them back to the cluster code
when new "upstream" connections are requested. The result: Envoy can
deliver traffic into networks where it cannot open sockets outbound.

Proto:
`envoy.extensions.bootstrap.reverse_tunnel.upstream_socket_interface.v3.UpstreamReverseConnectionSocketInterface`.
Factory (socket interface) name:
`envoy.bootstrap.reverse_tunnel.upstream_socket_interface`.

## Files
- `reverse_tunnel_acceptor.h/cc` - `ReverseTunnelAcceptor`, a
  `Network::SocketInterfaceBase` whose `socket()` returns cached reverse
  connections.
- `reverse_tunnel_acceptor_extension.h/cc` -
  `ReverseTunnelAcceptorExtension` (`SocketInterfaceExtension`) holding
  per-worker state via `UpstreamSocketThreadLocal`, plus optional
  `ReverseTunnelReporter` wiring for lifecycle events.
- `upstream_socket_manager.h/cc` - `UpstreamSocketManager`, the per-worker
  cache of accepted sockets with ping-based health checks, FD->node/cluster
  indexing, rebalancing across workers, and per-node/cluster counters.
- `reverse_connection_io_handle.h/cc` -
  `UpstreamReverseConnectionIOHandle` (derives from `RpingInterceptor`),
  custom `IoHandle` that takes ownership of a cached socket and presents
  it to cluster code as if it were freshly dialed.

## Interface
- Socket interface base: `Network::SocketInterfaceBase` (registered via
  `DECLARE_FACTORY`).
- Bootstrap extension base: `Network::SocketInterfaceExtension` (which
  itself implements `Server::BootstrapExtension`).
- Thread-local object base: `ThreadLocal::ThreadLocalObject`.
- IO handle base: `Network::IoHandle` (through `RpingInterceptor` and
  `Network::IoSocketHandleImpl`).

## Logic
- `ReverseTunnelAcceptor::socket(type, addr, options)` is the hot path:
  it looks up the thread-local `UpstreamSocketManager` via
  `getLocalRegistry()` and pulls a cached `ConnectionSocketPtr` for the
  node encoded in `addr`. The socket is wrapped in an
  `UpstreamReverseConnectionIOHandle` whose `connect()` is a no-op (the
  connection is already established).
- The extension is constructed with
  `stat_prefix`, `ping_failure_threshold` (default 3, clamped to >= 1),
  `enable_detailed_stats`, and `enable_tenant_isolation`. If a
  `reporter_config` is provided the factory loads the named
  `ReverseTunnelReporterFactory` and stores its reporter for
  connection-event hooks.
- `onServerInitialized` creates a `TypedSlot<UpstreamSocketThreadLocal>`
  and, per worker, constructs an `UpstreamSocketManager` with the
  configured miss threshold and tenant isolation flag
  (`reverse_tunnel_acceptor_extension.cc:42`).
- `UpstreamSocketManager`:
  - `addConnectionSocket` registers a socket under `(node_id, cluster_id)`
    keys, starts a per-connection ping send timer with 15% jitter
    (`pingIntervalWithJitterMs`), and sets up a recv timer.
  - `getConnectionSocket` pops the least-recently-added socket for the
    node; if keyed by a cluster id it round-robins through the cluster's
    member nodes via `ClusterInfo.round_robin_index`.
  - `onPingResponse` clears miss counters; `onPingTimeout` increments
    them and calls `markSocketDead` once the threshold is hit.
  - `pickLeastLoadedSocketManager` iterates a static vector of all
    worker socket managers under `socket_manager_lock` to rebalance new
    sockets to the least-loaded worker.
  - Per-worker aggregate gauges (`total_clusters`, `total_nodes`) are
    published via `UpstreamSocketThreadLocal` at construction
    (`reverse_tunnel_acceptor_extension.cc:25`).
- `UpstreamReverseConnectionIOHandle::shutdown` is a no-op when the
  handle owns the socket; otherwise it would close the underlying FD
  that was just handed off.

## Key decision points
- `reverse_tunnel_acceptor_extension.h:101` - ping failure threshold is
  clamped to a minimum of 1 so callers cannot accidentally disable dead
  detection.
- `upstream_socket_manager.h:165` - dual indexing by FD and by node
  avoids linear scans when a socket dies; all three maps
  (`accepted_reverse_connections_`, `fd_to_node_map_`,
  `fd_to_socket_it_map_`) are kept in sync.
- `reverse_connection_io_handle.h:43` - `connect()` returns success
  without contacting the peer because the socket is already connected;
  the cluster code must not re-dial.
- `upstream_socket_manager.h:222` - `socket_managers_` is a global
  list so rebalancing can find peer workers; the lock is held only to
  enumerate managers, not during individual socket moves.

## Configuration
- `stat_prefix` - default `reverse_tunnel_acceptor`.
- `ping_failure_threshold` - consecutive missed pings before declaring
  a socket dead (default 3, min 1).
- `enable_detailed_stats` - emit per-node/per-cluster counters.
- `enable_tenant_isolation` - key cached sockets by tenant as well as
  node/cluster.
- `reporter_config` - optional typed config selecting a registered
  `ReverseTunnelReporterFactory` for connect/disconnect event export.

## Stats / errors
- Per-worker gauges: `<prefix>.<dispatcher>.total_clusters`,
  `<prefix>.<dispatcher>.total_nodes`.
- Detailed counters (enabled via `enable_detailed_stats`) emitted per
  node and per cluster by `UpstreamSocketManager` / the extension's
  update helpers.
- Lifecycle events forwarded to the configured reporter via
  `reportConnection` / `reportDisconnection`.
