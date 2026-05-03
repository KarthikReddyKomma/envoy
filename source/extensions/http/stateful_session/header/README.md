# Header-Based Stateful Session

A `SessionStateFactory` consumed by the `stateful_session` HTTP filter.
Mirrors the cookie strategy but uses a request/response header instead of
a `Set-Cookie`. The extension base64-encodes the selected upstream
address into a configurable header; clients are expected to echo the
header back on subsequent requests so Envoy can pin them to the same
upstream.

Proto: `envoy.extensions.http.stateful_session.header.v3.HeaderBasedSessionState`.

## Files
- `header.h/cc` - factory + `SessionStateImpl`.
- `config.h/cc` - factory registration.

## Interface
- Implements `Envoy::Http::SessionStateFactory`.
- `SessionState` implements `upstreamAddress()` + `onUpdate()` with the
  same contract as the other stateful session extensions.

## Logic
- `parseAddress` (`header.h:45`): read the configured header from the
  request. If present, base64-decode it; empty decoded string -> no
  pinning. No TTL handling - the extension assumes the client (or an
  intermediary) manages the lifetime of the header.
- `onUpdate` (`header.cc:9`): compare the actually-used host with the
  one that came in on the request. If it changed (or was absent),
  base64-encode the new upstream address and set the header on the
  response with `setCopy`. Returns whether the host changed so the
  filter can decide whether to emit the response header.
- Empty header name is rejected at config load with an
  `EnvoyException` (`header.cc:24`).

## Key decision points
- `header.cc:12` - same `host_changed` short-circuit as cookie: the
  header is only emitted when pinning actually shifted.
- `header.h:52` - empty decoded value treated as "no pinning" so a
  bad/forged header cannot drop the request.
- `header.cc:24` - required `name` validation happens once, at config
  load.

## Configuration
- `name` (required) - header used to carry the encoded upstream address
  both directions.

## Stats / errors
No dedicated stats. Only validation error is the empty `name` at config
load.
