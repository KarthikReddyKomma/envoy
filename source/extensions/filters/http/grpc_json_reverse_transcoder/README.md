# gRPC-JSON Reverse Transcoder (`envoy.filters.http.grpc_json_reverse_transcoder`)

Bridges an incoming gRPC unary request to a JSON REST upstream: decodes the gRPC-framed protobuf body, maps it to an HTTP method/path/body via the `google.api.http` annotation, calls the REST backend, and re-encodes the JSON response as a gRPC-framed protobuf message back to the caller. Only unary methods are supported; streaming is rejected.

Proto: `envoy.extensions.filters.http.grpc_json_reverse_transcoder.v3.GrpcJsonReverseTranscoder`.

## Files
- `config.h/cc` — `GrpcJsonReverseTranscoderFactory`; main and per-route factory hooks both build a `GrpcJsonReverseTranscoderConfig`. Per-route config reuses the same config type.
- `filter_config.h/cc` — `GrpcJsonReverseTranscoderConfig` (also a `RouteSpecificFilterConfig`): loads the descriptor pool, builds a `TypeHelper`, stores `max_request_body_size_`, `max_response_body_size_`, `api_version_header_`, and the JSON print options. Exposes `GetMethodDescriptor`, `IsRequestNestedHttpBody`, `CreateTranscoder`, `ChangeBodyFieldName`. `TranscoderImpl` glues a `ResponseToJsonTranslator` (request side, proto->JSON) with a `JsonRequestTranslator` (response side, JSON->proto).
- `filter.h/cc` — `GrpcJsonReverseTranscoderFilter` (`StreamFilter`) and the thin `TranscoderInputStreamImpl` used to feed the transcoding library.
- `utils.h/cc` — `BuildPath`, `BuildGrpcMessage` and related helpers.

## Lifecycle
- `decodeHeaders` (filter.cc:373): pass-through for header-only or non-gRPC requests. Else `InitPerRouteConfig`, look up `MethodDescriptor` from `:path`; not-found rejects with `BadRequest`/`InvalidArgument`/`grpc_transcode_failed_early`. Rejects client/server streaming methods. Extracts HTTP method/path/body field from the `google.api.http` annotation (`ExtractHttpAnnotationValues`, filter.cc:307) including `get|post|put|delete|patch|custom` and rewrites the body field through `ChangeBodyFieldName` for `json_name`/camelCase. Flags `is_request_http_body_` / `is_response_http_body_` when the input/output type is `google.api.HttpBody`, and `is_request_nested_http_body_` when the body field itself is an `HttpBody`. Creates the transcoder, replaces `{$api_version}` in the path via `api_version_header_`, maybe expands buffer limits, stores original `content-type` in `request_content_type_`, sets request `content-type: application/json`, sets the REST method, drops `content-length`, clears the route cache, returns `StopIteration`.
- `decodeData` (filter.cc:460): pass-through if no transcoder. For `HttpBody` input, rejects paths that contain variables (only `{$api_version}` allowed) then calls `BuildRequestFromHttpBody` which decodes gRPC frames, parses the `HttpBody` message, and swaps `content-type`/`content-length`/`:path` to mirror the embedded HTTP body. For typed messages, moves data into `request_in_`, enforces `DecoderBufferLimitReached`, checks transcoder status, and on `end_stream` parses the transcoded JSON to either forward as-is (body="*"), extract a named sub-field via `CreateDataBuffer` (handles nested `HttpBody` with `WebSafeBase64Unescape`), or build the final URL path via `BuildPath`.
- `encodeHeaders` (filter.cc:548): marks `is_response_passed_through_` if upstream already returned gRPC. Else replaces response `content-type` with the original request content-type, computes `grpc_status_` via `GrpcStatusFromHeaders` (2xx => OK), forces HTTP 200, drops `content-length`. Header-only responses produce a zero-length gRPC frame and trailers-only `grpc-status`.
- `encodeData` (filter.cc:582): pass-through when appropriate. Non-OK responses are buffered into `error_buffer_` and later attached as `grpc-message` on the trailers (via `BuildGrpcMessage`). `HttpBody` responses accumulate into `response_data_` and are emitted via `SendHttpBodyResponse` which wraps them in a `google.api.HttpBody` proto envelope (`AppendHttpBodyEnvelope`, filter.cc:69) and prepends the gRPC frame. Regular JSON responses feed `response_in_` and pull transcoded proto bytes into `response_buffer_`. Response size is capped by `EncoderBufferLimitReached`.
- `encodeTrailers` (filter.cc:632): sets `grpc-status`, finalizes non-OK / `HttpBody` / buffered-JSON paths, and finishes `response_in_`.

## Decision / logic
- Streaming rejected early (filter.cc:396).
- HttpBody path constraint: only `{$api_version}` allowed in the URL when the request type is `google.api.HttpBody` (filter.cc:468).
- Body field renaming respects proto `json_name` and rewrites path templates that reference fields under the body (filter.cc:352-358).
- `is_response_passed_through_` skips the whole encode path when the upstream is actually gRPC (filter.cc:550-556), useful when the filter is enabled on a route that might still talk gRPC.
- `MaybeExpandBufferLimits` (filter.cc:152) bumps the per-stream buffer limit when the proto caps exceed the defaults; `DecoderBufferLimitReached`/`EncoderBufferLimitReached` use those caps.
- API version replacement only happens when `api_version_header_` is configured and the header is present (filter.cc:132).

## Configuration
- `descriptor_set` — protobuf descriptor source used to build the method registry.
- `api_version_header` — request header whose value replaces `{$api_version}` in path templates.
- `max_request_body_size`, `max_response_body_size` — optional caps; also raise the stream buffer limits.
- JSON print options (preserve-field-names, etc.) flow from the proto into `request_translate_options_`.
- Per-route override: providing the same message type on a route replaces the listener config (`createRouteSpecificFilterConfigTyped` returns a full `GrpcJsonReverseTranscoderConfig`, config.cc:39).

## Stats
- No counters/gauges. Failure paths encode the cause into `response_code_details`:
  - `early_grpc_json_reverse_transcode_failure{<code>}` — setup failures during `decodeHeaders` (method not found, streaming, transcoder creation).
  - `grpc_json_reverse_transcode_failure{<code|reason>}` — body-time failures (`request_buffer_size_limit_reached`, `response_buffer_size_limit_reached`, `failed_to_create_request_body`, `failed_to_decode_request_body`, `failed_to_parse_request_body`, `failed_to_build_request_path`, `failed_to_create_request_path`, and raw gRPC codes from transcoder errors).

## Factory
- `REGISTER_FACTORY(GrpcJsonReverseTranscoderFactory, NamedHttpFilterConfigFactory)` (config.cc:51). Name `envoy.filters.http.grpc_json_reverse_transcoder`.
