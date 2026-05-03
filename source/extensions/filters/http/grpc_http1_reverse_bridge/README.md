# gRPC HTTP/1.1 Reverse Bridge (`envoy.filters.http.grpc_http1_reverse_bridge`)

The mirror image of `grpc_http1_bridge`: accepts an incoming gRPC request, downgrades it to a plain HTTP/1.1 request for a non-gRPC upstream (stripping the optional gRPC frame header and rewriting `content-type`), then re-wraps the upstream HTTP response as a gRPC response with a synthesized frame header, `grpc-status` trailer, and HTTP 200 status.

Proto: `envoy.extensions.filters.http.grpc_http1_reverse_bridge.v3.FilterConfig` (+ `FilterConfigPerRoute`).

## Files
- `config.h/cc` — `Config` (`FactoryBase`) for both the main filter config and the per-route config. Constructs `Filter` with `(content_type, withhold_grpc_frames, response_size_header)`.
- `filter.h/cc` — `Filter` (`PassThroughFilter`) plus `FilterConfigPerRoute` (holds just `disabled_`). Also registers the inline Accept header handle for fast rewriting.

## Lifecycle
- `decodeHeaders` (filter.cc:77): short-circuits to `Continue` if header-only. Resolves per-route config via `Http::Utility::resolveMostSpecificPerFilterConfig<FilterConfigPerRoute>`; if disabled there, sets `enabled_ = false` and passes through. For a gRPC request, sets `enabled_ = true`, saves the original content-type into `content_type_`, rewrites request `content-type` and `Accept` to `upstream_content_type_`, and if `withhold_grpc_frames_` adjusts the request `content-length` by `-GRPC_FRAME_HEADER_SIZE`. Clears the route cache so post-rewrite routing applies.
- `decodeData` (filter.cc:118): when enabled and withholding frames, on the first call guards against too-small bodies (`sendLocalReply` with `grpc_bridge_data_too_small`) and drains the 5-byte gRPC frame header exactly once (`prefix_stripped_`).
- `encodeHeaders` (filter.cc:136): validates upstream `content-type` matches `upstream_content_type_`; mismatch returns a synthetic gRPC error reply (`grpc_bridge_content_type_wrong`). Restores the downstream content-type. If `withhold_grpc_frames_` and `response_size_header_` is configured, reads the size from that header and sets `content-length = size + 5`; missing/unparseable size returns `grpc_bridge_content_length_missing`. If no size header, adds `+5` to any existing `content-length` (buffered path). Captures `grpc_status_` via `grpcStatusFromHeaders` (upstream 200 => OK, else `httpToGrpcStatus`). Overrides HTTP status to 200 for gRPC clients. Header-only responses (`end_stream` + withholding) synthesize a zero-length gRPC frame and attach a trailers map with `grpc-status`.
- `encodeData` (filter.cc:206): tracks `upstream_response_bytes_`. When streaming via `response_size_header_`, prepends the gRPC frame header to the first chunk (`frame_header_added_`). On `end_stream`, appends a trailers map with `grpc-status`; when buffering (no size header) moves the accumulated `buffer_` before the current chunk and prepends the frame header with the now-known length. When `response_size_header_` is set, validates actual bytes match the announced size, else rejects with `grpc_bridge_content_length_wrong`. When withholding and buffering (no size header), returns `StopIterationAndBuffer`.
- `encodeTrailers` (filter.cc:256): adds `grpc-status`. If withholding without a size header, finalizes the buffered response by prepending a frame header and calling `addEncodedData`.
- `buildGrpcFrameHeader` (filter.cc:271) uses `Grpc::Encoder().prependFrameHeader(GRPC_FH_DEFAULT, ...)`.

## Decision / logic
- `enabled_` gate set only for real gRPC requests AND non-disabled route (filter.cc:87, 96).
- Three streaming modes driven by `withhold_grpc_frames_` and `response_size_header_`:
  - off — no frame stripping; upstream already produces framed bytes.
  - on + header — stream through, prepend frame header from announced length, validate total at end.
  - on + no header — buffer the entire response, compute length, then emit.
- `grpcStatusFromHeaders` (filter.cc:38) treats upstream HTTP 200 as `OK` even though strict HTTP semantics might require 2xx range; intentional for gRPC bridging.
- Every failure path returns HTTP 200 with a non-OK `grpc-status` so gRPC clients see a proper gRPC error.

## Configuration
- `content_type` — upstream content-type (e.g. `application/x-protobuf`).
- `withhold_grpc_frames` — strip/synthesize the 5-byte gRPC frame header.
- `response_size_header` — name of the upstream header carrying the response body size; enables streaming instead of buffering.
- Per-route: `FilterConfigPerRoute.disabled` — short-circuits the filter (filter.cc:85-91).

## Stats
- No counters or gauges. Observability is via `response_code_details` (`RcDetailsValues` at filter.cc:21-34):
  - `grpc_bridge_data_too_small`
  - `grpc_bridge_content_type_wrong`
  - `grpc_bridge_content_length_missing`
  - `grpc_bridge_content_length_wrong`

## Factory
- `REGISTER_FACTORY(Config, NamedHttpFilterConfigFactory)` (config.cc:43). Name `envoy.filters.http.grpc_http1_reverse_bridge`.
