# Compressor Filter (`envoy.filters.http.compressor`)

Compresses request and/or response bodies using a pluggable compressor library
(gzip, brotli, zstd, etc.). Bidirectional — each direction has its own
config.

Proto: `envoy.extensions.filters.http.compressor.v3.Compressor`.

## Lifecycle

- **Decode** — inspects the request, records `Accept-Encoding` for later, and
  may compress the request body itself if a `request_direction_config` is set.
- **Encode** — decides whether to compress the response body based on
  `Accept-Encoding`, `Content-Type`, `Content-Length`, `Cache-Control`, and
  response code.

`decodeHeaders()` entry: `compressor_filter.cc:248`. Per-route override logic:
`compressor_filter.cc:190–250`.

## Decision gates (response side)

A response is compressed only when every check passes:

1. Filter enabled (`compressionEnabled()`).
2. Response code not in `uncompressible_response_codes`.
3. `Content-Type` matches configured list (if any).
4. `Content-Length` ≥ `min_content_length` (if present).
5. `Cache-Control` does not contain `no-transform`.
6. `Accept-Encoding` includes the configured encoding with non-zero quality.
7. No upstream `Content-Encoding` already set (unless re-encoding is allowed).

## ETag handling

- `disable_on_etag_header=true` — skip compression when response carries an
  `ETag`.
- `remove_accept_encoding_header=true` — strip from upstream request.
- `weaken_etag_on_compress` — flip strong `"x"` ETags to weak `W/"x"`.

## Observability

- `envoy-compression-status` header (if `status_header_enabled`).
- Stats: `header_compressor_used`, `header_compressor_overridden`,
  `header_gzip`, `total_uncompressed_bytes`, `total_compressed_bytes`,
  `content_length_too_small`, `not_compressed`, `compressed`.

## Files

- `compressor_filter.{h,cc}` — filter + direction configs.
- `config.{h,cc}` — factory.
