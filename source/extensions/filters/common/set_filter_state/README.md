# Set Filter State (shared filter infrastructure)

Shared configuration object and runtime helper that write entries into `StreamInfo::FilterState` from a list of `envoy.extensions.filters.common.set_filter_state.v3.FilterStateValue` protos. It is the single engine behind the HTTP, network, and listener `set_filter_state` filters — each of them is a thin wrapper that instantiates one `Config`, calls `updateFilterState(context, stream_info)` at the appropriate lifecycle point, and (for HTTP) optionally invalidates the route cache.

## Files
- `filter_config.h/cc` — `Value` struct, `Config` class, and two built-in `FilterState::ObjectFactory` implementations (`envoy.string`, `envoy.hashable_string`) (`filter_config.h:22`, `filter_config.h:31`, `filter_config.cc:15`, `filter_config.cc:26`).
- `BUILD` — single `envoy_cc_library` `set_filter_state_config`.

## Public interface
- Aliases: `LifeSpan = FilterState::LifeSpan`, `StateType = FilterState::StateType`, `StreamSharing = StreamInfo::StreamSharingMayImpactPooling`, `FilterStateValueProto = envoy::...FilterStateValue` (`filter_config.h:16`-`20`).
- `struct Value { key_, factory_, state_type_, stream_sharing_, skip_if_empty_, value_ (formatter) };` (`filter_config.h:22`).
- `class Config : public Router::RouteSpecificFilterConfig`:
  - `Config(proto_values, life_span, GenericFactoryContext&)` (`filter_config.h:34`).
  - `Config(proto_values, life_span, GenericFactoryContext&, bool clear_route_cache)` (`filter_config.h:37`).
  - `void updateFilterState(const Formatter::Context&, StreamInfo::StreamInfo&) const` (`filter_config.h:41`).
  - `bool clearRouteCache()` (`filter_config.h:42`).
- `using ConfigSharedPtr = std::shared_ptr<Config>;` (`filter_config.h:52`).
- Registered factories (global side effects from including the `.cc`): `GenericStringObjectFactory` (name `envoy.string`) and `GenericHashableStringObjectFactory` (name `envoy.hashable_string`) (`filter_config.cc:17`, `filter_config.cc:36`). The hashable variant wraps `Router::StringAccessorImpl` and implements `Hashable::hash()` via `HashUtil::xxHash64`, enabling it to influence LB hashing (`filter_config.cc:26`-`32`).

Because `Config` extends `RouteSpecificFilterConfig`, it doubles as the per-route config type for the HTTP variant.

## Implementation logic
`Config::parse` (`filter_config.cc:45`) runs once at config load and turns each proto entry into a `Value`:
1. `key_` is copied from `object_key`. The factory is looked up by `factory_key` (falling back to `object_key`) in the global `FilterState::ObjectFactory` registry; a missing factory throws `EnvoyException("'<key>' does not have an object factory")` (`filter_config.cc:52`-`59`).
2. `state_type_` is `ReadOnly` iff `read_only` is set, else `Mutable` (`filter_config.cc:60`).
3. `shared_with_upstream` is mapped: `ONCE` -> `SharedWithUpstreamConnectionOnce`, `TRANSITIVE` -> `SharedWithUpstreamConnection`, default -> `None`. These control whether the state pushes to the upstream L4 connection on first pool check-out only, or for every pooled connection (`filter_config.cc:61`-`71`).
4. `skip_if_empty_` is carried through (`filter_config.cc:72`).
5. `value_` is a compiled `Formatter` built from `format_string` via `SubstitutionFormatStringUtils::fromProtoConfig`, so `%REQ(...)%` / `%DYNAMIC_METADATA(...)%` / etc. are evaluated at runtime against the supplied `Formatter::Context` + `StreamInfo` (`filter_config.cc:73`).

`Config::updateFilterState` (`filter_config.cc:81`) is called per request/connection:
- For each `Value`, format the string against the runtime `Formatter::Context` + `StreamInfo`.
- If the result is empty and `skip_if_empty_` is set, log debug and skip (`filter_config.cc:85`).
- Otherwise call `factory_->createFromBytes(bytes_value)`; a `nullptr` (factory-level validation failure, e.g. the bytes aren't a valid proto) is logged and skipped — it does **not** abort the request (`filter_config.cc:90`).
- On success, `info.filterState()->setData(key_, object, state_type_, life_span_, stream_sharing_)` stores the entry. `life_span_` (fixed per `Config`) determines whether the entry lives for the filter chain, the request, the connection, or TOP/stream-lifetime (`filter_config.cc:95`).

There is no state machine, no async callback, and no mutation rollback; every `Value` is independent and failures are per-key.

## Consumers
- `source/extensions/filters/http/set_filter_state` — HTTP filter; constructs `Config` with `clear_route_cache=true/false` and clears the route cache after `updateFilterState` when the flag is set.
- `source/extensions/filters/network/set_filter_state` — TCP network filter; calls `updateFilterState` on `onNewConnection`.
- `source/extensions/filters/listener/set_filter_state` — listener filter; calls `updateFilterState` during `onAccept` with the listener's stream info.

All three filters share the same factory parsing, registry lookup, and formatter evaluation; they differ only in when they invoke `updateFilterState` and what `LifeSpan` they pass in.

## Stats / errors / failure modes
- No stats. Diagnostics are debug-level log lines tagged with the key and the formatted value (`filter_config.cc:86`, `:91`, `:94`).
- Hard failure at config time: unknown `factory_key` -> `EnvoyException` bubbling up through the filter factory (`filter_config.cc:58`).
- Hard failure at config time: `format_string` that fails to parse -> `THROW_OR_RETURN_VALUE` raises from `SubstitutionFormatStringUtils::fromProtoConfig` (`filter_config.cc:73`).
- Soft failure at runtime: empty formatted value + `skip_if_empty_ = true` or factory-returned `nullptr` -> key is not written but other keys in the same `Config` still apply. This is intentional so partial updates don't poison the filter state.
