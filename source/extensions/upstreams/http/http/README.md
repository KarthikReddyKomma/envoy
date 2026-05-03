# HTTP-over-HTTP Upstream

The standard router-side connection pool used for HTTP upstream requests.
Wraps the thread-local cluster's HTTP pool and produces an `HttpUpstream`
that forwards headers/data/trailers from the router to the upstream encoder.

Proto: `envoy.extensions.upstreams.http.generic.v3.GenericConnectionPoolProto`
(this directory is reachable via the `generic` factory; no dedicated proto).

## Files
- `upstream_request.h` - `HttpConnPool` and `HttpUpstream` class
  declarations.
- `upstream_request.cc` - `newStream`, pool-ready / pool-failure callbacks.
- `config.h` / `config.cc` - module-local config glue.

## Interface
- `HttpConnPool` implements `Router::GenericConnPool` and
  `Envoy::Http::ConnectionPool::Callbacks`.
- `HttpUpstream` implements `Router::GenericUpstream` and
  `Envoy::Http::StreamCallbacks`.

## Logic
1. Construction (`upstream_request.h:23`): fetch the thread-local cluster's
   HTTP pool for the given host, priority, and downstream protocol via
   `thread_local_cluster.httpConnPool(...)`. Result is stored in
   `pool_data_`; `valid()` returns true iff `pool_data_.has_value()`.
2. `newStream(callbacks)` (`upstream_request.cc:29`): call
   `pool_data_->newStream(upstreamToDownstream, *this, options)`. If the
   pool returns a non-null cancel handle it is stored in
   `conn_pool_stream_handle_` so a later `cancelAnyPendingStream` can cancel
   it. The pool may reset the stream inline, so the handle is only captured
   when the callback hasn't already fired.
3. `onPoolReady(encoder, host, info, protocol)` (`upstream_request.cc:58`):
   build an `HttpUpstream` wrapping the router's `UpstreamToDownstream` and
   the provided `RequestEncoder`, then call
   `callbacks_->onPoolReady(std::move(upstream), host, connectionInfoProvider,
   info, protocol)`.
4. `onPoolFailure` forwards the reason + host to the router callbacks.

`HttpUpstream` then acts as a thin pass-through:
- `encodeHeaders` / `encodeData` / `encodeTrailers` / `encodeMetadata`
  delegate to the `RequestEncoder`.
- `readDisable`, `resetStream`, `setAccount` poke the underlying stream.
- `onResetStream`, watermark callbacks propagate stream events back up the
  `UpstreamToDownstream` chain.

## Key decision points
- `upstream_request.cc:29-40` - inline-reset safety when capturing the
  pool's cancel handle.
- `upstream_request.cc:58-67` - wrap the encoder into an `HttpUpstream`
  before notifying the router.
- `upstream_request.h:82-85` - reset semantics: remove callbacks before
  issuing `resetStream(LocalReset)`.

## Configuration
No direct configuration; pool selection is driven by the owning cluster's
`HttpProtocolOptions` and the per-route upstream protocol choice.

## Stats
Emitted by the underlying cluster's HTTP connection pool.
