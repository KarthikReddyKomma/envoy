# Default QUIC Client Packet Writer

Registers the default factory that produces the QUIC client packet writer
Envoy uses when no custom writer is configured. The packet writer is
responsible for emitting UDP datagrams out of a QUIC client connection.

Proto: `envoy.extensions.quic.client_writer_factory.v3.DefaultClientWriter`
(empty configuration message).

## Files
- `default_quic_client_packet_writer_factory_config.h/cc` -
  `DefaultQuicClientPacketWriterFactoryConfig` registration class.

## Interface
- Base: `Quic::QuicClientPacketWriterConfigFactory`.
- Extension name: `envoy.quic.packet_writer.default`.

## Logic
- `createEmptyConfigProto` returns a default `DefaultClientWriter`
  message.
- `createQuicClientPacketWriterFactory` ignores its inputs and returns a
  `std::make_unique<QuicClientPacketWriterFactoryImpl>` (from
  `source/common/quic/`), which wraps QUICHE's default writer.
- `REGISTER_FACTORY` plugs the class into
  `QuicClientPacketWriterConfigFactory` in `.cc`.

## Key decision points
- `default_quic_client_packet_writer_factory_config.h:25` - this
  extension exists to provide a registered default so the QUIC upstream
  stack can always find a writer factory even when config omits the
  field.

## Configuration
- Empty; there are no knobs.

## Stats / errors
- None.
