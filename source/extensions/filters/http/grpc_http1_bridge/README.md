# gRPC HTTP/1.1 Bridge (`envoy.filters.http.grpc_http1_bridge`)

Bidirectional bridge that lets an HTTP/1.1 client speak to an HTTP/2 gRPC upstream. On the request side it optionally wraps a raw protobuf body in a gRPC frame; on the response side it strips the gRPC frame, buffers the body, and promotes the gRPC trailers `grpc-status` / `grpc-message` into response headers (with `Content-Length`) so HTTP/1.1 clients see a conventional response.

Proto: `envoy.extensions.filters.http.grpc_http1_bridge.v3.Config`.

## Files
- `config.h/cc` — `GrpcHttp1BridgeFilterConfig` (`FactoryBase`); adds the filter via `callbacks.addStreamFilter` with a shared `Http1BridgeFilter`. Has both factory context and server-context variants.
- `http1_bridge_filter.h/cc` — `Http1BridgeFilter` implementing `Http::StreamFilter` (both decoder and encoder interfaces) plus the `ignoreQueryParams` helper.

## Lifecycle
- `decodeHeaders` (http1_bridge_filter.cc:34): if `upgrade_protobuf_` and `isProtobufRequestHeaders`, sets `do_framing_ = true`, rewrites content-type to `application/grpc`, removes `content-length`, and clears the route cache so routing re-evaluates with the new content-type. Sets `do_bridging_ = true` when the downstream protocol is below HTTP/2 and the request is gRPC. If `do_bridging_ && ignore_query_parameters_`, strips the query string via `ignoreQueryParams`. Always `Continue`.
- `decodeData` (http1_bridge_filter.cc:57): passes through unless both `do_bridging_` and `do_framing_`. Otherwise appends into the decoding buffer via `addDecodedData`; on `end_stream` uses `modifyDecodingBuffer` to prepend a gRPC frame header (`Grpc::Common::prependGrpcFrameHeader`) and `Continue`; otherwise `StopIterationAndBuffer`.
- `decodeTrailers` (http1_bridge_filter.h:36): `Continue`.
- `encodeHeaders` (http1_bridge_filter.cc:72): when bridging and not `end_stream`, stashes `response_headers_` and returns `StopIteration` so trailers can be merged in before the client sees headers. Otherwise `Continue`.
- `encodeData` (http1_bridge_filter.cc:83): when bridging, if the request was upgraded (`do_framing_` still true on response), drains the first 5 bytes (`GRPC_FRAME_HEADER_SIZE`) once and clears `do_framing_`. Returns `StopIterationAndBuffer` to accumulate the full body until trailers arrive, unless `end_stream`.
- `encodeTrailers` (http1_bridge_filter.cc:99): when bridging, parses `grpc-status`; non-zero or unparseable sets HTTP status to 503. Copies `grpc-status` and `grpc-message` onto `response_headers_` and sets `content-length` from `encoder_callbacks_->encodingBuffer()->length()` so HTTP/1 clients get a framed response. Always returns `Continue`; the HTTP/1 codec discards the trailers.

## Decision / logic
- `do_bridging_` is set only when the downstream protocol is pre-HTTP/2 AND the request looks gRPC (http1_bridge_filter.cc:47). This guarantees no-op for HTTP/2 gRPC.
- `do_framing_` is independent and opt-in via `upgrade_protobuf_to_grpc` (http1_bridge_filter.cc:36), used for clients that send raw `application/x-protobuf`.
- Query-param stripping runs only when bridging and the flag is on (http1_bridge_filter.cc:51), to avoid breaking native gRPC upstreams that reject unknown query parameters.
- Response 503 override keys on `grpc-status` parse failure OR non-OK status (http1_bridge_filter.cc:108).

## Configuration
- `upgrade_protobuf_to_grpc` — bridge from `application/x-protobuf` to `application/grpc` on the request.
- `ignore_query_parameters` — strip `?...` from the path when bridging.
- No per-route config.

## Stats
- No counters or gauges are emitted by this filter. Stats tracking of gRPC success/failure is delegated to the separate `grpc_stats` filter or the codec layer via `Grpc::Context`.

## Factory
- `LEGACY_REGISTER_FACTORY(GrpcHttp1BridgeFilterConfig, NamedHttpFilterConfigFactory, "envoy.grpc_http1_bridge")` (config.cc:34). Canonical name is `envoy.filters.http.grpc_http1_bridge`.
