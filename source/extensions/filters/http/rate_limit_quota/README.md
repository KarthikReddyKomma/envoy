# Rate Limit Quota (`envoy.filters.http.rate_limit_quota`)

A decode-only HTTP filter implementing the Rate Limit Quota Service (RLQS) client. On each request it runs a match tree against the request headers to pick a bucket, looks the bucket up in a thread-local cache, applies the current quota strategy (blanket allow/deny or a token bucket), and updates usage counters that a main-thread `GlobalRateLimitClient` streams to the RLQS server. Cache misses for a bucket trigger an async `ReportQuotaUsage` and rely on a configurable `no_assignment_behavior` to decide whether to admit the initial request. Matches flagged with `keep_matching` are handled in a "preview" mode so operators can dry-run policy changes.

Proto: `envoy.extensions.filters.http.rate_limit_quota.v3.RateLimitQuotaFilterConfig` (and `RateLimitQuotaOverride` for route-level overrides).

## Files
- `config.h/cc` — `RateLimitQuotaFilterFactory`, TLS store lookup, matcher tree compilation. `REGISTER_FACTORY` at `config.cc:78`.
- `filter.h/cc` — `RateLimitQuotaFilter` (`PassThroughFilter` decoder), matcher evaluation, bucket processing.
- `global_client_impl.h/cc` — main-thread `GlobalRateLimitClient` that owns the bidi gRPC stream to RLQS and the cross-worker `GlobalTlsStores` (TLS stores keyed by `(config_hash_key, domain)`).
- `client_impl.h/cc` — `createLocalRateLimitClient`: per-filter façade over the shared global client.
- `quota_bucket_cache.h` — `CachedBucket` / `QuotaUsage` structs including `token_bucket_limiter`.
- `matcher.h/cc` — `RateLimitOnMatchAction` (the match-tree action type) and `BucketId` generation.
- `filter_persistence.h/cc` — helpers for persisting/restoring TLS state.

## Lifecycle
Installed as a stream filter with a freshly-created local client per stream (`config.cc:65-72`). The global client and matcher tree live at listener scope, shared through `GlobalTlsStores` (`config.cc:62`).

Overridden callbacks on `RateLimitQuotaFilter`:
- `decodeHeaders(RequestHeaderMap&, bool)` (`filter.cc:204-221`) — entire decision path.
- `onDestroy()` (`filter.cc:279-281`) — TODO: per-filter TLS cleanup not yet implemented.
- `setDecoderFilterCallbacks` (`filter.h:62-64`) — stashes callbacks.

No data/trailer overrides; buckets are evaluated strictly on headers (`filter.cc:237-238` TODO acknowledges this).

## Decision / logic
`decodeHeaders` pipeline:
- `filter.cc:208-218`: `requestMatching(headers)` returns an `absl::StatusOr<ActionConstSharedPtr>`. `NotFound` -> fail-open `Continue` (no usage reported).
- `filter.cc:220`: `recordBucketUsage(*match_result, /*is_preview_match=*/false)`.

`requestMatching` (`filter.cc:239-277`):
- `filter.cc:244-249`: lazy-build `data_ptr_` (`HttpMatchingDataImpl`) once per filter instance.
- `filter.cc:260-264`: `Matcher::evaluateMatch` with a `skipped_action` callback routing preview-mode matches to `handlePreviewMatch`.
- Translates match states into `Internal` / `NotFound` / `Internal` errors (`filter.cc:265-273`).

`handlePreviewMatch` (`filter.cc:223-235`): only the first preview hit per request is recorded (guarded by `first_skipped_match_`), calling `recordBucketUsage(skipped_action, true)` so preview usage lands under `preview_bucket` metadata rather than affecting the real decision.

