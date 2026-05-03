# OAuth2 Filter (`envoy.filters.http.oauth2`)

Implements the OAuth 2.0 **authorization code** flow (with optional OpenID
Connect) in front of upstream services, so the application doesn't have to
handle login itself. Authenticated session state is carried in signed cookies.

Proto: `envoy.extensions.filters.http.oauth2.v3.OAuth2`.

## The flow

1. Unauthenticated request → filter redirects the user to the IdP
   `authorization_endpoint` with PKCE parameters.
2. IdP redirects back to the configured `redirect_uri` (matched by
   `redirect_path_matcher`) with an authorization `code`.
3. Filter exchanges the code at `token_endpoint` (`BASIC_AUTH`,
   `TLS_CLIENT_AUTH`, or `URL_ENCODED_BODY` auth style) for access / id /
   refresh tokens.
4. Filter stores the tokens in HMAC-signed, HttpOnly cookies and redirects the
   user back to the original URL (`finishGetAccessTokenFlow`, line 1053).
5. On subsequent requests, the cookies are validated; if the access token is
   expired and `use_refresh_token=true`, the refresh flow runs transparently
   (`finishRefreshAccessTokenFlow`).

`signout_path` clears the cookies and optionally redirects to
`end_session_endpoint`.

## Cookies

Configurable `CookieNames`: `bearer_token`, `oauth_hmac`, `oauth_expires`,
`id_token`, `refresh_token`, `oauth_nonce`, `code_verifier`.

- `SameSite` ∈ {Strict, Lax, None}.
- Optional `domain`, `Partitioned` attribute.
- Integrity: SHA-256 HMAC over `{domain, expires, bearer, id, refresh}` with
  `token_secret`, validated by `OAuth2CookieValidator`
  (`oauth.cc:196–220`). Both base64 and hex-base64 formats accepted for
  backward compatibility.
- Tokens themselves are AES-256-CBC encrypted with random IVs.

## Pass-through / opt-out

- `pass_through_matchers` (line 199) — matching requests bypass OAuth (use
  for health checks, machine-to-machine calls).
- `allow_failed_matchers` — matching requests proceed even with missing or
  expired credentials (useful for graceful degradation).
- `auth_scopes`, `resources` — sent with the authorization request.

## Entry points

- `decodeHeaders()` (`oauth.cc:528`) — the main orchestrator: validate the
  cookie, handle an OAuth callback, trigger a refresh, or redirect to the
  IdP.
- `finishGetAccessTokenFlow()` / `finishRefreshAccessTokenFlow()` — token
  exchange completion handlers.

## Stats

`oauth_unauthorized_rq`, `oauth_failure`, `oauth_success`,
`oauth_refreshtoken_success`, `oauth_refreshtoken_failure`,
`oauth_passthrough` (`oauth.h:77–84`).

## Files

- `oauth.{h,cc}` — filter, cookie validator, token client.
- `config.{h,cc}` — factory.
- `oauth_client.{h,cc}` — HTTP client for the token endpoint.
