# HTTP Filters Common Infrastructure

Not a filter. This directory is a grab-bag of header-only helpers and small utility libraries that every other `source/extensions/filters/http/*` filter depends on. There is no proto and no `REGISTER_FACTORY` here.

## Files
- `factory_base.h` — templated base classes that remove boilerplate from each filter's config-factory class.
- `pass_through_filter.h` — base classes that implement `StreamDecoderFilter` / `StreamEncoderFilter` / `StreamFilter` as pure pass-through so subclasses only override what they need.
- `stream_rate_limiter.h/cc` — generic byte-rate limiter used by `bandwidth_limit` and `fault`.
- `ratelimit_headers.h` — singleton for the `x-ratelimit-*` header names.
- `jwks_fetcher.h/cc` — `JwksFetcher` abstract interface + factory for fetching remote JWKS; used by `jwt_authn`.

## `factory_base.h` — factory templates
Everything extends `CommonFactoryBase<ConfigProto, RouteConfigProto = ConfigProto>` (`factory_base.h:15`) which provides:
- `createEmptyConfigProto` / `createEmptyRouteConfigProto` returning fresh `ConfigProto` / `RouteConfigProto` instances.
- `createRouteSpecificFilterConfig` — downcasts the incoming `Protobuf::Message` to `RouteConfigProto` via `MessageUtil::downcastAndValidate` and forwards to a subclass-provided `createRouteSpecificFilterConfigTyped` (default returns `nullptr`).
- `name()` / `isTerminalFilterByProto` boilerplate.

Three concrete specializations:
- `FactoryBase` (`factory_base.h:62`) — downstream-only filters. Subclass implements `createFilterFactoryFromProtoTyped` (pure). Also exposes `createFilterFactoryFromProtoWithServerContextTyped` for contexts that only have a `ServerFactoryContext`; default throws.
- `ExceptionFreeFactoryBase` (`factory_base.h:100`) — same shape but `createFilterFactoryFromProtoTyped` returns `absl::StatusOr<FilterFactoryCb>` so the filter can report config errors without throwing.
- `DualFactoryBase` (`factory_base.h:120`) — for filters usable in both downstream and upstream filter chains. Implements both `HttpFilterConfigFactoryBase` paths and packs the differing context types into a `DualInfo { init_manager, scope, is_upstream }` struct.

Used by basically every filter: e.g. `basic_auth` extends `FactoryBase<BasicAuth, BasicAuthPerRoute>`, `bandwidth_limit` extends `ExceptionFreeFactoryBase<BandwidthLimit>`, `composite` extends `DualFactoryBase<Composite>`.

## `pass_through_filter.h` — default filter skeletons
Three classes:
- `Http::PassThroughDecoderFilter` (`pass_through_filter.h:9`) — implements `StreamDecoderFilter`. All overrides return `Continue`; `setDecoderFilterCallbacks` caches the pointer into a protected `decoder_callbacks_` member. `onDestroy` is a no-op.
- `Http::PassThroughEncoderFilter` (`pass_through_filter.h:33`) — symmetric for encoding; caches `encoder_callbacks_`.
- `Http::PassThroughFilter` (`pass_through_filter.h:63`) — multiple-inherits from `StreamFilter` + both pass-throughs, so subclasses can override only the methods they care about. Note that this creates diamond inheritance via `StreamFilterBase`; callers doing `onDestroy` dispatch on the composite filter sometimes need explicit `static_cast<Http::StreamDecoderFilter&>(...)` (see `composite/filter.cc:127`).

Used by `basic_auth`, `cdn_loop`, `cache_v2`, `composite`, and many more. `bandwidth_limit` intentionally does **not** use these — it inherits `StreamFilter` directly because it needs non-default behavior for every callback.

## `stream_rate_limiter.h/cc` — byte-rate throttle
`Common::StreamRateLimiter` (`stream_rate_limiter.h:31`) is the core of any filter doing byte-rate shaping. Construction takes:
- `max_buffered_data`, `pause_data_cb`, `resume_data_cb` — high/low-watermark hooks for the owning filter.
- `write_data_cb(Buffer, bool end_stream)` — how to inject bytes back into the filter chain (e.g. `injectDecodedDataToFilterChain`).
- `continue_cb` — resumes paused trailers once body is drained.
- `write_stats_cb(bytes_sent, bytes_buffered, delay)` — per-release accounting callback; `bandwidth_limit` uses this to increment `*_enforced_` when any buffering occurred.
- `dispatcher`, `scope`, `token_bucket`, `fill_interval`.

Key methods: `writeData(buf, end_stream, trailer_added=false)` feeds bytes in, `onTrailers()` returns `true` if data is still buffered, `destroy()` cancels the timer. A helper `simpleTokenBucket(limit_kbps, time_source)` and `kiloBytesToBytes` constant are also provided.

The timer calls `onTokenTimer()` on each `fill_interval`, pulling tokens from the bucket and dispatching up to that many buffered bytes through `write_data_cb_`.

## `ratelimit_headers.h` — shared header names
`XRateLimitHeaderValues` (`ratelimit_headers.h:16`) is a `ConstSingleton` holding lowercase-canonical `x-ratelimit-limit`, `x-ratelimit-remaining`, and `x-ratelimit-reset`. Plus two `absl::string_view` constants `QuotaPolicyWindow = "w"` and `QuotaPolicyName = "name"` for policy-encoded header values. Used by rate-limit filters to emit response headers.

## `jwks_fetcher.h/cc` — async JWKS fetcher
Abstract interface `Common::JwksFetcher` (`jwks_fetcher.h:23`) with:
- Nested `JwksReceiver` with `onJwksSuccess(JwksPtr&&)` / `onJwksError(Failure::{Network,InvalidJwks})`.
- `fetch(parent_span, receiver)` — must have at most one outstanding fetch.
- `cancel()` — abort in-flight.
- `static create(cm, retry_policy, RemoteJwks)` — factory returning the default HTTP implementation backed by a cluster manager.

Consumed by `jwt_authn` to resolve `RemoteJwks` configurations into verified keysets.

## How this folder fits in
- Every filter factory inherits one of the `factory_base.h` templates to register itself with Envoy.
- Filters that don't need encode-side or decode-side behavior inherit the matching `PassThrough*` class.
- Any filter that wants to shape bytes reuses `StreamRateLimiter` rather than rolling its own token bucket.
- There are no stats, no registrations, and no runtime behavior here — only link-time utilities.
