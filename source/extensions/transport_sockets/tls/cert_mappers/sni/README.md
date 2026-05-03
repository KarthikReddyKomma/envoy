# SNI Certificate Mapper (`envoy.tls.certificate_mappers.sni`)

Downstream certificate mapper that returns the Server Name Indication value from the incoming TLS ClientHello. When used with the on-demand certificate selector, this lets Envoy look up a per-SNI certificate on the fly. If the client omits SNI, the configured `default_value` is returned instead.

Proto: `envoy.extensions.transport_sockets.tls.cert_mappers.sni.v3.SNI` (field: `default_value`).

## Files
- `config.h` — declares `SNIMapperFactory` implementing `Ssl::TlsCertificateMapperConfigFactory`. Name: `envoy.tls.certificate_mappers.sni`.
- `config.cc` — defines the `SNIMapper` class (implementing `Ssl::TlsCertificateMapper`) plus the factory's `createTlsCertificateMapperFactory` which captures `default_value` in a closure and returns a lambda that constructs new `SNIMapper` instances.

## Interface
- `std::string deriveFromClientHello(const SSL_CLIENT_HELLO& ssl_client_hello)` (`config.cc:16`).

## Implementation
- `config.cc:16-20` — calls `SSL_get_servername(ssl_client_hello.ssl, TLSEXT_NAMETYPE_host_name)`; wraps the result in `absl::NullSafeStringView`. If empty (no SNI), returns `default_value_`; otherwise returns the SNI string as the certificate name.
- `config.cc:27-36` — `createTlsCertificateMapperFactory` downcasts the proto, then returns a `TlsCertificateMapperFactory` lambda that captures `default_value` by value. Each call builds a fresh `SNIMapper` (mappers are owned per TLS context).

## Stats / errors
None. Proto validation is handled by the generic `downcastAndValidate` path.
