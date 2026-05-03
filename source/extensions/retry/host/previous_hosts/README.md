# PreviousHosts Retry Host Predicate

Prevents the router from retrying against any host that was already tried for
the same downstream request. Useful for ensuring retries try fresh endpoints.

Proto: `envoy.extensions.retry.host.previous_hosts.v3.PreviousHostsPredicate`

## Files
- `previous_hosts.h` - predicate implementation (header-only).
- `config.h` / `config.cc` - factory registered as
  `envoy.retry_host_predicates.previous_hosts`.

## Interface
Implements `Upstream::RetryHostPredicate`. The factory implements
`Upstream::RetryHostPredicateFactory`.

## Logic
The predicate owns a `std::vector<HostDescription const*>` of attempted hosts.
`onHostAttempted` pushes the raw pointer of each attempted host into that
vector. `shouldSelectAnotherHost` performs a linear `std::find` over the
vector and returns true (reject) if the candidate pointer is present.

Pointer identity is safe here because the router keeps each attempted host
alive (via its `HostDescriptionConstSharedPtr`) for the lifetime of the
retry policy, which is also the lifetime of this predicate.

## Key decision points
- `previous_hosts.h:9-12` - membership test in the attempted set.
- `previous_hosts.h:13-15` - append on `onHostAttempted`.

## Configuration
The proto message has no fields; enable by referencing the predicate by name
in `RetryPolicy.retry_host_predicate`.

## Stats
None locally; upstream retry counters live on the router.
