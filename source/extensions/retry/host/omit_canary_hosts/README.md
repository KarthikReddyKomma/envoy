# OmitCanaryHosts Retry Host Predicate

Skips hosts marked as canary when the router is selecting a retry target.
This allows retry attempts to avoid hosts that were deployed as canaries,
keeping retried traffic on the stable fleet.

Proto: `envoy.extensions.retry.host.omit_canary_hosts.v3.OmitCanaryHostsPredicate`

## Files
- `omit_canary_hosts.h` - predicate implementation (header-only).
- `config.h` - factory that registers the predicate under the name
  `envoy.retry_host_predicates.omit_canary_hosts`.
- `config.cc` - factory registration.

## Interface
Implements `Upstream::RetryHostPredicate`. The factory implements
`Upstream::RetryHostPredicateFactory`.

## Logic
`shouldSelectAnotherHost` returns `candidate_host.canary()`. The router will
reject the candidate host (pick a different one) whenever the candidate is
flagged as canary in its `HostDescription`. `onHostAttempted` is a no-op: the
predicate does not need to track previously attempted hosts since the decision
is purely a per-host property.

## Key decision points
- `omit_canary_hosts.h:9` - `shouldSelectAnotherHost` simply delegates to
  `Upstream::Host::canary()`.

## Configuration
The proto message has no fields. Enabling the extension by name is the only
knob; turn it on via `retry_host_predicate` on a `RetryPolicy`.

## Stats
None beyond the generic upstream retry stats emitted by the router.