`recordBucketUsage` (`filter.cc:110-202`):
- `filter.cc:115-123`: `match_action.generateBucketId(...)`; failure -> fail-open.
- `filter.cc:125-128`: hash the `BucketId` proto for cache keys.
- `filter.cc:130-137`: write the matched bucket fields to dynamic metadata under `envoy.extensions.http_filters.rate_limit_quota.bucket` (or `..preview_bucket` for previews).
- `filter.cc:143`: `client_->getBucket(bucket_id)` returns the cached entry if any.
- `filter.cc:144-147`: hit -> `processCachedBucket`.
- `filter.cc:151-163`: miss -> build `default_bucket_action`. If `no_assignment_behavior` is set, seed the bucket with its `fallback_rate_limit` and call `noAssignmentBehaviorShouldAllow` (`filter.cc:47-52`: only `DENY_ALL` blanket rule denies; everything else allows the first request). Otherwise default to `ALLOW_ALL`.
- `filter.cc:168-186`: compute `expiration_fallback_action` / `expiration_fallback_ttl` from `expired_assignment_behavior`.
- `filter.cc:190-194`: `client_->createBucket(...)` posts the new bucket to the main thread for streaming registration.
- `filter.cc:196-201`: preview and allow cases return `Continue`; otherwise `sendDenyResponse(..., ResponseFromCacheFilter)`.

`processCachedBucket` (`filter.cc:326-343`):
- `filter.cc:333-336`: if `shouldAllowRequest` is true, atomic-increment `num_requests_allowed` and `Continue`.
- `filter.cc:338-342`: else increment `num_requests_denied` and send the configured deny response.

`shouldAllowRequest` (`filter.cc:289-324`) — switches on `RateLimitStrategy`:
- `kBlanketRule`: `ALLOW_ALL` / `DENY_ALL`.
- `kTokenBucket`: `cached_bucket.token_bucket_limiter->consume(1, false)`.
- `kRequestsPerTimeUnit`: TODO, fails open with a warn log.
- `STRATEGY_NOT_SET`: logs an error-level "bug" message and fails open.

`sendDenyResponse` (`filter.cc:98-106`) uses `sendLocalReply` with `getDenyResponseCode`, `getResponseBodyText`, `addDenyResponseHeadersCb`, and `getGrpcStatus` (`filter.cc:55-96`) derived from the bucket's `DenyResponseSettings`, then sets the provided `CoreResponseFlag`.

## Configuration
`RateLimitQuotaFilterConfig`:
- `rlqs_server` (`core.v3.GrpcService`) — upstream RLQS endpoint; used as part of the TLS store key (`config.cc:40-63`).
- `domain` — logical scope used in the RLQS `domain` field.
- `bucket_matchers` — an xDS matcher tree compiled via `MatchTreeFactory<HttpMatchingData, RateLimitOnMatchActionContext>` (`config.cc:46-54`). The visitor is constructed with `setSupportKeepMatching(true)` so `keep_matching` becomes the preview mechanism.
- Each `RateLimitQuotaBucketSettings` action carries `reporting_interval`, `no_assignment_behavior`, `expired_assignment_behavior`, and `deny_response_settings`.

Route overrides: the factory declares `RateLimitQuotaOverride` as its second template argument (`config.h:20-21`), so per-route configs are accepted. The implementation in this folder consumes the override only through the matcher-tree machinery; the concrete override handling lives in the shared match action types.

## Stats
None are emitted in this folder. `decodeHeaders` leaves behind a TODO (`filter.cc:213`: "Add stats here ..."). Observability today comes through:
- Dynamic metadata namespaces `envoy.extensions.http_filters.rate_limit_quota.bucket` and `...preview_bucket` (`filter.cc:34-36`, `filter.cc:136-137`).
- `CoreResponseFlag::ResponseFromCacheFilter` when denied (`filter.cc:201`, `filter.cc:342`).
- Atomic per-bucket counters in `QuotaUsage.num_requests_allowed` / `num_requests_denied` reported to RLQS by the global client.

## Factory
`RateLimitQuotaFilterFactory` (`config.h:18`):
- `FactoryBase<RateLimitQuotaFilterConfig, RateLimitQuotaOverride>` with name `"envoy.filters.http.rate_limit_quota"` (`config.h:16`, `config.h:24`).
- `createFilterFactoryFromProtoTyped` (`config.cc:30-73`): compiles the matcher tree (throws on `visitor.errors()`), acquires/initializes a shared `TlsStore` via `GlobalTlsStores::getTlsStore`, and returns a lambda that constructs a per-stream `RateLimitQuotaFilter` with its own local client.
- `REGISTER_FACTORY(RateLimitQuotaFilterFactory, NamedHttpFilterConfigFactory)` at `config.cc:78`.
