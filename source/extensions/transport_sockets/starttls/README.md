# StartTLS Transport Socket (`envoy.transport_sockets.starttls`)

Transport that begins life as a cleartext `raw_buffer` socket and can be upgraded to TLS mid-connection when a filter decides the handshake should start (e.g. SMTP `STARTTLS`, Postgres/MySQL protocol negotiation). Provided for both downstream (listener) and upstream (cluster) sides.

Proto:
- Downstream: `envoy.extensions.transport_sockets.starttls.v3.StartTlsConfig`.
- Upstream: `envoy.extensions.transport_sockets.starttls.v3.UpstreamStartTlsConfig`.

Both messages wrap a `cleartext_socket_config` and a `tls_socket_config`.

## Files
- `config.h/cc` — registers `DownstreamStartTlsSocketFactory` and `UpstreamStartTlsSocketFactory` under the name `starttls`. Each factory reads the two sub-transport configs and builds the raw and TLS inner factories via `rawSocketConfigFactory()` / `tlsSocketConfigFactory()` (looked up by the well-known names `raw_buffer` and `tls`).
- `starttls_socket.h/cc` — defines `StartTlsSocket` plus `StartTlsSocketFactory` (upstream) and `StartTlsDownstreamSocketFactory`. Also defines the internal `CallbackProxy` that filters `Connected` events so they are only raised once across the socket-swap.

## Transport socket role
`StartTlsSocket` directly implements `Network::TransportSocket` (not via `PassthroughSocket`) because it holds two inner sockets and actively swaps between them:
- `doRead` / `doWrite`, `onConnected`, `closeSocket`, `canFlushClose`, `ssl`, `failureReason`, `configureInitialCongestionWindow` all forward to `active_socket_`.
- `protocol()` returns the string `"starttls"`.
- `startSecureTransport()` (`starttls_socket.cc:9`) is the entry point: if not yet upgraded, it attaches callbacks to the TLS socket, runs its `onConnected()` to kick off the TLS handshake, moves the TLS socket into `active_socket_` (discarding the raw socket), and calls `connectionInfoSetter().setSslConnection(...)` so downstream callers can observe SSL info.

## Lifecycle
- Initial connect: `active_socket_` is the raw buffer socket; data flows cleartext.
- Upgrade: a network filter invokes `callbacks->connection().startSecureTransport()`, which ends up calling `StartTlsSocket::startSecureTransport()`. TLS handshake begins immediately against already-exchanged or pending bytes on the same connection.
- After upgrade: `active_socket_` is the TLS socket; the `using_tls_` flag prevents re-entry. The `CallbackProxy` ensures the `Connected` event is only raised once, suppressing the duplicate that would come from the TLS socket's own handshake completion.
- Close path: closes whichever socket is currently active.

## Key decision points
- `starttls_socket.cc:10-23` — upgrade is idempotent (`if (!using_tls_)`). The TODO notes that this assumes `active_socket_` has no buffered data at swap time (true for `RawBufferSocket`).
- `starttls_socket.h:63-93` — `CallbackProxy::raiseEvent` swallows a second `Connected` event post-upgrade and instead calls `flushWriteBuffer()` on the parent callbacks so pending writes are flushed once TLS is ready.
- `starttls_socket.h:121` — `implementsSecureTransport()` returns `false` on the factory because the socket is *not* initially secure; upstream/downstream code paths that check for TLS look at the connection info instead.
- `config.cc:18-31` — downstream factory passes `server_names` through to the TLS inner factory (needed for SNI filter-chain match); the raw factory also receives it for symmetry.
- `config.cc:41-54` — upstream factory is a straight composition with no per-host variation.

## Configuration
- `cleartext_socket_config` — a `TransportSocket` message, normally `envoy.transport_sockets.raw_buffer`.
- `tls_socket_config` — a `TransportSocket` message, normally `envoy.transport_sockets.tls` with matching up/downstream `TlsContext`.

Triggering the upgrade is the job of a network filter (e.g. the SMTP STARTTLS filter or the Postgres TLS-termination filter), not of this transport socket.

## Stats / errors
None emitted directly. TLS errors surface via the inner TLS socket's `failureReason()`. If `startSecureTransport()` is called without valid TLS config, connection failure propagates from the wrapped TLS socket.
