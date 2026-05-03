# QUIC Crypto Streams

Factories for QUIC client- and server-side crypto streams. These are thin
shims that delegate to QUICHE's default implementations but exist as
extensions so downstream consumers (e.g. custom TLS integrations) can swap
them out.

Proto (server side):
`envoy.extensions.quic.crypto_stream.v3.CryptoServerStreamConfig`
(empty message).

## Files
- `envoy_quic_crypto_client_stream.h/cc` -
  `EnvoyQuicCryptoClientStreamFactoryImpl`.
- `envoy_quic_crypto_server_stream.h/cc` -
  `EnvoyQuicCryptoServerStreamFactoryImpl` and its registration.

## Interface
- Client factory base:
  `Envoy::Quic::EnvoyQuicCryptoClientStreamFactoryInterface`.
- Server factory base:
  `Envoy::Quic::EnvoyQuicCryptoServerStreamFactoryInterface`.
- Server extension name: `envoy.quic.crypto_stream.server.quiche`.
- The client factory is constructed in-process (no registration in the
  file), while the server factory is registered via `REGISTER_FACTORY`.

## Logic
- Client: `createEnvoyQuicCryptoClientStream` returns
  `quic::QuicCryptoClientStream` (QUICHE's default) with `true` for the
  `has_application_state` parameter.
- Server: `createEnvoyQuicCryptoServerStream` calls
  `quic::CreateCryptoServerStream`. The
  `transport_socket_factory` and `dispatcher` parameters are accepted but
  unused locally; the header comment explicitly notes they must remain
  in the signature because downstream forks may rely on them.

## Key decision points
- `envoy_quic_crypto_server_stream.cc:13` - the parameters are held
  deliberately unused to preserve the ABI for downstream crypto stream
  implementations.
- Client factory is not registered because Envoy's QUIC upstream is
  wired to the default implementation; registered extensions only
  override when explicitly swapped in.

## Configuration
- Server proto is empty; there are no tunables on the default
  implementation.

## Stats / errors
- None emitted from this extension; errors come from QUICHE.
