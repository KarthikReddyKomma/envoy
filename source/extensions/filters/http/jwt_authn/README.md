# JWT Authentication Filter (`envoy.filters.http.jwt_authn`)

Verifies JWT tokens: signature against a JWKS, plus `iss` / `aud` / `sub` /
`exp` / `nbf` / lifetime / subject / clock-skew checks. Successful verification
optionally forwards the payload to upstream and clears the route cache.

Proto: `envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication`.

## Configuration shape

- **`providers`** — named JWT sources. Each provider declares issuer, audiences,
  subject matcher, local or remote JWKS, `clock_skew_seconds`,
  `forward_payload_header`, `claim_to_headers`, `payload_in_metadata`,
  `header_in_metadata`.
- **`requirement_map`** — named `JwtRequirement`s reusable across rules.
- **`rules`** — ordered list of `{route matcher, requirement}` pairs. Matches
  on prefix / path / safe-regex / `path_separated_prefix` / CONNECT.
- **`filter_state_rules`** — pick a requirement dynamically from a filter-state
  value (e.g., based on an earlier filter decision).

## Token extraction (`extractor.cc:259–318`)

Candidate sources, in order:

1. **Headers** — `Authorization: Bearer …` by default, or a configured
   `{name, value_prefix}`. Multiple instances of a header are all considered.
2. **Query parameters** — `access_token` by default, or a configured key.
3. **Cookies** — configured cookie names, parsed from the `Cookie` header.

Each candidate becomes a `JwtLocation` (token + which issuers are acceptable
here). Matched tokens can be sanitised out of the forwarded request.

## JWKS fetching

- **Local** — inline JWKS loaded into the per-thread `JwksCache`
  (`jwks_cache.cc:67–78`).
- **Remote** — `JwksAsyncFetcher` (`jwks_async_fetcher.cc:82–91`) fetches over
  HTTP with `cache_duration` (default 600 s); refetches 5 s before expiry, 1 s
  after failure. New JWKS is published atomically to all worker threads
  (`jwks_cache.cc:187–194`).
- **JWT cache** — LRU of verified tokens (default 100 entries, 4 KiB per token,
  `jwt_cache.cc`). Hit skips signature verification until token expiry.

## Verifier semantics (`verifier.cc:224–462`)

- **`requires_any`** — first success wins; aggregates
  `JwtMissed` / `JwtUnknownIssuer` errors (lines 282–320).
- **`requires_all`** — short-circuits on first failure (lines 340–350).
- **`allow_missing`** — `JwtMissed` is treated as OK (`verifier.cc:314`,
  `authenticator.cc:463`).
- **`allow_missing_or_failed`** — any non-OK status is treated as OK
  (`verifier.cc:313`).
- **`extract_only_without_validation`** — parse claims, skip signature
  (`verifier.cc:371–412`).

## Authenticator flow (`authenticator.cc`)

1. Extract token(s).
2. Whitelist issuer (line 186); check time (212), audience (219), subject
   matcher (225), lifetime (236).
3. JWT-cache lookup (lines 161–169). Hit → skip signature.
4. On miss: parse JWT, ensure JWKS is available (fetch if remote), call
   `verifyKey()` (line 308).
5. On success (`handleGoodJwt()`, lines 357–399):
   - Base64url-encode the payload into `forward_payload_header` (361–368).
   - Map claims → headers (373–375).
   - Write `payload_in_metadata` / `header_in_metadata` (dynamic metadata,
     namespace `envoy.filters.http.jwt_authn`).
   - If headers or metadata changed, `clearRouteCache()` (376, → `filter.cc:111`).
   - Insert into the JWT cache (394–396).

## Filter entry points (`filter.cc`)

- `decodeHeaders()` (line 52)
  - CORS preflight bypass (58).
  - Look up per-route config + verifier by route / header matcher (69–86).
  - Build a `ContextImpl` and call `verifier.verify(ctx)` (94–95). Returns
    `StopIteration` until the verifier is done (may block for remote JWKS).
- `onComplete()` (lines 113–149)
  - Success → `continueDecoding()`.
  - Failure → local reply with 401 *Unauthorized* or 403 *Forbidden*
    depending on the status.

## Stats

Per-provider counters: `jwt_cache_hit`, `jwt_cache_miss`, `jwks_fetch_success`,
`jwks_fetch_failed`, plus the standard JWT error codes (missing issuer, bad
audience, expired, bad signature, etc.). Defined in `stats.h`.

## Files

- `filter.{h,cc}`, `filter_config.{h,cc}`, `filter_factory.{h,cc}`
- `authenticator.{h,cc}` — verifies a single JWT in isolation.
- `verifier.{h,cc}` — composes multiple authenticators (any/all/missing).
- `extractor.{h,cc}` — where tokens come from.
- `jwks_cache.{h,cc}`, `jwks_async_fetcher.{h,cc}`, `jwt_cache.{h,cc}`
- `matcher.{h,cc}` — per-route rule → requirement lookup.
