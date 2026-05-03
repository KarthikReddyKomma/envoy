# Cookie-Based Stateful Session

A `SessionStateFactory` consumed by the `stateful_session` HTTP filter
(`source/extensions/filters/http/stateful_session`) and the router's
stateful-session host selection. For each request the filter asks the
factory to produce a `SessionState`; this extension encodes the selected
upstream host's address in a client cookie so that subsequent requests
from the same client are pinned to the same upstream.

Proto: `envoy.extensions.http.stateful_session.cookie.v3.CookieBasedSessionState`.

## Files
- `cookie.h/cc` - factory + per-request `SessionStateImpl`.
- `cookie.proto` - internal wire format stored inside the cookie value
  (`envoy.Cookie` with `address` and `expires` fields).
- `config.h/cc` - factory registration.

## Interface
- Implements `Envoy::Http::SessionStateFactory`. Per request, `create`
  returns a `SessionStatePtr` (or `nullptr` if the request path does not
  match the configured cookie `path`).
- `SessionState` exposes `upstreamAddress()` (what the router should
  pin to) and `onUpdate(host_address, response_headers)` (called after
  the response is produced so the cookie can be (re)issued if the host
  changed).
- Factory registered via
  `Envoy::Http::SessionStateFactoryConfigFactory`.

## Logic
- Construction builds a `path_matcher_` lambda based on the configured
  cookie path (`cookie.cc:36`):
  - Empty or `/` path -> match everything.
  - Path ending in `/` -> prefix match.
  - Other path -> exact match, or prefix match where the next character
    is `/`, `?`, or `#` (RFC 6265 semantics).
- `create` early-returns `nullptr` if `path_matcher_` rejects the
  request - callers then skip stateful behavior for that request.
- `parseAddress`
  - Extracts the cookie value with `Http::Utility::parseCookieValue`.
  - Base64-decodes it, then tries to parse as `envoy.Cookie`. If parsed
    successfully, reads `address` and, if `expires != 0`, compares
    against `monotonicTime()` - an expired cookie yields `nullopt` so
    the router picks a fresh host.
  - Falls back to treating the raw decoded value as the address for
    backward compatibility with pre-proto cookies (logs a deprecation
    warning exactly once via `ENVOY_LOG_ONCE_MISC`).
- `SessionStateImpl::onUpdate` compares the actually-used host address
  against the cookie-provided one. On mismatch (or no prior cookie), it
  encodes a fresh `envoy.Cookie` with `address` and TTL-based
  `expires`, base64s it, and appends a `Set-Cookie` header built via
  `Http::Utility::makeSetCookieValue` using the configured `path`, `ttl`
  and attributes.

## Key decision points
- `cookie.h:44` - `create` returns nullptr on path mismatch, disabling
  pinning per-request.
- `cookie.h:67` - envoy.Cookie proto is the current format; raw value is
  a deprecated legacy path.
- `cookie.h:75` - expiry check uses monotonic clock (avoids wall-clock
  changes).
- `cookie.cc:14` - `onUpdate` only rewrites the cookie when the host
  actually changed.
- `cookie.cc:40` - empty cookie name throws at config load.

## Configuration
- `cookie.name` (required).
- `cookie.ttl` - cookie TTL, encoded both as `Max-Age` in `Set-Cookie`
  and as `expires` in the proto payload.
- `cookie.path` - scope of the cookie and of the path matcher.
- `cookie.attributes` - arbitrary additional `Set-Cookie` attributes
  (e.g. `Secure`, `HttpOnly`, `SameSite`).

## Stats / errors
No dedicated stats. Empty cookie name throws at config load; expired or
unparsable cookies silently degrade to "no pinning" for that request.
