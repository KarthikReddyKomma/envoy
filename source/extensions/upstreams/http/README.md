# HTTP Upstream Protocol Options

Shared configuration and connection-pool wiring for clusters that serve HTTP
traffic from the router side. This directory defines the
`HttpProtocolOptions` extension (the modern way to configure upstream HTTP
protocols on a cluster) and hosts the per-protocol connection-pool
implementations in subdirectories.

Proto: `envoy.extensions.upstreams.http.v3.HttpProtocolOptions`

## Files
- `config.h` / `config.cc` - `ProtocolOptionsConfigImpl` and
  `ProtocolOptionsConfigFactory`, registered as
  `envoy.extensions.upstreams.http.v3.HttpProtocolOptions` in the
  `envoy.upstream_options` category.
- `generic/` - a `GenericConnPool` factory that picks HTTP, TCP, or UDP
  based on the router's upstream-protocol enum.
- `http/` - HTTP connection pool + `HttpUpstream` for router->upstream HTTP
  streams.
- `tcp/` - TCP connection pool + `TcpUpstream` for HTTP CONNECT upstreams.
- `udp/` - UDP datagram "pool" + `UdpUpstream` for HTTP CONNECT-UDP.
- `dynamic_modules/` - TCP-backed bridge that delegates protocol
  transformation to a dynamic module.

## Interface
`ProtocolOptionsConfigFactory` implements
`Server::Configuration::ProtocolOptionsFactory`. The returned
`ProtocolOptionsConfigImpl` implements `Upstream::HttpProtocolOptionsConfig`
and is queried by clusters to produce per-stream pool settings, HTTP/1 /
HTTP/2 / HTTP/3 options, retry/hash/shadow policies, header validator, and
alternate protocols cache config.

## Logic
`createProtocolOptionsConfig` downcasts+validates the proto, then dispatches
to `ProtocolOptionsConfigImpl::createProtocolOptionsConfig` which:
- builds the HTTP/1 and validated HTTP/2 settings,
- optionally constructs a `HeaderValidatorFactory`,
- builds per-cluster `ShadowPolicy` list,
- builds the optional cluster-level `RetryPolicy`,
- builds the cluster-level `HashPolicy` (for LB consistent-hash input).

The two-argument `createProtocolOptionsConfig` (legacy overload) supports
clusters that declare `http_protocol_options` inline in the cluster proto
instead of via this extension.

`parseFeatures` folds cluster/proto options into a
`Upstream::ClusterInfo::Features` bitmask (HTTP2/HTTP3/USE_ALPN/etc) that
the cluster manager consults at pool-creation time.

## Key decision points
- `config.h:33-43` - the two creation entry points (new + legacy).
- `config.h:47-48` - `parseFeatures` for the cluster features bitmask.
- `config.h:131-157` - `ProtocolOptionsConfigFactory` registration.

## Configuration
Users attach this as a typed extension under
`Cluster.typed_extension_protocol_options["envoy.extensions.upstreams.http.v3.HttpProtocolOptions"]`.
Fields cover all three HTTP versions, `upstream_http_protocol_options`,
`common_http_protocol_options`, `http_filters`, `auto_config`,
`alternate_protocols_cache_options`, `header_validation_config`, and the
cluster-scoped retry/shadow/hash policies.

## Stats
None directly; emitted by the owning cluster / pools.
