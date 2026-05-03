# Generic HTTP Upstream Connection-Pool Factory

Router-facing factory that produces the right `GenericConnPool` for the
requested upstream protocol (HTTP, TCP tunnel, or UDP tunnel). This is the
default plug-in the router uses when no specialized upstream connection-pool
factory is configured.

Proto: `envoy.extensions.upstreams.http.generic.v3.GenericConnectionPoolProto`

## Files
- `config.h` - `GenericGenericConnPoolFactory` declaration.
- `config.cc` - switch over the `UpstreamProtocol` enum and instantiation
  of the per-protocol pool wrapper.

## Interface
Implements `Router::GenericConnPoolFactory`, registered as
`envoy.filters.connection_pools.http.generic` in the `envoy.upstreams`
category.

## Logic
`createGenericConnPool(host, cluster, upstream_protocol, priority,
downstream_protocol, ctx, config)`:
- `UpstreamProtocol::HTTP` -> `Upstreams::Http::Http::HttpConnPool` (uses
  the thread-local cluster's HTTP pool).
- `UpstreamProtocol::TCP` -> `Upstreams::Http::Tcp::TcpConnPool` (HTTP
  CONNECT tunnel over raw TCP).
- `UpstreamProtocol::UDP` -> `Upstreams::Http::Udp::UdpConnPool` (CONNECT-
  UDP datagram path).

Each returned pool reports `valid()`; the factory discards it if
construction failed.

## Key decision points
- `config.cc:21-33` - protocol switch selecting the pool implementation.
- `config.cc:25-31` - `valid()` check on every branch.

## Configuration
The proto `GenericConnectionPoolProto` is empty today; the factory exists
mainly so routes / TCP-proxy filters can name a connection-pool
implementation via a typed config extension.

## Stats
None here; each pool reports through its own cluster stats.
