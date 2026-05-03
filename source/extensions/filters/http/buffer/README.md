# Buffer Filter (`envoy.filters.http.buffer`)

Accumulates the entire request body before the downstream filter chain
continues decoding. Used when a later filter (auth, transform, WAF) needs the
full body in one piece, or when a `Content-Length` must be synthesised.

Proto: `envoy.extensions.filters.http.buffer.v3.Buffer`.

## Lifecycle

Pure decoder filter.

- **`decodeHeaders()` (`buffer_filter.cc:50`)**
  - If `end_stream` (no body) or the route disables the filter → pass through.
  - Set the stream buffer limit from `max_request_bytes` (line 63).
  - Return `StopIteration` (line 66) so the chain pauses.
- **`decodeData()` (`buffer_filter.cc:69`)**
  - Accumulates bytes. Returns `StopIterationAndBuffer` until `end_stream`.
- **`decodeTrailers()` (line 81)** — flushes the accumulated body.
- On body completion: inject `Content-Length` if missing
  (`maybeAddContentLength`, line 91) and continue iteration.

If the accumulated body exceeds `max_request_bytes`, the HTTP connection
manager returns **413 Payload Too Large** (buffer limit is enforced there).

## Configuration

- `max_request_bytes` — hard cap on request body size.
- Per-route `BufferPerRoute` — disable the filter for a specific route.

## Stats

None directly; relies on HCM's payload-too-large stat.

## Files

- `buffer_filter.{h,cc}` — filter.
- `config.{h,cc}` — factory.
