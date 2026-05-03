# Processing Effect (shared filter infrastructure)

A single header that defines the `Effect` enum shared between the external processing filter (`ext_proc`) and the external authorization filter (`ext_authz`). It is the common vocabulary used to record what a remote processor's mutation did to the request/response — so per-mutation outcomes can be logged, flagged on `StreamInfo`, and counted identically across both filters.

## Files
- `processing_effect.h` — the only header; defines `enum class Effect : uint8_t` with five values (`processing_effect.h:12`).
- `BUILD` — header-only `envoy_cc_library` target `processing_effect_lib`.

No `.cc`. There is no logic, no state machine, and no runtime object to construct.

## Public interface
- `Envoy::Extensions::Filters::Common::ProcessingEffect::Effect` — 8-bit enum (`processing_effect.h:12`):
  - `None` — no mutation was applied; default for non-body messages (`processing_effect.h:16`).
  - `MutationApplied` — mutation succeeded; default for `FULL_DUPLEX_STREAMED` body messages (`processing_effect.h:20`).
  - `InvalidMutationRejected` — header/trailer rewrite rejected by the mutation checker (invalid name or value) (`processing_effect.h:23`).
  - `MutationRejectedSizeLimitExceeded` — body/header mutation refused for exceeding a configured size cap (`processing_effect.h:26`).
  - `MutationFailed` — mutation attempted but failed for another reason (e.g. allocation failure, unsupported operation) (`processing_effect.h:29`).

## Implementation logic
None. The enum is consumed by the mutation pipelines inside `ext_proc` and `ext_authz`; they pick one of these values after calling the mutation checker (`source/extensions/filters/common/mutation_rules`) and use it both for debug logging and for attaching outcomes onto `StreamInfo::filterState`. Because the type is `uint8_t`, it is cheap to store in per-stream state and per-callback records.

## Consumers
- `source/extensions/filters/http/ext_proc/mutation_utils.{h,cc}` — returns `Effect` from each header/body mutation helper.
- `source/extensions/filters/http/ext_proc/processor_state.{h,cc}` — carries the last `Effect` per decode/encode direction.
- `source/extensions/filters/http/ext_proc/ext_proc.{h,cc}` — sets filter-state and logs based on `Effect`.
- `source/extensions/filters/http/ext_authz/ext_authz.{h,cc}` — records the outcome of header overrides requested by the authz response.

No network filter currently depends on this library.

## Stats / errors / failure modes
Not applicable: the library has no code paths. Consumers are responsible for mapping `Effect` values onto stats (counters) and/or `StreamInfo::responseCodeDetails()` / filter-state entries.
