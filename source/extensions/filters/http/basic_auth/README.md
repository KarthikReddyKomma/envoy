# Basic Auth Filter (`envoy.filters.http.basic_auth`)

Enforces HTTP Basic authentication on requests. The filter parses the `Authorization: Basic <b64>` header (or a configurable alternative), SHA-1 hashes the presented password, looks the user up in a htpasswd-sourced map, and either lets the request continue, optionally forwarding the authenticated username in a header, or replies `401 Unauthorized`.

Proto: `envoy.extensions.filters.http.basic_auth.v3.BasicAuth` (and `BasicAuthPerRoute` for overrides).

## Files
- `basic_auth_filter.h/cc` — `BasicAuthFilter` (a `PassThroughDecoderFilter`), `FilterConfig`, `FilterConfigPerRoute`, `UserMap`, and stats.
- `config.h/cc` — `BasicAuthFilterFactory`, the htpasswd parser, and factory registration.

## Lifecycle
`BasicAuthFilter` extends `Http::PassThroughDecoderFilter` (`basic_auth_filter.h:82`), so only the decode side is active. Encode callbacks and `onDestroy` keep their pass-through defaults.

Overridden method:
- `decodeHeaders(RequestHeaderMap&, bool)` (`basic_auth_filter.cc:42`): the entire check lives here. Returns `StopIteration` when sending a 401, `Continue` otherwise.

## Decision / logic
Performed in order inside `decodeHeaders`:
1. Resolve the user map (`basic_auth_filter.cc:43-48`): per-route `FilterConfigPerRoute` (looked up via `resolveMostSpecificPerFilterConfig<FilterConfigPerRoute>`) overrides the filter-level `users_` map when present.
2. Pick which header to read (`basic_auth_filter.cc:50-55`): `authentication_header_` if non-empty, else the standard `Authorization` header.
3. If the header is missing → `onDenied("... Missing username and password.", "no_credential_for_basic_auth")` (`basic_auth_filter.cc:57`).
4. Require the value to start with `"Basic "` (`basic_auth_filter.cc:64`). Otherwise `"invalid_scheme_for_basic_auth"`.
5. Base64-decode the tail without padding (`basic_auth_filter.cc:70-71`).
6. Find the `:` separating `username:password` (`basic_auth_filter.cc:74`). No colon → `"invalid_format_for_basic_auth"`.
7. `validateUser(*users, username, password)` (`basic_auth_filter.cc:97`): look up the username, compare `computeSHA1(password)` (base64-encoded SHA-1, matching `{SHA}` htpasswd format) against the stored hash (`basic_auth_filter.cc:21-29`). Mismatch/unknown user → `"invalid_credential_for_basic_auth"`.
8. On success, if `forward_username_header_` is non-empty, set that header to the decoded username (`basic_auth_filter.cc:89-91`), bump `allowed_`, and return `Continue`.

`onDenied` (`basic_auth_filter.cc:107`): bumps `denied_`, calls `sendLocalReply(Http::Code::Unauthorized, ...)` with a `WWW-Authenticate: Basic realm="<uri>"` header built from the original request URI (truncated to `MaximumUriLength = 256`, `basic_auth_filter.cc:18`), and the supplied response-code-details string.

## Configuration
`FilterConfig` (`basic_auth_filter.h:45`) holds:
- `users_` — `UserMap` built by `readHtpasswd` in `config.cc:16` from the `users` DataSource. Accepts only `{SHA}`-prefixed hashes of exactly 28 base64 chars; rejects duplicates, empties, and bad lines (`config.cc:42-54`).
- `forward_username_header_` — when set, authenticated requests get the username copied into this lowercase header.
- `authentication_header_` — alternate header name to read credentials from.

Per-route: `FilterConfigPerRoute` (`basic_auth_filter.h:72`) owns only a `UserMap`. Built by `createRouteSpecificFilterConfigTyped` (`config.cc:91`) so a vhost/route/weighted-cluster can override the allowed-user set without changing headers.

## Stats
Prefix `<stats_prefix>basic_auth.` (`basic_auth_filter.cc:38`). From `ALL_BASIC_AUTH_STATS` (`basic_auth_filter.h:19`):
- `allowed` — requests that passed validation.
- `denied` — requests rejected for any reason (missing/invalid/bad creds).

## Factory
`BasicAuthFilterFactory` extends `Common::FactoryBase<BasicAuth, BasicAuthPerRoute>` (`config.h:14`). Three entry points:
- `createFilterFactoryFromProtoTyped` (`config.cc:64`): reads htpasswd via `Config::DataSource::read` and returns a callback that installs the filter with `addStreamDecoderFilter`.
- `createFilterFactoryFromProtoWithServerContextTyped` (`config.cc:78`): same but for contexts without a listener scope (e.g., for dynamic filter config).
- `createRouteSpecificFilterConfigTyped` (`config.cc:91`): builds per-route user maps.

Registered by `REGISTER_FACTORY(BasicAuthFilterFactory, NamedHttpFilterConfigFactory)` at `config.cc:100`.
