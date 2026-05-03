# Basic QUIC Connection Debug Visitor

QUIC connection debug visitor that logs connection-close events at `info`
level. Intended as a lightweight default for troubleshooting QUIC sessions
without wiring up detailed stats.

Proto: `envoy.extensions.quic.connection_debug_visitor.v3.BasicConfig`
(empty configuration message).

## Files
- `envoy_quic_connection_debug_visitor_basic.h/cc` -
  `EnvoyQuicConnectionDebugVisitorBasic` (the visitor), its per-listener
  factory `EnvoyQuicConnectionDebugVisitorFactoryBasic`, and the outer
  config-factory `EnvoyQuicConnectionDebugVisitorFactoryFactoryBasic`.

## Interface
- Visitor base: `quic::QuicConnectionDebugVisitor`.
- Registered factory base:
  `Envoy::Quic::EnvoyQuicConnectionDebugVisitorFactoryFactoryInterface`.
- Extension name: `envoy.quic.connection_debug_visitor.basic`.

## Logic
- The listener factory's `createFactory` is a no-op transform
  (`std::make_unique<EnvoyQuicConnectionDebugVisitorFactoryBasic>`).
- `createQuicConnectionDebugVisitor` constructs a visitor per session,
  holding references to the `QuicSession` and `StreamInfo::StreamInfo`.
- `OnConnectionClosed` emits a structured log line containing peer
  address, connection id, `ConnectionCloseSource`, and error details from
  the close frame.

## Key decision points
- `envoy_quic_connection_debug_visitor_basic.h:28` - `stream_info_` is
  captured by reference and explicitly `(void)`-cast to appease GCC's
  `[[maybe_unused]]` handling for class members.
- Log level is `info` so close diagnostics reach production logs without
  debug flags; keep in mind one line per closed connection.

## Configuration
- Empty; no tunables.

## Stats / errors
- None. Diagnostic output is log-only.
