# Reverse Tunnel Common

Shared helpers used by both the downstream (initiator) and upstream
(acceptor) sides of the reverse tunnel bootstrap extensions. Handles the
`RPING` keep-alive protocol and the HTTP/TLS handshake conventions for
reverse connections.

## Files
- `reverse_connection_utility.h/cc` - `ReverseConnectionUtility` static
  helpers, reverse-tunnel header names
  (`reverseTunnelNodeIdHeader`, `reverseTunnelClusterIdHeader`,
  `reverseTunnelTenantIdHeader`, `reverseTunnelUpstreamClusterNameHeader`),
  and the lightweight `PingMessageHandler` class used to count ping
  messages.
- `rping_interceptor.h/cc` - `RpingInterceptor`, a virtual subclass of
  `Network::IoSocketHandleImpl` that transparently intercepts reads to
  detect and echo `RPING` keep-alive frames before application data
  arrives.

## Interface
- Internal helpers; not registered as factories.
- `ReverseConnectionUtility`: all methods `static`; `ReverseConnectionUtility()
  = delete` so it behaves like a namespace.
- `RpingInterceptor`: overrides `Network::IoHandle::read`; derived classes
  implement `onPingMessage` to react to ping traffic (for metrics, etc.).

## Logic
- Reserved tokens: `PING_MESSAGE = "RPING"`, `PROXY_MESSAGE = "PROXY"`.
- Tenant-scoped identifiers are joined with `TENANT_SCOPE_DELIMITER = ":"`
  via `buildTenantScopedIdentifier` and split via
  `splitTenantScopedIdentifier` into a `TenantScopedIdentifierView`.
- `isPingMessage` recognizes raw `RPING` framing.
  `extractPingFromHttpData` recognises `RPING` embedded inside the first
  HTTP framing bytes emitted during the handshake period.
- `sendPingResponse` writes an `RPING` reply either onto a
  `Network::Connection` or a raw `Network::IoHandle`, returning an
  `Api::IoCallUint64Result` for the low-level path.
- `applySslQuietClose` - convenience wrapper so reverse tunnels flip the
  TLS socket into quiet-close mode (prevents noisy close_notify errors
  when draining).
- `RpingInterceptor::read` intercepts reads while `ping_echo_active_` is
  true; once the first non-`RPING` application byte is seen the flag is
  cleared and subsequent reads fall through to the base class. This keeps
  HTTP/2 framing intact after the initial handshake.

## Key decision points
- `rping_interceptor.h:20` - `ping_echo_active_` is deliberately
  "sticky-off": once disabled it cannot re-enable, so stray later RPINGs
  will be delivered as application bytes instead of being silently
  swallowed. This avoids ambiguity if a peer sends "RPING"-shaped
  application data.
- `reverse_connection_utility.h:59` - header names are built from
  `Http::Headers::get().prefix()` so they pick up Envoy's configured
  header prefix (e.g. `x-envoy-reverse-tunnel-node-id`).
- `PingMessageHandler` keeps the ping count as a per-instance uint64,
  intentionally without a mutex - it is owned by a single worker and
  accessed from that worker only.

## Configuration
- None directly; consumers pass connection/headers as needed.

## Stats / errors
- None emitted here. Ping counts live in the handler; broader
  reverse-tunnel metrics live in the acceptor / initiator extensions.
