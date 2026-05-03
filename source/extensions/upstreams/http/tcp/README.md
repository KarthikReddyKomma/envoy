# HTTP-over-TCP (CONNECT) Upstream

Router upstream implementation for HTTP CONNECT tunnels where the upstream
is a raw TCP connection. Produces a 200 OK synthetic response downstream on
connect, optionally prepends a PROXY-protocol header, and pipes body data in
both directions.

Proto: `envoy.extensions.upstreams.http.generic.v3.GenericConnectionPoolProto`

## Files
- `upstream_request.h` - `TcpConnPool`, `TcpUpstream` declarations.
- `upstream_request.cc` - pool wiring, request/response forwarding, and
  half-close handling.
- `config.h` / `config.cc` - module-local config glue.

## Interface
- `TcpConnPool` implements `Router::GenericConnPool` and
  `Envoy::Tcp::ConnectionPool::Callbacks`.
- `TcpUpstream` implements `Router::GenericUpstream` and
  `Envoy::Tcp::ConnectionPool::UpstreamCallbacks`.

## Logic
1. Construction (`upstream_request.h:24`): fetch the thread-local cluster's
   TCP pool via `thread_local_cluster.tcpConnPool(host, priority, ctx)`.
2. `newStream(callbacks)` (`upstream_request.h:33`): call
   `conn_pool_data_->newConnection(*this)` to grab a TCP connection;
   `upstream_handle_` stores the cancel handle.
3. `onPoolReady(conn_data, host)` (`upstream_request.cc:24`): enable
   half-close on the connection, wrap it in a `TcpUpstream`, and hand it
   back to the router via `callbacks_->onPoolReady(...)`.
4. `TcpUpstream::encodeHeaders` (`upstream_request.cc:49`): emit a
   PROXY-protocol header if the route's `connect_config` has
   `proxy_protocol_config`, then synthesize a `:status 200` response
   downstream so the CONNECT handshake is complete.
5. `encodeData` / `encodeTrailers` write bytes straight to the upstream
   connection; `readDisable` pauses reads if the connection is still open.
6. `onUpstreamData` (`upstream_request.cc:101`) passes bytes back through
   `upstream_request_->decodeData(...)`. With
   `allow_multiplexed_upstream_half_close`, an upstream-initiated half-close
   before downstream completion is converted into a full stream reset so the
   TCP proxy semantics remain intact.
7. `onEvent` turns local/remote close into
   `onResetStream(ConnectionTermination)` while a request is in flight.

## Key decision points
- `upstream_request.cc:49-80` - CONNECT handshake completion (proxy-proto
  + synthesized 200 response).
- `upstream_request.cc:101-124` - upstream half-close -> full-close
  promotion.
- `upstream_request.h:55-62` - `resetUpstreamHandleIfSet` cancels any
  pending connection attempt.

## Configuration
Driven by the route's `connect_config` (including optional
`proxy_protocol_config`) and the runtime feature
`envoy.reloadable_features.allow_multiplexed_upstream_half_close`.

## Stats
None emitted here; the TCP connection pool supplies its own cluster stats.
