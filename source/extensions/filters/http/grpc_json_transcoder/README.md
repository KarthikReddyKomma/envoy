# gRPC-JSON Transcoder (`envoy.filters.http.grpc_json_transcoder`)

Transcodes incoming HTTP/JSON requests into gRPC requests for a gRPC upstream and transcodes the gRPC response back to JSON for the client. Uses the `grpc-httpjson-transcoding` library, driven by `google.api.http` annotations in the supplied protobuf descriptor set. Supports unary, client-streaming, server-streaming, bidi-streaming (via SSE-style delimited output), and `google.api.HttpBody` passthrough on either side.

Proto: `envoy.extensions.filters.http.grpc_json_transcoder.v3.GrpcJsonTranscoder`.

## Files
- `config.h/cc` — `GrpcJsonTranscoderFilterConfig` (`FactoryBase`); creates one `JsonTranscoderConfig` and a stats struct, supports per-route config overriding the descriptor pool.
- `json_transcoder_filter.h/cc` — `JsonTranscoderConfig` (listener/route config: descriptor pool, `PathMatcher`, `TypeHelper`, ignored query params, request-validation options, response translate options) and `JsonTranscoderFilter` (`StreamFilter`). Inner `TranscoderImpl` wraps a `RequestMessageTranslator` or `JsonRequestTranslator` and a `ResponseToJsonTranslator`.
- `stats.h` — stats macros (`transcoder_request_buffer_bytes`, `transcoder_response_buffer_bytes` gauges).
- `transcoder_input_stream_impl.h/cc` — Envoy-buffer-backed `TranscoderInputStream` implementation.
- `http_body_utils.h/cc` — helpers for constructing / parsing `google.api.HttpBody` proto envelopes by field path.

## Lifecycle
- `decodeHeaders` (json_transcoder_filter.cc:462): `initPerRouteConfig`; if disabled or the request is already gRPC, pass through. Otherwise calls `per_route_config_->createTranscoder(...)` which matches the path via `PathMatcher` and builds the transcoder. `NotFound` may pass through based on `request_validation_options_.reject_unknown_method()`; `InvalidArgument` (unknown query params) likewise keyed by `reject_unknown_query_parameters()`. Any other failure sends a local reply with code mapped from the gRPC status. On success: `maybeExpandBufferLimits`, then for `HttpBody` input synthesizes an initial request chunk from query/path bindings and stashes it in `initial_request_data_`. Rewrites the request to gRPC form: drops `content-length`, sets `content-type: application/grpc`, stores `x-envoy-original-path` / `x-envoy-original-method`, sets `:path = /service/method`, method `POST`, `TE: trailers`. Clears the route cache unless `matchIncomingRequestInfo()`. If `end_stream` and `HttpBody`, emits the request message; otherwise flushes any remaining buffered bytes via `addDecodedData`.
- `decodeData` (json_transcoder_filter.cc:586): pass-through if no transcoder. For `HttpBody` requests, accumulates bytes into `request_data_` and, for client-streaming, chunks into 1MB pieces (`MaxStreamedPieceSize`) each sent via `maybeSendHttpBodyRequestMessage` (which calls `HttpBodyUtils::appendHttpBodyEnvelope` + `Grpc::Encoder().prependFrameHeader`). For typed requests feeds `request_in_`, checks decoder buffer limit, finalizes on end-of-stream, and drains the transcoded bytes with `readToBuffer` — adjusting the `transcoder_request_buffer_bytes_` gauge as bytes move from stream to output. Fails via `checkAndRejectIfRequestTranscoderFailed` (json_transcoder_filter.cc:866).
- `decodeTrailers` (json_transcoder_filter.cc:646): for `HttpBody` requests flushes the final message; for typed ones finishes `request_in_` and flushes any remaining transcoded bytes.
- `encodeHeaders` (json_transcoder_filter.cc:679): if we never built a transcoder or responded early, pass through. Rejects non-gRPC upstream responses (`error_ = true`, pass through). Sets the response content-type to `application/json` (or `text/event-stream` when `stream_sse_style_delimited` + server streaming). Trailers-only responses feed directly into `doTrailers`. Non-streaming, non-streaming-HttpBody cases return `StopIteration` so the unary body can be buffered.
- `encodeData` (json_transcoder_filter.cc:729): for `HttpBody` responses feeds gRPC frames through `buildResponseFromHttpBodyOutput` (json_transcoder_filter.cc:956) extracting the nested `HttpBody` via `HttpBodyUtils::parseMessageByFieldPath`, setting content-type/length from the proto and streaming bytes. For typed responses feeds `response_in_`, enforces `encoderBufferLimitReached`, drains through `readToBuffer`, adjusting `transcoder_response_buffer_bytes_`. Unary non-end returns `StopIterationNoBuffer` to internally buffer.
- `encodeTrailers` (json_transcoder_filter.cc:784): delegates to `doTrailers` (json_transcoder_filter.cc:791) which finalizes the response stream, maps the gRPC status to HTTP status (unknown/invalid => 503), may call `maybeConvertGrpcStatus` to materialize `google.rpc.Status` into the body, copies `grpc-message`, strips the `Trailer` header on HTTP/1 clients, and sets `content-length` for non-streaming.
- `onDestroy` (json_transcoder_filter.cc:919): decrements both buffer gauges by any bytes still held in `request_data_`/`request_in_` and `response_data_`/`response_in_`.

