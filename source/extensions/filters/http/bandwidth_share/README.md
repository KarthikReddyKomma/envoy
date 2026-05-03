# Bandwidth Share (Fair Token Bucket)

Not a registered HTTP filter on its own — this folder contains only the `FairTokenBucket` token-distribution library used to fairly share a single byte/second budget across weighted tenants (and their concurrent clients). A filter that wants weighted fair sharing constructs a `Bucket` and hands each stream its own `Client`, which implements `Envoy::TokenBucket`.

No proto in this folder; the component is consumed programmatically.

## Files
- `fair_token_bucket_impl.h/cc` — `Bucket` (shared, mutex-guarded state), `Client` (`TokenBucket` wrapper owned per-stream), and `Tenant` hashset entry.

## Lifecycle (of a Client)
- `Client` is created with a tenant name and weight (`fair_token_bucket_impl.h:66`) and keeps a `shared_ptr<Bucket>`.
- `consume(tokens, allow_partial=true)` (`fair_token_bucket_impl.cc:109`) forwards to `Bucket::requestTokens`. `allow_partial=false` and `consume(..., time_to_next_token)` are not supported and trigger `IS_ENVOY_BUG`. `nextTokenAvailable()` is likewise unsupported.
- Destructor (`fair_token_bucket_impl.cc:15`) calls `Bucket::clientDestroyed`, which under the mutex runs `clientDrained` so the tenant's active-client count can decay.
- `maybeReset` is a deliberate no-op (`fair_token_bucket_impl.h:73`): the real bucket lives in the factory and is only reset at creation.

## Decision / logic
Core algorithm lives in `Bucket::requestTokens` (`fair_token_bucket_impl.cc:80`):
1. `purgeDrainedTenants()` (`fair_token_bucket_impl.cc:42`) expires any tenants whose last drain is older than `fill_interval` — this is what allows the active-tenant set to shrink when clients stop asking.
2. `tokensInBucket()` (`fair_token_bucket_impl.cc:70`) converts the `empty_at_` timestamp into currently-available tokens (capped at 1 second worth).
3. "Loose tokens" = `tokens_in_bucket - tokensPerInterval()` are given out freely (unlimited phase).
4. If the request exceeds loose tokens:
   - First time being limited: call `clientLimited` (`fair_token_bucket_impl.cc:31`) to insert the tenant (or bump `active_clients_`) and return just the loose portion.
   - Already limited: compute the fair share as `tokensPerInterval * tenant.weight_ / total_weight / tenant.active_clients_` (`fair_token_bucket_impl.cc:96-97`).
5. If the fair share satisfies the request, the client is moved back to the draining list with a grace deadline of `now + fill_interval` (`fair_token_bucket_impl.cc:65`) so a one-off spike doesn't instantly vacate the tenant.
6. `consumeTokens(avail)` advances `empty_at_` by the equivalent nanoseconds (`fair_token_bucket_impl.cc:78`).

Key design choices:
- Fast path (common case): when below the near-empty threshold, no tenant bookkeeping happens at all — the lock is held briefly and the client just gets its tokens.
- "Near empty" is defined as within one `fill_interval` worth of tokens from zero (`fair_token_bucket_impl.cc:85`).
- Well-behaved clients are assumed to ask at most once per `fill_interval` — matching `StreamRateLimiter`'s timer cadence. Misbehaving callers could drain unfairly (comment at `fair_token_bucket_impl.h:54-63`).

## Configuration
Constructed via `Bucket::create(max_tokens, time_source, fill_interval)` (`fair_token_bucket_impl.cc:24`). Asserts in the ctor (`fair_token_bucket_impl.cc:17-22`) require `max_tokens > 0`, `max_tokens < 2^64/1e9`, and `10ms <= fill_interval <= 1000ms`. Weights are per-Client and must be `> 0` (`fair_token_bucket_impl.h:66`).

## Stats
None emitted by this library; the consuming filter is expected to instrument its own counters.

## Factory
None. There is no `config.cc`, no proto, and no `REGISTER_FACTORY` call — this directory is a dependency for other filter extensions to link against.
