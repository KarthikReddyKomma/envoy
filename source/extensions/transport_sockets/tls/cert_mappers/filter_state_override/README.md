# Filter State Override Certificate Mapper (`envoy.tls.upstream_certificate_mappers.filter_state_override`)

Upstream-only certificate mapper that lets a downstream network or HTTP filter dictate which client certificate the upstream connection should present, by writing a string into the shared filter state under the key `envoy.tls.certificate_mappers.on_demand_secret`. If no such filter-state entry is found, a configured `default_value` is returned.

Proto: `envoy.extensions.transport_sockets.tls.cert_mappers.filter_state_override.v3.Config` (field: `default_value`).

## Files
- `config.h` — declares `MapperFactory` implementing `Ssl::UpstreamTlsCertificateMapperConfigFactory`. Name: `envoy.tls.upstream_certificate_mappers.filter_state_override`.
- `config.cc` — defines the `Mapper` class implementing `Ssl::UpstreamTlsCertificateMapper::deriveFromServerHello` and the factory's `createTlsCertificateMapperFactory`.

## Interface
- `std::string deriveFromServerHello(const SSL&, const Network::TransportSocketOptionsConstSharedPtr& options)` (`config.cc:18`).

## Implementation
- `config.cc:20-32` — iterates `options->downstreamSharedFilterStateObjects()` (these are the filter-state objects shared from the originating downstream connection via the cluster-side options propagation). For each, compares the well-known name `envoy.tls.certificate_mappers.on_demand_secret`. If found and castable to `Router::StringAccessor`, returns its `asString()`. Otherwise, returns `default_value_`.
- `config.cc:39-48` — factory captures `default_value` in a closure and returns a lambda that constructs `Mapper` instances.

The typical producer is any filter that writes a `Router::StringAccessorImpl` to the stream filter state with the matching name before the upstream connection is established.

## Stats / errors
None emitted. Misconfiguration (proto) surfaces at factory-creation time through the standard `downcastAndValidate` path.