## Decision / logic
- Transcoding is skipped when `disabled_` (no services configured; `per_route_config_->disabled()`) or when the incoming request is already gRPC (json_transcoder_filter.cc:472).
- Strictness is governed by `request_validation_options_`: unknown method / query params can either be rejected with the mapped HTTP status or transparently passed through (json_transcoder_filter.cc:487-503).
- `convert_grpc_status_` causes `maybeConvertGrpcStatus` to put a `google.rpc.Status` JSON body on the response when the upstream used gRPC trailers for an error; `addBuiltinSymbolDescriptor` ensures `Any`/`Status` are registered in the pool.
- Server-streaming responses use either SSE framing (`text/event-stream`) or newline-delimited JSON depending on `stream_sse_style_delimited`.
- Client-streaming `HttpBody` splits into 1MB frames so a single gRPC frame never exceeds the 4MB default grpc cap (json_transcoder_filter.h:56, .cc:602-612).
- `matchIncomingRequestInfo()` suppresses `clearRouteCache()` so routing can key on the original REST request.
- Error signaling uses `checkAndRejectIfRequestTranscoderFailed` / `checkAndRejectIfResponseTranscoderFailed` to `sendLocalReply` with the transcoder status message and a `response_code_details` tag.

## Configuration
- `proto_descriptor` / `proto_descriptor_bin` — descriptor source (json_transcoder_filter.cc:124).
- `services` — list of FQ gRPC service names to expose; empty disables the filter.
- `auto_mapping` — synthesize `POST /service/Method` with body `"*"` when no `google.api.http` annotation exists.
- `ignored_query_parameters`, `match_incoming_request_route`, `ignore_unknown_query_parameters`, `capture_unknown_query_parameters`, `convert_grpc_status`, `case_insensitive_enum_parsing`, `request_validation_options`, `max_request_body_size`, `max_response_body_size`, `print_options`, `query_param_unescape_plus`.
- Per-route: an entire `GrpcJsonTranscoder` message overrides the listener config (config.cc:42).

## Stats
Emitted by the filter itself (prefix = listener stats prefix):
- gauge `transcoder_request_buffer_bytes` (Accumulate)
- gauge `transcoder_response_buffer_bytes` (Accumulate)

RC details tags on failures: `early_grpc_json_transcode_failure{<code>}`, `grpc_json_transcode_failure{<code>}`.

## Factory
- `LEGACY_REGISTER_FACTORY(GrpcJsonTranscoderFilterConfig, NamedHttpFilterConfigFactory, "envoy.grpc_json_transcoder")` (config.cc:53). Canonical name `envoy.filters.http.grpc_json_transcoder`.
