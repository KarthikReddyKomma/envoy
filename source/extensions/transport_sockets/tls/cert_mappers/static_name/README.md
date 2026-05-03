# Static Name Certificate Mapper (`envoy.tls.certificate_mappers.static_name`)

Certificate mapper that always returns a single fixed name, ignoring any handshake context. Useful when the on-demand certificate selector is configured with a single pre-known secret, or as a test / bootstrap fixture. Registered for both downstream and upstream sides.

Proto: `envoy.extensions.transport_sockets.tls.cert_mappers.static_name.v3.StaticName` (field: `name`).

## Files
- `config.h` — declares two factories sharing the same `StaticNameExtension` name `envoy.tls.certificate_mappers.static_name`:
  - `StaticNameMapperFactory` for downstream (`Ssl::TlsCertificateMapperConfigFactory`).
  - `UpstreamStaticNameMapperFactory` for upstream (`Ssl::UpstreamTlsCertificateMapperConfigFactory`).
- `config.cc` — defines the unified `StaticNameMapper` class that implements both `Ssl::TlsCertificateMapper` and `Ssl::UpstreamTlsCertificateMapper`, plus both factories.

## Interface
- `std::string deriveFromClientHello(const SSL_CLIENT_HELLO&)` — returns `name_` (`config.cc:15`).
- `std::string deriveFromServerHello(const SSL&, const Network::TransportSocketOptionsConstSharedPtr&)` — returns `name_` (`config.cc:16`).

## Implementation
- `config.cc:11-23` — `StaticNameMapper` stores only `name_` and returns it from both hook points.
- `config.cc:26-34` — downstream factory returns a lambda capturing `config.name()` by value.
- `config.cc:36-44` — upstream factory is identical; separate factory types exist because Envoy keeps separate registries for up/downstream mappers.
- `config.cc:46-47` — both factories are registered via `REGISTER_FACTORY`.

## Stats / errors
None.
