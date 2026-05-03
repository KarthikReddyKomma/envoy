# TLS Certificate Mappers

A *certificate mapper* is a small plug-in that derives a logical certificate name — typically used as a key into an on-demand secret manager — from an in-flight TLS handshake. Mappers are consumed by certificate *selectors* (see `../cert_selectors/`) such as the on-demand selector, which uses the mapper's output to fetch the corresponding SDS secret and resume the handshake.

Two interfaces are defined by `envoy/ssl/handshaker.h`:
- `Ssl::TlsCertificateMapper` — downstream (server) side. Method: `deriveFromClientHello(const SSL_CLIENT_HELLO&) -> std::string`.
- `Ssl::UpstreamTlsCertificateMapper` — upstream (client) side. Method: `deriveFromServerHello(const SSL&, TransportSocketOptionsConstSharedPtr) -> std::string`.

Each mapper registers via `Ssl::TlsCertificateMapperConfigFactory` or `Ssl::UpstreamTlsCertificateMapperConfigFactory`, and is referenced from a certificate-selector config (e.g. `on_demand_secret.Config.certificate_mapper`).

## Subfolders
- `sni/` — downstream mapper that returns the SNI host name from the ClientHello (falling back to a configured default).
- `static_name/` — mapper that always returns a single configured name; implements both the downstream and upstream interfaces.
- `filter_state_override/` — upstream mapper that reads a name from a `Router::StringAccessor` filter-state object on the downstream connection (allowing a filter to dictate which certificate to use upstream).

None of these plug-ins implement the `Network::TransportSocket` interface themselves — they are pure name-derivation strategies consumed by the TLS context.

## Stats / errors
Mappers do not emit stats. Proto validation errors surface at `createTlsCertificateMapperFactory` time.
