# QUIC-LB Connection ID Generator

Implements
[draft-ietf-quic-load-balancers](https://datatracker.ietf.org/doc/draft-ietf-quic-load-balancers/)
style connection id generation. Produces CIDs that encode a `server_id` and
an encrypted nonce so an off-path load balancer can route packets to the
correct Envoy instance without parsing TLS. Adds a trailing worker-routing
byte so inside a single Envoy, packets land on the correct worker thread.

Proto: `envoy.extensions.quic.connection_id_generator.quic_lb.v3.Config`.

## Files
- `quic_lb.h/cc` - `QuicLbConnectionIdGenerator` (per-worker generator),
  `ThreadLocalData` (owns a QUICHE `LoadBalancerEncoder`), `Factory`
  (worker factory + SDS wiring), and BPF filter construction.
- `config.h/cc` - `ConfigFactory` registered as
  `envoy.quic.connection_id_generator.quic_lb`.
- `compile_bpf.sh` / `route.bpf` - source for the compiled SO_REUSEPORT
  CBPF filter used for in-kernel worker steering.

## Interface
- Generator base: `quic::ConnectionIdGeneratorInterface`.
- Factory base: `Envoy::Quic::EnvoyQuicConnectionIdGeneratorFactory`.
- Config factory base:
  `Quic::EnvoyQuicConnectionIdGeneratorConfigFactory`.

## Logic
- `Factory::create` reads `server_id` from the configured `DataSource`
  (optionally base64-decoded), validates `expected_server_id_length`, and
  checks the combined `server_id` + `nonce_length_bytes` fits in
  `quic::kQuicMaxConnectionIdWithLengthPrefixLength - 2` (= 18 bytes, for
  the length prefix byte and the worker-routing byte).
- A test `ThreadLocalData` instance is built with a placeholder key to
  surface misconfigurations at load time before SDS delivers the real
  encryption key.
- Key/version come from a generic SDS secret with entries
  `encryption_key` (must be `kLoadBalancerKeyLen`) and
  `configuration_version` (one byte, `< kNumLoadBalancerConfigs`). The
  factory subscribes for updates and calls
  `ThreadLocalData::updateKeyAndVersion` on every worker via
  `runOnAllThreads`.
- The per-generator `appendRoutingId` runs the QUICHE encoder, copies the
  produced CID into a scratch buffer, appends the caller's `worker_id`
  as a trailing byte, and returns a new `QuicConnectionId`.
- `getCompatibleConnectionIdWorkerSelector` returns a userspace version of
  the packet parser used by the BPF program, so workers can route spilled
  or migrated packets consistently with the kernel filter.

## Key decision points
- `quic_lb.cc:200` - concurrency above `UINT8_MAX` is rejected because
  the worker id is encoded in a single trailing byte. A TODO notes this
  could grow if needed.
- `quic_lb.cc:233` - packs `server_id`, `nonce`, length prefix and the
  worker-routing byte into the 20-byte max CID space; exceeds that and
  the factory errors out at load time.
- `quic_lb.cc:317` - the CBPF program is kept verbatim as a member
  `filter_` so the `sock_fprog` pointer stays valid for the lifetime of
  the socket option.
- `quic_lb.cc:343` `bpfEquivalentFunction` mirrors the BPF logic in C++
  for long-header (by CID-length byte) and short-header (by trailing
  worker byte) packets; default value is returned whenever the packet is
  too short or worker id is out of range.

## Configuration
- `server_id` (`DataSource`), optional `server_id_base64_encoded`.
- `expected_server_id_length` - enforced length check (0 = skip).
- `nonce_length_bytes` - per QUIC-LB spec.
- `unencrypted_mode` - use
  `quic::LoadBalancerConfig::CreateUnencrypted` for testing.
- `encryption_parameters` - generic SDS secret with `encryption_key` and
  `configuration_version` entries.

## Stats / errors
- Configuration errors surface as `absl::InvalidArgumentError` at
  extension load time. An `ENVOY_BUG` fires on the extremely unlikely
  case where a previously-validated configuration fails to apply on an
  SDS update (`quic_lb.cc:303`).
