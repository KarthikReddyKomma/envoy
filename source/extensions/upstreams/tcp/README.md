# TCP Upstream Protocol Options

Defines `TcpProtocolOptions`, the typed-extension knob for cluster-scoped
TCP upstream settings. Today it exposes a single field - `idle_timeout` -
used by TCP connection pools.

Proto: `envoy.extensions.upstreams.tcp.v3.TcpProtocolOptions`

## Files
- `config.h` - `ProtocolOptionsConfigImpl` and
  `ProtocolOptionsConfigFactory`, registered under
  `envoy.extensions.upstreams.tcp.v3.TcpProtocolOptions` in the
  `envoy.upstream_options` category.
- `config.cc` - populates `idle_timeout_` from the proto.
- `generic/` - `GenericConnPoolFactory` for the TCP proxy filter, chooses
  between a raw TCP pool and an HTTP-tunnel pool.

## Interface
`ProtocolOptionsConfigFactory` implements
`Server::Configuration::ProtocolOptionsFactory`;
`ProtocolOptionsConfigImpl` implements `Upstream::ProtocolOptionsConfig`.

## Logic
`createProtocolOptionsConfig` downcasts and validates the proto, then builds
a shared `ProtocolOptionsConfigImpl` that clusters hand out to the TCP
connection pool via `ThreadLocalCluster::tcpConnPool`. `idleTimeout()`
returns an `absl::optional<std::chrono::milliseconds>` that the pool
consults on each new connection.

## Key decision points
- `config.h:25-34` - class definition; single observable field.
- `config.h:38-45` - factory `createProtocolOptionsConfig` (validates +
  constructs).

## Configuration
Attached as
`Cluster.typed_extension_protocol_options["envoy.extensions.upstreams.tcp.v3.TcpProtocolOptions"]`.
The only field today is `idle_timeout`.

## Stats
None directly.
