# Rate Limit Config (shared filter infrastructure)

Descriptor-generation engine shared by HTTP rate-limit filters. It takes the `envoy.config.route.v3.RateLimit` proto that used to live only on the router's virtual host, compiles the embedded `actions` / `hits_addend` / `x_ratelimit_option`, and then at runtime turns a request into a flat list of `Envoy::RateLimit::Descriptor`s that are handed to either the `Common::RateLimit::Client` (global RLS) or a local token-bucket. The `apply_on_stream_done` bit also gets surfaced so filters can split "pre-request" from "post-response" rate-limiting.

## Files
- `ratelimit_config.h/cc` — `RateLimitPolicy` (one proto entry) and `RateLimitConfig` (a container around `RepeatedPtrField<ProtoRateLimit>`) (`ratelimit_config.h:20`, `ratelimit_config.h:37`).
- `BUILD` — single `envoy_cc_library` `ratelimit_config`.

## Public interface
- `using ProtoRateLimit = envoy::config::route::v3::RateLimit;` (`ratelimit_config.h:17`).
- `using RateLimitDescriptors = std::vector<Envoy::RateLimit::Descriptor>;` (`ratelimit_config.h:18`).
- `RateLimitPolicy(const ProtoRateLimit&, CommonFactoryContext&, absl::Status& creation_status, bool no_limit = true)` — rejects `limit`/`stage`/`disable_key` unless the caller opts in (`ratelimit_config.h:22`, `ratelimit_config.cc:18`).
- `void RateLimitPolicy::populateDescriptors(headers, stream_info, local_service_cluster, descriptors)` — emits at most one descriptor (`ratelimit_config.cc:148`).
- `bool RateLimitPolicy::applyOnStreamDone() const` — whether this policy fires on stream completion instead of pre-request (`ratelimit_config.h:26`).
- `RateLimitConfig(const RepeatedPtrField<ProtoRateLimit>&, ...)` — constructs a vector of `RateLimitPolicy` (`ratelimit_config.cc:201`).
- `RateLimitConfig::populateDescriptors(..., bool on_stream_done = false)` — filters policies by the `apply_on_stream_done` bit before delegating (`ratelimit_config.cc:209`).
- `RateLimitConfig::empty()`, `::size()` helpers (`ratelimit_config.h:41`).

Constructors take `absl::Status& creation_status` out-parameter — callers check it after construction (no exceptions in the hot path).

## Implementation logic
`RateLimitPolicy` ctor (`ratelimit_config.cc:18`):
- Validates `hits_addend` — exactly one of `format` or `number`, otherwise `InvalidArgumentError`. A formatter expression is compiled via `Formatter::SubstitutionFormatParser::parse` and must yield exactly one provider (`ratelimit_config.cc:20`-`41`).
- Explicitly rejects deprecated `stage` and `disable_key` fields (`ratelimit_config.cc:43`).
- Rejects `limit` when the caller passes `no_limit = true` (the HTTP filter sets it `false` when it does support override limits; the local ratelimit filter sets `true`) (`ratelimit_config.cc:48`).
- Walks `config.actions()` and maps each oneof case to a `Router::*Action` descriptor-producer:
  - `source_cluster`, `destination_cluster`, `remote_address`, `masked_remote_address` - plain actions (`ratelimit_config.cc:58`-`72`, `:122`).
  - `generic_key`, `header_value_match`, `query_parameter_value_match` - legacy literal form, or formatter-backed form when runtime feature `envoy.reloadable_features.enable_formatter_for_ratelimit_action_descriptor_value` is enabled (`ratelimit_config.cc:73`-`85`, `:89`-`101`, `:125`-`137`).
  - `metadata`, `query_parameters`, `request_headers`, `remote_address_match` - straight-through wrappers (`ratelimit_config.cc:64`, `:67`, `:86`, `:138`).
  - `extension` - looks up a `DescriptorProducerFactory` by name; if no factory matches, falls back to a generic HTTP matcher-input descriptor (`MatchInputRateLimitDescriptor`) so any registered matcher input usable in routes can double as a descriptor producer (`ratelimit_config.cc:102`-`121`).
  - `default:` -> `InvalidArgumentError` with the numeric oneof tag (`ratelimit_config.cc:141`).

`populateDescriptors` (`ratelimit_config.cc:148`) iterates `actions_` calling `populateDescriptor(entry, local_service_cluster, headers, stream_info)`. If any action returns `false` the entire descriptor is discarded (short-circuit that matches the router). Empty keys are skipped. Then it computes `hits_addend`:
- if a formatter provider is set, it formats against the request and requires a `number_value` or a numeric `string_value`; invalid values are logged at warn via `ENVOY_LOG_EVERY_POW_2` and the descriptor is dropped (`ratelimit_config.cc:161`-`187`).
- `MAX_HITS_ADDEND = 1000000000` caps the addend and negative values are rejected (`ratelimit_config.cc:16`, `ratelimit_config.cc:178`).
- static `hits_addend_` copied through when set (`ratelimit_config.cc:189`).
Finally the descriptor carries `is_negative_hits_` and `x_ratelimit_option_` back to the caller (`ratelimit_config.cc:194`).

`RateLimitConfig::populateDescriptors` simply iterates all policies, skipping those whose `applyOnStreamDone()` differs from the caller's `on_stream_done` argument (`ratelimit_config.cc:209`).

## Consumers
- `source/extensions/filters/http/ratelimit/ratelimit.{h,cc}` — global RLS HTTP filter (`ratelimit.h` pulls in `RateLimitConfig` to compile route-level rate-limit entries that were moved from the router).
- `source/extensions/filters/http/local_ratelimit/local_ratelimit.{h,cc}` — uses `RateLimitConfig` with `no_limit = true` (local token-bucket supplies the limit itself).

No network-filter consumer; action producers here depend on `Http::RequestHeaderMap`.

## Stats / errors / failure modes
No stats owned here. Errors:
- Configuration errors surface through the `absl::Status& creation_status` out-parameter — callers propagate to the proto factory, which turns it into a config load failure (`ratelimit_config.cc:23`, `:30`, `:44`, `:142`).
- Runtime `hits_addend` parse failures log-and-skip: the entire descriptor is dropped and the request is **not** rate-limited by that policy (`ratelimit_config.cc:185`). Rate-limited logging (`ENVOY_LOG_EVERY_POW_2`) prevents log floods.
- Action `populateDescriptor` returning `false` silently drops the descriptor; this is the idiomatic way for `header_value_match` and friends to signal "does not apply".
