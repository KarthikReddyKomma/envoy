# Connect-gRPC Bridge (`envoy.filters.http.connect_grpc_bridge`)

Translates Connect RPC (unary, unary-GET, and streaming) requests/responses to and from gRPC so a Connect client can talk to a gRPC upstream transparently. On request, it rewrites content-type, TE, timeout, and encoding headers and (for unary) wraps the body with a gRPC length-prefixed frame. On response, it converts gRPC trailers back into Connect error JSON envelopes or trailer-prefixed headers.

Proto: `envoy.extensions.filters.http.connect_grpc_bridge.v3.FilterConfig` (empty — no runtime configuration).

## Files
- `filter.h/cc` — `ConnectGrpcBridgeFilter` (both decoder and encoder sides).
- `config.h/cc` — `ConnectGrpcFilterConfigFactory`, static `REGISTER_FACTORY` (`config.cc:34`).
- `end_stream_response.h/cc` — `EndStreamResponse`/`Error` JSON (de)serialization used to build Connect streaming EOS frames and unary error bodies.

## Lifecycle
Implemented as `Http::PassThroughFilter` (`filter.h:17`). `FilterConfigFactory::createFilterFactoryFromProtoTyped` installs a new filter per stream via `addStreamFilter` (`config.cc:17-19`). The filter classifies the request in `decodeHeaders` and sets one of three mutually exclusive mode flags: `is_connect_streaming_`, `is_connect_unary_`, or neither (pass-through).

- `decodeHeaders` (`filter.cc:233`):
  - Streaming branch (content-type starts with `application/connect+<codec>`, `filter.cc:237`): rewrites content-type to `application/grpc+<codec>`, calls `convertConnectTimeoutToGrpcTimeout` (`filter.cc:94`), removes `connect-protocol-version`, sets `TE: trailers`, renames `content-encoding`/`accept-encoding` to gRPC variants, clears `content-length`.
  - Unary POST branch (`connect-protocol-version` present and content-type starts with `application/`, `filter.cc:263`): same header rewriting; derives `unary_payload_frame_flags_` (`filter.cc:287-291`) from the encoding header; if `end_stream`, prepends the gRPC frame header to the empty `request_buffer_` and injects it (`filter.cc:295-298`).
  - Unary GET branch (`?connect=v1` query param, `filter.cc:300-303`): converts to POST, strips query params, base64-decodes `message=` if `base64=1` (`filter.cc:318-320`), builds the gRPC frame.
- `decodeData` (`filter.cc:359`): for unary only, accumulates into `request_buffer_`; on `end_stream`, prepends the gRPC frame header and moves buffered bytes back into `data`.
- `decodeTrailers` (`filter.cc:378`): for unary (zero-body case), still injects the prefixed frame.
- `encodeHeaders` (`filter.cc:387`): gated by `Grpc::Common::isGrpcResponseHeaders` (`filter.cc:392`); on non-gRPC responses, pass-through. Rewrites `application/grpc+<codec>` back to Connect content-type (`filter.cc:405-414` streaming, `filter.cc:435-444` unary). Streaming: renames `grpc-encoding`/`grpc-accept-encoding` back; trailers-only response is converted immediately via `convertGrpcResponseToConnectStreamingResponse` (`filter.cc:427-430`). Unary: `StopIteration` if `!end_stream` to collect the body (`filter.cc:451-453`); on trailers-only unary, may emit an error JSON body via `convertGrpcResponseToConnectUnaryResponse` (`filter.cc:456-461`). gRPC-prefixed headers are stripped at the end of both branches.
- `encodeData` (`filter.cc:469`): for unary gRPC responses, strips the leading 5-byte gRPC frame header (`filter.cc:479-484`), buffers the rest into `response_buffer_`, and releases it on `end_stream`. If the frame flag byte indicates uncompressed, removes `content-encoding` (`filter.cc:474-477`).
- `encodeTrailers` (`filter.cc:501`): streaming mode produces a Connect EOS frame from trailers and clears them (`filter.cc:507-510`); unary copies non-`grpc-` trailers into response headers with a `trailer-` prefix (`filter.cc:207-219`, called at `filter.cc:514`), then emits either the error JSON or the buffered payload (`filter.cc:516-523`).

## Decision / logic
- Request mode selection: `filter.cc:237` (streaming), `filter.cc:263` (unary POST), `filter.cc:302` (unary GET). Modes are mutually exclusive.
- gRPC timeout encoding picks the smallest unit that fits in 8 digits (`filter.cc:71-92`).
- Compression flag bit is set when an encoding is present and not `identity` (`filter.cc:288-291`, `filter.cc:326-329`).
- Unary response buffering is bounded by the stream's configured buffer limit. Overflow on request → `413` with rc-detail `connect_bridge_unary_request_too_large` (`filter.cc:531-544`). Overflow on response → `500` with rc-detail `connect_bridge_unary_response_too_large` (`filter.cc:546-561`).
- For gRPC status `!= 0`, an `Error` is built from `grpc-status`, `grpc-message`, and optional `grpc-status-details-bin`, and the HTTP status is mapped via `statusCodeToConnectUnaryStatus` (`filter.cc:194`).
- On unary gRPC responses, the `content-encoding` header is removed if the first frame is uncompressed (`filter.cc:473-478`).

## Configuration
The proto is empty; there are no fields and no per-route overrides. All behavior is driven by the request headers / query string.

## Stats
None emitted by this filter; it relies on the response-code/rc-details plumbing for error visibility.
