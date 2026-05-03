# CORS Filter (`envoy.filters.http.cors`)

Implements the CORS protocol: answers preflight `OPTIONS` requests locally and
stamps `Access-Control-Allow-*` headers onto responses for simple/actual
requests.

Proto: `envoy.extensions.filters.http.cors.v3.Cors`.

## Lifecycle

- **`decodeHeaders()`** (`cors_filter.cc:85–181`)
  - Detects preflight: `OPTIONS` + `Origin` + `Access-Control-Request-Method`.
  - If the origin matches `allow_origins`, respond locally with the configured
    `Access-Control-Allow-*` headers and 200/204.
  - If no match and `forward_not_matching_preflights=true`, pass the preflight
    through to the upstream; otherwise answer locally without Allow headers.
  - For non-preflight, latches the request `Origin` into `latched_origin_` for
    the encode phase.
- **`encodeHeaders()`** (`cors_filter.cc:185–206`) — on non-preflight responses
  where `latched_origin_` was set and the origin was allowed, injects the
  response CORS headers.

## Origin matching

`isOriginAllowed()` (`cors_filter.cc:208–215`) — iterates the
`allow_origins` `StringMatcher` list (supports exact, prefix, suffix, regex,
contains, wildcard `*`).

## Configuration

Configured on the route / virtual host as `CorsPolicy`. Route-level overrides
win over virtual host (`cors_filter.cc:56–80`).

- `allow_origin_string_match`, `allow_methods`, `allow_headers`,
  `expose_headers`, `max_age`
- `allow_credentials`, `allow_private_network_access`
- `filter_enabled`, `shadow_enabled` — runtime fractional gates
- `forward_not_matching_preflights` — pass unknown-origin preflights upstream.

## Stats

- `origin_valid` — origin matched an allow entry.
- `origin_invalid` — origin did not match.

## Files

- `cors_filter.{h,cc}` — filter implementation.
- `config.{h,cc}` — factory.
