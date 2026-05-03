# QUIC Proof Source

Factory for the default QUIC `ProofSource` implementation. The proof source
signs server configs and certificates during the QUIC handshake, sourcing
certs from the listener's filter-chain TLS contexts.

Proto: `envoy.extensions.quic.proof_source.v3.ProofSourceConfig`
(empty configuration message).

## Files
- `envoy_quic_proof_source_factory_impl.h/cc` -
  `EnvoyQuicProofSourceFactoryImpl`. The actual `EnvoyQuicProofSource`
  class lives in `source/common/quic/envoy_quic_proof_source.*`.

## Interface
- Base: `Envoy::Quic::EnvoyQuicProofSourceFactoryInterface`.
- Extension name: `envoy.quic.proof_source.filter_chain`.

## Logic
- `createEmptyConfigProto` returns the empty `ProofSourceConfig`.
- `createQuicProofSource` forwards the listen socket, filter chain
  manager, listener stats, and time source into a
  `std::make_unique<EnvoyQuicProofSource>`. That common-code class is
  responsible for selecting the filter chain based on SNI and handing
  back the matching `CertificateChainWithProof`.

## Key decision points
- Name `filter_chain` reflects the strategy: certs come from the listener
  filter chain manager rather than a standalone config blob, keeping QUIC
  listeners consistent with TLS filter-chain matching.

## Configuration
- Empty; no tunables on the default implementation.

## Stats / errors
- None emitted from the factory. Listener-level errors use
  `Server::ListenerStats` passed into the proof source.
