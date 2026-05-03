# HTTP/1.1 Proxy Transport Socket (`envoy.transport_sockets.http_11_proxy`)

Upstream-only transport that tunnels a TLS (or other) connection through an HTTP/1.1 forward proxy. When the upstream connection is established, this socket first sends an `HTTP/1.1 CONNECT` request to the proxy, waits for a `200` response, and then transparently passes through the inner transport (typically TLS). Used by clients that must reach origin servers through an explicit proxy.

Proto: `envoy.extensions.transport_sockets.http_11_proxy.v3.Http11ProxyUpstreamTransport`.

## Files
- `config.h/cc` — registers `UpstreamHttp11ConnectSocketConfigFactory`. Parses the outer proto, resolves the optional `default_proxy_address`, builds the wrapped inner transport factory (defaults to `raw_buffer` if none specified), and constructs a `UpstreamHttp11ConnectSocketFactory`.
- `connect.h/cc` — defines `UpstreamHttp11ConnectSocket` (the passthrough-based socket that injects the CONNECT handshake), `UpstreamHttp11ConnectSocketFactory`, and the `SelfContainedParser` used to parse the proxy's CONNECT response via the Balsa HTTP/1 parser.

## Transport socket role
Extends `PassthroughSocket` (from `../common/passthrough.h`). Overrides:
- `setTransportSocketCallbacks` (`connect.cc:115`) — captures the callbacks before forwarding.
- `doWrite` (`connect.cc:121`) — on the very first write, emits the buffered `CONNECT` header; while waiting for the `200` response it returns `KeepOpen, 0` so upper layers (e.g. TLS handshake) don't see data yet.
- `doRead` (`connect.cc:133`) — peeks into the receive buffer, feeds it to `SelfContainedParser`, and once a `200` response is parsed, drains exactly `bytes_processed` from the kernel buffer and flips to pure passthrough. On a non-200 response or missing headers after 2000 bytes, returns `Close`.

The factory inherits from `PassthroughFactory` so ALPN, SNI, sslCtx, and hashKey delegate to the inner TLS factory.

## Lifecycle
- Connect path:
  1. `UpstreamHttp11ConnectSocket` constructor resolves the target string, either from `options->http11ProxyInfo()` (preferred), the constructor's `proxy_info` (from the proto), or the host's metadata (`ENVOY_HTTP11_PROXY_TRANSPORT_SOCKET_ADDR`). See `connect.cc:37-69`.
  2. `handleProxyInfoConnect` / `handleHostMetadataConnect` build the CONNECT request into `header_buffer_`. Modern RFC 9110 format includes a `Host:` header; the runtime feature `envoy.reloadable_features.http_11_proxy_connect_legacy_format` falls back to the legacy no-`Host:` format (`connect.cc:82-112`).
  3. `need_to_strip_connect_response_` is set to `true` to cause `doRead` to intercept the upstream's reply.
- Data path: on `doWrite`, bytes from `header_buffer_` go out first; once drained and the 200 response has been consumed by `doRead`, all reads/writes pass through to the inner socket.
- Close path: delegated to the inner socket via `PassthroughSocket`.

## Key decision points
- `connect.cc:20-30` — `isValidConnectResponse` uses `BalsaParser` in response mode to assert headers are complete and status is `200`. It pauses on `onHeadersComplete` to avoid parsing the body.
- `connect.cc:133-178` — `doRead` uses `recv(..., MSG_PEEK)` to read up to 2000 bytes without draining, then `read` to drain only what the parser consumed. Anything beyond the CONNECT response remains for the inner TLS socket.
- `connect.cc:180-204` — `writeHeader` loops over `ioHandle().write(header_buffer_)`, translating `Again` to `KeepOpen` and any other error to `Close`.
- `connect.cc:222-232` — `hashKey` mixes proxy address and hostname into the upstream connection-pool hash so connections to different proxies are not shared.
- `config.cc:36-49` — the inner transport factory can be any registered upstream transport socket config factory; defaults to `envoy.transport_sockets.raw_buffer` if omitted.

## Configuration
- `transport_socket` — inner upstream transport (commonly TLS). Defaults to `raw_buffer`.
- `default_proxy_address` — fallback proxy address used when the transport socket options do not supply `http11ProxyInfo` and host metadata is absent.

Per-connection proxy info may also come from:
- `Network::TransportSocketOptions::Http11ProxyInfo` (highest precedence; `connect.cc:40`).
- Host-level `typed_filter_metadata` under `ENVOY_HTTP11_PROXY_TRANSPORT_SOCKET_ADDR` (`connect.cc:58-68`).

## Stats / errors
No stats exposed by the extension itself. Failures (non-200 CONNECT, oversized headers, write errors) close the connection and are logged at trace level under `Logger::Id::connection`.
