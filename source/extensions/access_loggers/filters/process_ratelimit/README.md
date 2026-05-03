# Process Rate Limit Access Log Filter

A process-wide (shared across worker threads) token-bucket rate-limit filter for access logs. It lets operators cap how many log records a single Envoy process emits per second, independent of request volume on any one worker. The token bucket is fetched via xDS as an `envoy.type.v3.TokenBucket` resource so the rate can be tuned at runtime without a restart.

Proto: `envoy.extensions.access_loggers.filters.process_ratelimit.v3.ProcessRateLimitFilter` (see `api/envoy/extensions/access_loggers/filters/process_ratelimit/v3/process_ratelimit.proto`).

## Files
- `filter.h` / `filter.cc` - `ProcessRateLimitFilter` (implements `AccessLog::Filter`).
- `config.h` / `config.cc` - `ProcessRateLimitFilterFactory` registered as `AccessLog::ExtensionFilterFactory`.
- `provider_singleton.h` / `provider_singleton.cc` - `RateLimiterProviderSingleton`, `TokenBucketSubscription`, and `RateLimiterWrapper`. Singleton storage lives under the generic `Extensions::Filters::Common::LocalRateLimit` namespace so other extensions can reuse it.

## Interface
- `AccessLog::Filter::evaluate(const Formatter::Context&, const StreamInfo::StreamInfo&) const override` - returns `true` to log, `false` to drop (filter.cc:54).

## Flow
1. Config load: `ProcessRateLimitFilterFactory::createFilter` in `config.cc:15` translates the typed config and builds a `ProcessRateLimitFilter`.
2. Construction (`filter.cc:15`): grabs the main-thread dispatcher and calls `RateLimiterProviderSingleton::getRateLimiter(...)` with the `dynamic_config.resource_name` and `config_source`. A `setter` callback receives the `LocalRateLimiterImpl` once xDS delivers the token bucket.
3. xDS delivery: `TokenBucketSubscription::onConfigUpdate` (in `provider_singleton.cc`) creates or updates a shared `LocalRateLimiterImpl` and fans the pointer out to every registered setter. Filters sharing the same `resource_name` share one token bucket.
4. Per-record check: `evaluate` calls `limiter->requestAllowed({})` (filter.cc:60). On success it bumps `stats_.allowed_`; on failure it bumps `stats_.denied_` and the record is suppressed.
5. Teardown (`filter.cc:43`): sets `cancel_cb_` atomically so any in-flight setter becomes a no-op, then posts a main-thread task to `removeSetter(setter_key_)` so the subscription drops its reference.

## Key decision points
- `filter.cc:35` - rejects configs missing `dynamic_config` (throws `EnvoyException`).
- `filter.cc:60` - `requestAllowed({})` is the single gate controlling `allowed` vs `denied`.
- `filter.cc:56` - `ENVOY_BUG` if the limiter hasn't been set by init; the `init_target_` in `RateLimiterWrapper` is meant to block logging until it is.
- `provider_singleton.h:127` - fallback limiter is `AlwaysDenyLocalRateLimiter`; when a resource is removed via xDS the subscription swaps to it so no pre-existing filters keep using the stale bucket.
- `provider_singleton.h:109` - `ThreadLocal::TypedSlot<ThreadLocalLimiter>` is how the shared limiter is read lock-free from worker threads while updates happen on the main thread.

## Configuration
```yaml
filter:
  extension_filter:
    name: envoy.access_loggers.extension_filters.process_ratelimit
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.process_ratelimit.v3.ProcessRateLimitFilter
      dynamic_config:
        resource_name: my_access_log_bucket
        config_source: { ads: {} }
```
The actual rate is delivered as an `envoy.type.v3.TokenBucket` resource named `my_access_log_bucket`.

## Stats / errors
- Scope prefix: `access_log.process_ratelimit.` (filter.cc:23).
- Counters (filter.h:16):
  - `allowed` - records that passed the limiter.
  - `denied` - records suppressed by the limiter.
- xDS discovery scope: `local_ratelimit_discovery` (provider_singleton.h:126).
- Config errors: missing `dynamic_config` throws; null limiter at `evaluate` time trips an `ENVOY_BUG`.
