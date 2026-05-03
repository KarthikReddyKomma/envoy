# Deterministic QUIC Connection ID Generator

Registers the default deterministic connection-id generator factory used by
Envoy when no other generator is configured. The actual generator logic is
shared out of `source/common/quic/envoy_deterministic_connection_id_generator`.

Proto:
`envoy.extensions.quic.connection_id_generator.v3.DeterministicConnectionIdGeneratorConfig`
(empty configuration message).

## Files
- `envoy_deterministic_connection_id_generator_config.h/cc` -
  `EnvoyDeterministicConnectionIdGeneratorConfigFactory` registration
  class.

## Interface
- Base: `Quic::EnvoyQuicConnectionIdGeneratorConfigFactory`.
- Extension name:
  `envoy.quic.deterministic_connection_id_generator`.

## Logic
- `createEmptyConfigProto` returns the empty proto.
- `createQuicConnectionIdGeneratorFactory` ignores its inputs and returns
  a `std::make_unique<EnvoyDeterministicConnectionIdGeneratorFactory>`
  from `source/common/quic/`. That factory produces connection ids derived
  from the hash of incoming packets, which lets all workers compute the
  same id for a given packet without coordination.

## Key decision points
- `envoy_deterministic_connection_id_generator_config.cc:19` - the factory
  is a thin registration layer: all logic lives in common QUIC code.
  Having a registered factory here keeps the dynamic extension system
  happy while the default implementation stays inline with QUICHE.

## Configuration
- Empty proto; no tunables.

## Stats / errors
- None.
