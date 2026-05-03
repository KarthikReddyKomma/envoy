# API Key Auth (`envoy.filters.http.api_key_auth`)

Decoder-only filter that authenticates requests by matching an API key (from a header, query parameter, or cookie) against a configured key-to-client-id map. Supports per-route credential overrides, per-route authorization (`allowed_clients`), and optional client-id forwarding with optional stripping of the credential.

Proto: `envoy.extensions.filters.http.api_key_auth.v3.ApiKeyAuth` (plus `ApiKeyAuthPerRoute`, `KeySource`, `Forwarding`).

## Files
- `api_key_auth.h/cc` — filter, per-route config, shared `ApiKeyAuthConfig` struct, `KeySources`, `Forwarding`, `FilterConfig`, stats.
- `config.h/cc` — `ApiKeyAuthFilterFactory`, registered as `envoy.filters.http.api_key_auth`.

## Lifecycle
Extends `Http::PassThroughDecoderFilter` (`api_key_auth.h:232`). Only `decodeHeaders` is overridden.

- `decodeHeaders(headers, end_stream)` (`api_key_auth.cc:116`) — the entire auth flow runs here.
  1. Resolves per-route config with `Http::Utility::resolveMostSpecificPerFilterConfig<RouteConfig>` (`api_key_auth.cc:117-118`).
  2. Seeds `credentials`, `key_sources`, `forwarding` from the listener-level `FilterConfig`, then overlays any route overrides that are present (`api_key_auth.cc:120-139`). Each field is overridden independently.
  3. Validates that after merging both `key_sources` and `credentials` are present; otherwise `onDenied(401, "missing_key_sources" / "missing_credentials")` (`api_key_auth.cc:141-148`).
  4. Extracts the key via `key_sources->getKey(headers, buffer)`. Empty key -> `onDenied(401, "missing_api_key")` (`api_key_auth.cc:150-155`).
  5. Looks up the key in the credentials map. Miss -> `onDenied(401, "unkonwn_api_key")` (sic) (`api_key_auth.cc:157-160`).
  6. If there is a route config, checks `allowClient(client_id)` (empty allowed_clients = allow-all) -> `onDenied(403, "client_not_allowed")` on fail (`api_key_auth.cc:164-168`).
  7. If `forwarding` is configured: sets the client id into the configured request header and, if `hide_credentials` is true, calls `key_sources->removeKey(headers)` to strip the credential from whichever source it came from (`api_key_auth.cc:170-183`).
  8. On success bumps `allowed` counter and returns `Continue` (`api_key_auth.cc:185-186`).

`onDenied` (`api_key_auth.cc:189`) bumps `unauthorized` or `forbidden` depending on status and calls `sendLocalReply(code, body, nullptr, nullopt, response_code_details)` then `StopIteration`.

## Decision / logic
- Key extraction (`KeySources::Source::getKey`, `api_key_auth.cc:44-71`):
  - Header source: if the header value starts with `Bearer `, strip the 7-char prefix (`api_key_auth.cc:50-52`).
  - Query source: parse-and-decode the query string, take the first value of the named param.
  - Cookie source: `Http::Utility::parseCookieValue`.
  - Result string view references either the header value or the caller-provided `buffer` (query/cookie write into the buffer).
- Sources are tried in order and the first non-empty key wins (`api_key_auth.cc:73-81`).
- Key removal for `hide_credentials=true` mirrors whichever source was configured: header `remove`, query re-serialize path without the param, or `removeCookieValue` (`api_key_auth.cc:89-102`).
- Merge precedence: per-route field overrides listener field only when route field is present; absent route fields inherit (`api_key_auth.cc:126-139`).
- Duplicate credential keys in proto are rejected at config-build time (`api_key_auth.h:116-120`).

## Configuration
- `credentials[]` — `{key, client}` pairs; loaded into a `flat_hash_map<string, string>`. Duplicates are an `InvalidArgument` error.
- `key_sources[]` — ordered list; each has one of `header`, `query`, or `cookie`. If none are set the source is an `InvalidArgument` (`api_key_auth.cc:31`).
- `forwarding.header` / `forwarding.hide_credentials` — optional client-id forwarding header and credential stripping.
- Per-route (`ApiKeyAuthPerRoute`, `RouteConfig`): may override `credentials`, `key_sources`, `forwarding`, and may set `allowed_clients` for authorization. Registered through `createRouteSpecificFilterConfigTyped` (`config.cc:25-35`).

## Stats
Prefix `<stats_prefix>api_key_auth.` (`api_key_auth.cc:112`):

- `allowed` — authenticated and authorized.
- `unauthorized` — any 401 denial path.
- `forbidden` — 403 `client_not_allowed`.

## Factory
`ApiKeyAuthFilterFactory::createFilterFactoryFromProtoTyped` (`config.cc:11`) builds the shared `FilterConfig` (propagating construction errors through `absl::Status`) and returns a callback that calls `addStreamDecoderFilter` — this filter does not touch the encoder side. `createRouteSpecificFilterConfigTyped` (`config.cc:25`) builds the `RouteConfig` shared pointer. Registration via `REGISTER_FACTORY(ApiKeyAuthFilterFactory, NamedHttpFilterConfigFactory)` (`config.cc:37`).
