# Envelope Stateful Session

A `SessionStateFactory` consumed by the `stateful_session` HTTP filter.
Unlike the cookie / header strategies, the envelope strategy does not
*own* its transport header - it piggy-backs on an existing
upstream-controlled header (e.g. the upstream's own session ID header) by
wrapping its value with an additional base64-encoded upstream address.
The upstream sees its original header value unchanged; Envoy uses the
wrapper to pin subsequent requests to the same backend.

Proto: `envoy.extensions.http.stateful_session.envelope.v3.EnvelopeSessionState`.

## Files
- `envelope.h/cc` - factory + `SessionStateImpl`.
- `config.h/cc` - factory registration.

## Interface
- Implements `Envoy::Http::SessionStateFactory`.
- `SessionState` implements `upstreamAddress()` and `onUpdate()`
  (identical contract to the cookie / header extensions).

## Logic
- Request-side `parseAddress` (`envelope.cc:36`):
  - Reads the configured header. If missing -> `nullopt` (no pinning).
  - Splits the value on `;` with `absl::SkipEmpty`. `parts[0]` is the
    base64-encoded upstream address that Envoy inserted on the previous
    response.
  - Remaining parts are scanned for the `UV:` prefix; the piece after
    `UV:` is the base64-encoded original upstream value that must be
    restored before the request is forwarded.
  - If the `UV:` part is missing or empty, the call logs at `info` and
    returns `nullopt` (so pinning is skipped but the request is not
    dropped).
  - Otherwise, the configured header is rewritten to the decoded
    original value (so the upstream sees its own value) and the decoded
    upstream address is returned for routing.
- Response-side `onUpdate` (`envelope.cc:13`):
  - Reads the upstream's outgoing value for the configured header. If
    absent or duplicated, returns `false` (envelope extension refuses
    to guess).
  - Builds `<base64(host_address)>;UV:<base64(upstream_value)>` and
    writes it back, always overwriting. The `host_changed` flag is
    returned based on whether the upstream address differs from the
    one the request arrived with.

## Key decision points
- `envelope.cc:11` - `UV:` prefix is the fixed marker for the original
  upstream value.
- `envelope.cc:17-21` - onUpdate bails if the header is missing or
  occurs multiple times; the envelope format presumes a single
  upstream-owned header value.
- `envelope.cc:51` - skip `parts[0]` when searching for `UV:` so the
  address portion cannot be misinterpreted.
- `envelope.cc:60` - missing / empty `UV:` is a soft failure: pinning is
  disabled but the request continues.
- `envelope.cc:64` - the header is rewritten in place before the request
  leaves Envoy so upstream observability is unaffected.

## Configuration
- `header.name` - header that the upstream owns and that Envoy wraps.

## Stats / errors
No stats. All failure modes log at `trace` / `info` and fall back to the
"no pinning" path.
