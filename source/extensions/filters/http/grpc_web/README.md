# gRPC-Web Filter (`envoy.filters.http.grpc_web`)

Transcodes gRPC-Web (used by browsers) into standard gRPC for the upstream and
back. Handles both the binary and the base64 "text" encodings.

Proto: `envoy.extensions.filters.http.grpc_web.v3.GrpcWeb`.

## Lifecycle

- **`decodeHeaders()`** (`grpc_web_filter.cc:154–200`)
  - Detects a gRPC-Web request by `Content-Type`.
  - If text variant, flips `is_text_request_`.
  - Normalises `Content-Type` to `application/grpc`.
  - Adds `TE: trailers`, `grpc-accept-encoding: identity`.
  - Removes `Content-Length` (HTTP/2 streams don't preserve it).
- **`decodeData()`** — base64-decodes the body when `is_text_request_`.
- **`encodeHeaders()`** (`grpc_web_filter.cc:241–271`)
  - If the response is proto gRPC, set up translation of trailers into a
    trailers frame at the end of the body (gRPC-Web does not carry real
    trailers).
- **`encodeData()` / `encodeTrailers()`** — buffer response bytes; on
  stream end, append the trailers frame and, if `is_text_response_`, base64
  encode the whole payload.

## Stats

Standard gRPC stats via `Grpc::Common::chargeStat` (per-service / per-method
`success` / `failure`).

## Files

- `grpc_web_filter.{h,cc}` — filter.
- `config.{h,cc}` — factory.
