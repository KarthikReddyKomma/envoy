# Local Rate Limiter (shared filter infrastructure)

In-process, thread-safe token-bucket rate limiter reused by the HTTP local_ratelimit filter, the network local_ratelimit filter, the listener-level local_ratelimit filter, and the process-level rate limit access log filter. It owns per-descriptor atomic token buckets, a default bucket, wildcard-driven dynamic descriptors with LRU eviction, and an optional `ShareProvider` that divides tokens evenly across the local cluster so a distributed fleet can approximate a cluster-wide limit with zero RPC cost.

## Files
- `local_ratelimit.h` - Abstract interface: `LocalRateLimiter` (`requestAllowed(...)` returning a `Result`) and `TokenBucketContext` (`maxTokens`/`remainingTokens`/`resetSeconds`/`shadowMode` for `x-ratelimit-*` headers).
- `local_ratelimit_impl.h/cc` - Concrete `LocalRateLimiterImpl`, `RateLimitTokenBucket`, `DynamicDescriptor`, `DynamicDescriptorMap`, `ShareProviderManager` (singleton), and `AlwaysDenyLocalRateLimiter` used when config asks for deny-all.

## Public interface
- `LocalRateLimiter::Result` (`local_ratelimit.h:27`) - `{bool allowed; shared_ptr<const TokenBucketContext> token_bucket_context; RateLimit::XRateLimitOption x_ratelimit_option;}`. The filter uses `token_bucket_context` to fill `x-ratelimit-limit`, `x-ratelimit-remaining`, and `x-ratelimit-reset` headers.
- `LocalRateLimiter::requestAllowed(absl::Span<const RateLimit::Descriptor>)` (`local_ratelimit.h:36`) - sole hot-path entrypoint.
- `LocalRateLimiterImpl(fill_interval, max_tokens, tokens_per_fill, Dispatcher&, descriptors_proto, always_consume_default_token_bucket=true, ShareProviderSharedPtr=nullptr, lru_size=20)` (`local_ratelimit_impl.h:123`) - constructed from the filter's proto config.
- `ShareProviderManager::singleton(dispatcher, cluster_manager, singleton_manager)` (`local_ratelimit_impl.h:80`) and `getShareProvider(ProtoLocalClusterRateLimit)` - returns a `ShareProvider` whose `getTokensShareFactor()` returns `1 / local_cluster.membership_total` (`local_ratelimit_impl.cc:22`).
- `RateLimitTokenBucket` (`local_ratelimit_impl.h:98`) - wraps `AtomicTokenBucketImpl`, implements `TokenBucketContext`, offers `consume(factor, tokens)` and `refill(tokens)`.
- `AlwaysDenyLocalRateLimiter` (`local_ratelimit_impl.h:142`) - always returns `{false, nullptr, UNSPECIFIED}` for safe fallback when config is invalid.

## Implementation logic
- Construction asserts each token-bucket fill interval is `>= 50ms` (both the default and every per-descriptor bucket), throwing `EnvoyException` otherwise (`local_ratelimit_impl.cc:101`, `:125`).
- Descriptors whose entries contain an empty value become "wildcard" and flow through `DynamicDescriptorMap`; exact descriptors land in `descriptors_` directly (`local_ratelimit_impl.cc:128`).
- `requestAllowed` (`local_ratelimit_impl.cc:148`):
  1. Looks each request descriptor up in `descriptors_`; falls back to `dynamic_descriptors_.getBucket()` which walks configured wildcards and materializes a per-tuple bucket on demand.
  2. Sorts matches ascending by `fillRate()` so the tightest bucket is consumed first (`:170`).
  3. Reads the current share factor from the `ShareProvider` if set (`:173`).
  4. For each matched descriptor: if the request has a negative addend, `refill(addend)`; otherwise `consume(share_factor, addend_or_1)`. First failure short-circuits and returns `{false, bucket, x_ratelimit_option}` (`:176`).
  5. If nothing matched, or `always_consume_default_token_bucket_` is true, consumes the default bucket as a final gate (`:189`).
- `DynamicDescriptor` keeps a mutex-guarded hash map plus an LRU list. On overflow the tail entry is evicted (`local_ratelimit_impl.cc:283`). Match semantics: keys must be equal, values match either exactly or when the config side is empty (wildcard) (`:218`).
- `RateLimitTokenBucket::consume` converts `to_consume / factor` so a share factor of `0.25` effectively requires 4x tokens to grant a single request (`local_ratelimit_impl.cc:73`).
- `RateLimitTokenBucket::refill` hands the bucket a negative delta, capped so the level never exceeds `max_tokens` (`local_ratelimit_impl.cc:79`).
- `ShareProviderManager` is a singleton registered under `local_ratelimit_share_provider_manager`. It only exists when `local_cluster_name` is set and present in the cluster manager; otherwise `singleton()` returns `nullptr` (`local_ratelimit_impl.cc:54`). The monitor watches `PrioritySet::addMemberUpdateCb` and recomputes `1/num_hosts` whenever local cluster membership changes (`:35`).
- The singleton's destructor hops the main dispatcher to unregister the callback so teardown is race-free (`local_ratelimit_impl.cc:43`).

## Consumers
- HTTP: `source/extensions/filters/http/local_ratelimit`.
- Network: `source/extensions/filters/network/local_ratelimit`.
- Listener: `source/extensions/filters/listener/local_ratelimit`.
- Access log filter: `source/extensions/access_loggers/filters/process_ratelimit`.

## Stats / errors / failure modes
- No stats here. Each consumer filter tracks `ok`, `rate_limited`, `shadow_mode_hits`, and tag-extracted descriptor counters in its own scope.
- Constructor throws `EnvoyException` on `fill_interval < 50ms` (`local_ratelimit_impl.cc:101`) or on duplicate descriptors (`:136`, `:244`). The factory must translate this into proto-config validation.
- A descriptor's `shadow_mode_` flag is carried on the `TokenBucketContext`; the filter is expected to treat the request as allowed but still emit the "would-have-denied" counter.
- When the local cluster is removed at runtime, the singleton keeps the last share factor until the next member update; a missing local cluster at startup silently disables sharing (factor stays `1.0`).
- Dynamic descriptor eviction is LRU per wildcard bucket (default `lru_size=20`) - this is an anti-abuse guard, not configurable per request; callers with high cardinality should bump `lru_size`.
