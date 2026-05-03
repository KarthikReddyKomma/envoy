# OAuth2 Credential Injector

A `CredentialInjector` plugin consumed by the `credential_injector` HTTP
filter (`source/extensions/filters/http/credential_injector`). It performs
an OAuth2 client-credentials grant against a configured token endpoint,
caches the bearer token in thread-local storage, refreshes it before
expiry, and injects `Authorization: Bearer <token>` into outgoing
requests.

Proto: `envoy.extensions.http.injected_credentials.oauth2.v3.OAuth2`.

## Files
- `client_credentials_impl.h/cc` -
  `OAuth2ClientCredentialTokenInjector` (the actual injector).
- `token_provider.h/cc` - `TokenProvider` that drives the async token
  fetch loop and exposes the cached token as a `SecretReader`.
- `oauth_client.h/cc` - HTTP client that formats the
  `POST /token` request and parses the JSON response.
- `oauth.h` - shared `FilterCallbacks` / `FailureReason` types.
- `oauth_response.proto` - wire-format of the token endpoint response.
- `config.h/cc` - factory that wires together secret reader, token
  provider, and injector.

## Interface
- Injector implements
  `Extensions::Http::InjectedCredentials::Common::CredentialInjector`.
- `TokenProvider` implements `Common::SecretReader` (so it plugs directly
  into the injector as the "credential source") and `FilterCallbacks` (so
  `OAuth2ClientImpl` can report success/failure).
- Factory is a
  `Common::CredentialInjectorFactoryBase<OAuth2>`, registered via
  `REGISTER_FACTORY`.

## Logic
- `config.cc` builds the graph on construction:
  1. SDS-loaded client secret -> `SDSSecretReader`.
  2. `TokenProvider` holds the client secret reader plus proto config,
     owns an `OAuth2ClientImpl` pointed at `token_endpoint`, and fires
     an initial `asyncGetAccessToken` immediately.
  3. Injector is handed the `TokenProvider` as its `SecretReader`.
- `TokenProvider::asyncGetAccessToken`
  (`token_provider.cc:72`) issues a `grant_type=client_credentials` POST
  with the configured `client_id`, the current client secret,
  space-joined `scopes`, and any `endpoint_params`. If the cluster is
  missing or the client secret is empty, it schedules a retry via a
  dispatcher timer (default 2s, configurable via
  `token_fetch_retry_interval`).
- `onGetAccessTokenSuccess` stores `"Bearer " + access_token` into a
  thread-local object so `credential()` is a lock-free read, then arms a
  refresh timer at `expires_in / 2`.
- `onGetAccessTokenFailure` increments the matching stat and retries
  unless `BadToken` (malformed response, not retryable).
- The injector's `inject` mirrors the generic one: honors
  `overwrite` on the existing `Authorization` header, fails with
  `NotFoundError` when the token is empty, otherwise sets
  `Authorization` by reference to the TLS-cached value.

## Key decision points
- `token_provider.cc:55` - retry interval defaults to 2s, user-tunable.
- `token_provider.cc:60-63` - TLS slot initialized to an empty token so
  that `inject` has a defined empty-state to check.
- `token_provider.cc:77` - no fetch is attempted when the client secret
  is empty (fixes a restart-order race with SDS).
- `token_provider.cc:117-119` - refresh timer uses `expires_in / 2` to
  give ample headroom.
- `token_provider.cc:131` - `BadToken` is classified non-retryable.
- `client_credentials_impl.cc:22` - the injector sets the header by
  reference (TLS-owned string) rather than copy.

## Configuration
- `token_endpoint` - `HttpUri`-style endpoint, cluster resolved at fetch
  time.
- `client_credentials.client_id` / `.client_secret` - client id plus an
  SDS-backed secret.
- `scopes` - optional scopes, space-joined.
- `endpoint_params` - extra form fields sent with the token request.
- `token_fetch_retry_interval` - retry backoff on transient failure.

## Stats / errors
Under `<stats_prefix>oauth2.` (`token_provider.h:24`):
- `token_requested`, `token_fetched`.
- `token_fetch_failed_on_client_secret`,
  `token_fetch_failed_on_cluster_not_found`,
  `token_fetch_failed_on_stream_reset`,
  `token_fetch_failed_on_bad_token`,
  `token_fetch_failed_on_bad_response_code`.

Errors from `inject` are `AlreadyExistsError` or `NotFoundError` (same
semantics as the generic injector).
