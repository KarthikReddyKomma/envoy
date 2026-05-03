# OmitHostMetadata Retry Host Predicate

Skips hosts whose `envoy.lb` metadata matches a user-supplied criteria set.
Useful for steering retries away from hosts with particular labels (e.g. a
specific zone, version, or shard identifier).

Proto: `envoy.extensions.retry.host.omit_host_metadata.v3.OmitHostMetadataConfig`

## Files
- `omit_host_metadata.h` - predicate class declaration and label-set capture.
- `omit_host_metadata.cc` - `shouldSelectAnotherHost` implementation.
- `config.h` / `config.cc` - factory registered as
  `envoy.retry_host_predicates.omit_host_metadata`.

## Interface
Implements `Upstream::RetryHostPredicate`. The factory implements
`Upstream::RetryHostPredicateFactory`.

## Logic
At construction, the predicate reads `metadata_match.filter_metadata["envoy.lb"]`
and flattens its field map into a `label_set_` vector of `{key, Value}` pairs
(`omit_host_metadata.h:17-24`).

On each retry decision, `shouldSelectAnotherHost` calls
`Config::Metadata::metadataLabelMatch(label_set_, host.metadata(), "envoy.lb",
true)`. When the host's metadata matches every label in the criteria, the host
is rejected (`return true`). The extra `!label_set_.empty()` guard exists
because `metadataLabelMatch` returns true on an empty label set; if no
criteria is provided the predicate must not skip any host.

`onHostAttempted` is a no-op; the predicate is stateless across attempts.

## Key decision points
- `omit_host_metadata.cc:14` - combined empty-set guard plus match call.
- `omit_host_metadata.h:17` - locates the `envoy.lb` filter-metadata bucket
  during construction.
- `config.cc:16` - validates and unwraps the `metadata_match` oneof before
  constructing the predicate.

## Configuration
Users set `OmitHostMetadataConfig.metadata_match` to a `core.v3.Metadata`
message; the matched labels live under the `envoy.lb` filter key to align
with the LB subset match pipeline.

## Stats
None; retries and retry outcomes are counted by the owning router.
