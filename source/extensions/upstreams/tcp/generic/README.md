# Generic TCP Upstream Connection-Pool Factory

TCP-proxy-facing factory that chooses between a direct TCP connection pool
and an HTTP-tunneling pool (HTTP/1.1 CONNECT, HTTP/2 CONNECT, or HTTP/3
CONNECT-UDP) based on cluster features and per-route tunneling config.

Proto: `envoy.extensions.upstreams.tcp.generic.v3.GenericConnectionPoolProto`

## Files
- `config.h` - `GenericConnPoolFactory` declaration.
- `config.cc` - tunneling decision logic and a `FilterState` accessor
  factory for disabling tunneling per-stream.

## Interface
- `GenericConnPoolFactory` implements `TcpProxy::GenericConnPoolFactory`,
  registered as `envoy.filters.connection_pools.tcp.generic` in the
  `envoy.upstreams` category.
- Also registers `DisableTunnelingObjectFactory` as a
  `StreamInfo::FilterState::ObjectFactory` keyed by
  `TcpProxy::DisableTunnelingFilterStateKey` so operators can flip
  tunneling off at runtime via filter state.

## Logic
`createGenericConnPool(host, cluster, tunneling_config, ctx,
upstream_callbacks, stream_decoder_callbacks, downstream_info)`:
1. If a tunneling config is present and tunneling is not disabled by the
   downstream filter state (`disableTunnelingByFilterState`), pick the HTTP
   codec type from cluster features:
   - `ClusterInfo::Features::HTTP2` set -> HTTP/2
   - else `HTTP3` set -> HTTP/3
   - else HTTP/1
   and build a `TcpProxy::HttpConnPool` that performs CONNECT tunneling.
2. Otherwise build a plain `TcpProxy::TcpConnPool`.

Every returned pool is `valid()`-gated.

## Key decision points
- `config.cc:22-37` - HTTP codec selection for CONNECT tunneling.
- `config.cc:43-50` - filter-state opt-out check.

## Configuration
The pool factory itself is not configurable; its behavior is controlled by:
- `TcpProxy` tunneling config (presence of
  `TunnelingConfigHelperOptConstRef`).
- Cluster-level `HttpProtocolOptions` (sets the HTTP2/HTTP3 features that
  drive codec selection).
- Per-stream `StreamInfo` filter state under
  `TcpProxy::DisableTunnelingFilterStateKey` (bool).

## Stats
None directly; pool-specific stats are emitted by `TcpConnPool` /
`HttpConnPool` in `source/common/tcp_proxy/upstream.{h,cc}`.
