# GCP Authentication (`envoy.filters.http.gcp_authn`)

Calls the GCP metadata server to mint an ID token for a configured audience (read from the matched cluster's typed filter metadata) and injects the token into the outgoing request's `Authorization` (or user-defined) header. An optional thread-local LRU `TokenCache` keyed by audience avoids a metadata-server round-trip on every request.

Proto: `envoy.extensions.filters.http.gcp_authn.v3.GcpAuthnFilterConfig` and the per-cluster `Audience` metadata message.

## Files
- `gcp_authn_filter.h/cc` — `GcpAuthnFilter` (decoder path + async callback on token fetch).
- `filter_config.h/cc` — `GcpAuthnFilterFactory`, builds shared `FilterConfig`, optional `TokenCache`, validates retry policy. `REGISTER_FACTORY` at `filter_config.cc:47`.
- `gcp_authn_impl.h/cc` — `GcpAuthnClient` (async HTTP client to the metadata endpoint), `RequestCallbacks` interface, URL template (`UrlString`).
- `token_cache.h` — `TokenCacheImpl<JwtToken>` (LRU over parsed JWTs) wrapped in TLS (`TokenCache::tls`).

## Lifecycle
`GcpAuthnFilter` extends `Http::PassThroughFilter` and `RequestCallbacks` (`gcp_authn_filter.h:38-40`). The factory builds one `FilterConfigSharedPtr` (the raw proto, shared), plus an optional `TokenCache` shared across workers (`filter_config.cc:21-23`); each stream gets a fresh filter via `addStreamFilter` (`filter_config.cc:38`). Each filter owns a `GcpAuthnClient` constructed from the config + `FactoryContext` (`gcp_authn_filter.h:51`).

Overridden callbacks:
- `setDecoderFilterCallbacks` (`gcp_authn_filter.cc:95-97`): caches the decoder callbacks.
- `decodeHeaders` (`gcp_authn_filter.cc:35-93`):
  1. No route / no route entry → `Continue` (`gcp_authn_filter.cc:36-40`).
  2. State becomes `Calling`, `initiating_call_ = true`.
  3. Look up the target cluster via `getThreadLocalCluster` and read the `gcp_authn` entry from its `typed_filter_metadata`; unpack into an `Audience` message (`gcp_authn_filter.cc:45-58`).
  4. If `audience_str_` is empty → increment `retrieve_audience_failed_`, mark state `Complete`, and continue the chain without a token (`gcp_authn_filter.cc:82-87`).
  5. Otherwise, if a cache is configured, probe `jwt_token_cache_->lookUp(audience)`. Cache hit → call `addTokenToRequest` and `Continue` immediately (`gcp_authn_filter.cc:61-69`).
  6. Cache miss → stash `&hdrs` in `request_header_map_`, substitute `[AUDIENCE]` in `UrlString` (`gcp_authn_filter.cc:79`), and fire `client_->fetchToken(*this, buildRequest(final_url))`. Clear `initiating_call_`.
  7. Return `StopAllIterationAndWatermark` to pause iteration and apply backpressure while the metadata fetch is in flight; return `Continue` on the fast-path (empty audience) case (`gcp_authn_filter.cc:91-92`).
- `onComplete(response)` (`gcp_authn_filter.cc:99-125`) — the `RequestCallbacks` entry point:
  - Marks `state_ = Complete`. If the call was started and completed asynchronously (i.e. `!initiating_call_`), installs the token and resumes the chain.
  - `addTokenToRequest` on `request_header_map_` when non-null (`gcp_authn_filter.cc:105-110`). Default header is `Authorization: Bearer <token>`; the proto's `token_header` overrides name and `value_prefix` (`gcp_authn_filter.cc:18-27`).
  - Parses the token with `JwtVerify::Jwt::parseFromString`; on success and when a cache is configured, inserts into the TLS cache (`gcp_authn_filter.cc:112-121`).
  - Calls `decoder_callbacks_->continueDecoding()` to resume filter iteration (`gcp_authn_filter.cc:123`).
- `onDestroy` (`gcp_authn_filter.cc:127-132`): if the filter is still in the `Calling` state, `client_->cancel()` to abort the in-flight metadata fetch before the stream tears down.

No encode-side overrides; the filter only mutates the request.

## Decision / logic
- State machine is the three-valued `State { NotStarted, Calling, Complete }` (`gcp_authn_filter.h:45`).
- `initiating_call_` disambiguates the synchronous (cache hit or no audience) path from the async completion. `onComplete` is a no-op w.r.t. chain continuation when `initiating_call_` is still true (`gcp_authn_filter.cc:101-122`).
- `StopAllIterationAndWatermark` is chosen over `StopIteration` so the decoder applies watermarks while waiting for the metadata server (`gcp_authn_filter.cc:92`).
- Audience is a per-cluster property, not per-route: it is read from `cluster->info()->metadata().typed_filter_metadata()` (`gcp_authn_filter.cc:51-57`).
- The retry policy field on the proto is invalid-validated by `Http::Utility::validateCoreRetryPolicy` at factory time (`filter_config.cc:27-29`).

## Configuration
- `http_uri` — metadata endpoint config used by `GcpAuthnClient`.
- `retry_policy` — validated at factory construction (`filter_config.cc:27-29`).
- `cache_config` — when `cache_size > 0`, creates a `TokenCache` with thread-local storage (`filter_config.cc:22-23`).
- `token_header` — optional `{name, value_prefix}`; empty means default `Authorization: Bearer ...` (`gcp_authn_filter.cc:19-27`).
- Cluster typed metadata under key `envoy.filters.http.gcp_authn` must carry an `Audience{ url }` message; missing/unparseable → filter no-ops and increments `retrieve_audience_failed_`.

No per-route filter config is defined; behavior is driven entirely by the listener-level config plus cluster metadata.

## Stats
Counter(s) under `<stats_prefix>` (see `ALL_GCP_AUTHN_FILTER_STATS`, `gcp_authn_filter.h:26`):
- `retrieve_audience_failed` — incremented whenever the matched cluster has no `gcp_authn` audience metadata (`gcp_authn_filter.cc:85`).
