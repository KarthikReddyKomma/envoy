# Raw Buffer Transport Socket (`envoy.transport_sockets.raw_buffer`)

The raw buffer transport socket is Envoy's simplest transport: it performs plain, unencrypted TCP I/O with no framing or handshake. It is the default transport for both listeners (downstream) and upstream clusters when no `transport_socket` is set, and also serves as the inner socket wrapped by passthrough transports (proxy_protocol, http_11_proxy, tap, tcp_stats, etc.).

Proto: `envoy.extensions.transport_sockets.raw_buffer.v3.RawBuffer` (empty message).

## Files
- `config.h/cc` — registers upstream and downstream config factories under the name `envoy.transport_sockets.raw_buffer`. The actual `Network::TransportSocket` implementation lives in `source/common/network/raw_buffer_socket.{h,cc}` as `Network::RawBufferSocket` / `Network::RawBufferSocketFactory`.

## Transport socket role
- Both factories in `config.cc` delegate to `Network::RawBufferSocketFactory`, which creates `RawBufferSocket` instances. That socket implements the `Network::TransportSocket` interface.
- `doRead` / `doWrite` call into the underlying `IoHandle` directly (raw POSIX-like read/write on the connection's FD) without any encryption or protocol framing.
- `onConnected`, `closeSocket`, `failureReason` are trivial (no handshake, no failure state).
- `ssl()` returns `nullptr` (this is a cleartext transport).

## Lifecycle
- Connect path: no handshake; data can flow as soon as the OS TCP connection is established.
- Data path: read/write is a direct passthrough to the socket FD.
- Close path: simply closes the FD via the `IoHandle`.

## Key decision points
- `config.cc:18` — `UpstreamRawBufferSocketFactory::createTransportSocketFactory` constructs a `Network::RawBufferSocketFactory`.
- `config.cc:25` — downstream counterpart, identical semantics (raw buffer does not vary between upstream/downstream).
- `config.cc:28` — `createEmptyConfigProto` returns the empty `RawBuffer` message; there is nothing to configure.
- `config.cc:32-37` — `LEGACY_REGISTER_FACTORY` registrations for both the upstream and downstream `TransportSocketConfigFactory` registries with the well-known name `raw_buffer`.

## Configuration
The proto has no fields. Selecting this transport is purely a choice of which factory name is used.

## Stats / errors
None emitted by this extension. Raw I/O errors are surfaced through the `Network::IoResult` return of `doRead` / `doWrite` in `RawBufferSocket` itself.
