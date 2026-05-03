# Transport Socket Common Helpers

Shared infrastructure used by transport socket extensions that wrap an inner transport. This folder does not register any extension itself; it provides base classes that other transport sockets (proxy_protocol, http_11_proxy, tap, tcp_stats, internal_upstream) derive from.

No proto — this is library code.

## Files
- `passthrough.h/cc` — defines three reusable base classes:
  - `PassthroughSocket` — a `Network::TransportSocket` that forwards every interface method to a wrapped `transport_socket_`.
  - `PassthroughFactory` — a `Network::CommonUpstreamTransportSocketFactory` that delegates `implementsSecureTransport`, `supportsAlpn`, `defaultServerNameIndication`, `hashKey`, `sslCtx`, `clientContextConfig`, and (when QUIC is enabled) `getCryptoConfig` to an inner upstream factory.
  - `DownstreamPassthroughFactory` — the downstream equivalent, exposing only `implementsSecureTransport`.

## Transport socket role
`PassthroughSocket` implements `Network::TransportSocket` by forwarding each call:
- `setTransportSocketCallbacks` (`passthrough.cc:15`) — forwards callbacks to the inner socket.
- `doRead` / `doWrite` (`passthrough.cc:35`, `passthrough.cc:39`) — forward I/O verbatim to the inner socket.
- `onConnected`, `closeSocket`, `canFlushClose`, `connect`, `protocol`, `failureReason`, `ssl`, `configureInitialCongestionWindow` — all pass through.
- `startSecureTransport` is hard-coded to return `false` (`passthrough.h:81`); subclasses that need StartTLS must override.

Subclasses typically override one or two methods to inject their behavior (e.g. prepending bytes on connect, mirroring traffic) while inheriting the rest.

## Lifecycle
Entirely defined by the wrapped inner socket. `PassthroughSocket` adds no state of its own beyond the `transport_socket_` unique_ptr.

## Key decision points
- `passthrough.h:14-47` — `PassthroughFactory` aggregates an `UpstreamTransportSocketFactoryPtr` and forwards factory-level capability queries. Subclasses override `createTransportSocket` to wrap the produced inner socket.
- `passthrough.h:49-64` — `DownstreamPassthroughFactory` is the simpler downstream variant (no ALPN / SNI / ssl context query surface).
- `passthrough.h:66-87` — `PassthroughSocket` owns the inner socket in `transport_socket_` and provides the default delegation. `ASSERT(transport_socket_factory_ != nullptr)` in both factory constructors prevents accidental null wrapping.

## Configuration
None — helpers only.

## Stats / errors
None directly. Wrapping transports are responsible for their own stats.
