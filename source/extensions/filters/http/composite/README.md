# Composite Filter (`envoy.filters.http.composite`)

A meta-filter that acts as a placeholder in the HTTP filter chain and, on a match-tree match, instantiates and delegates to another real filter (or a chain of them). This lets operators attach filters conditionally (per-host, per-header, per-runtime-fraction) without reconfiguring the whole HCM.

Proto: `envoy.extensions.filters.http.composite.v3.Composite` plus `ExecuteFilterAction` (the match action).

## Files
- `filter.h/cc` — `Filter` (the composite wrapper), `DelegatedFilterChain`, and `Filter::StreamFilterWrapper` for single-side filters.
- `action.h/cc` — `ExecuteFilterAction` plus `ExecuteFilterActionFactory` (match action that points at a real filter config).
- `factory_wrapper.h/cc` — `FactoryCallbacksWrapper`, the shim passed to the inner filter's factory callback to collect what it tried to add.
- `config.h/cc` — `CompositeFilterFactory` (registered for downstream and upstream), named-filter-chain compilation.

## Lifecycle
Two layers of filter:
- `Filter` (`filter.h:90`) derives from `Http::StreamFilter` and `AccessLog::Instance`. Every callback (`decodeHeaders`, `decodeData`, `decodeTrailers`, `decodeMetadata`, `decodeComplete`, `encode1xxHeaders`, `encodeHeaders`, `encodeData`, `encodeTrailers`, `encodeMetadata`, `encodeComplete`, `onDestroy`, `onStreamComplete`) forwards via the `delegateFilterActionOr` helper (`filter.cc:15`) to `delegated_filter_` when one exists, otherwise returns `Continue`.
- `DelegatedFilterChain` (`filter.h:58`) wraps multiple `StreamFilterSharedPtr`s and runs them sequentially. Decode iteration is forward (`filter.cc:315-354`); encode iteration is reverse to mirror HCM ordering (`filter.cc:369-439`).
- `Filter::StreamFilterWrapper` (`filter.h:160`) wraps a single `StreamEncoderFilter` *or* `StreamDecoderFilter` into the `StreamFilter` shape used by `delegated_filter_`; pass-through on the side that's not populated.

Key callbacks:
- `setDecoderFilterCallbacks` / `setEncoderFilterCallbacks` just cache the callbacks so they can be forwarded into whichever filter ends up getting delegated (`filter.h:105`, `filter.h:117`).
- `onMatchCallback(action)` (`filter.cc:95`): invoked by the HCM match-tree when a match fires. This is where delegation actually happens.
- `log(...)` (`filter.h:142`): fans out to any `access_loggers_` the delegated factory installed via its callback wrapper.

## Decision / logic
`onMatchCallback` branches three ways based on the matched `ExecuteFilterAction`:

1. **Named filter chain lookup** (`composite_action.isNamedFilterChainLookup()`, `filter.cc:98`):
   - Early exit if `actionSkip()` (runtime-sample rolled against it).
   - Return quietly if the filter wasn't configured with `named_filter_chains_` (soft fail, `filter.cc:107`).
   - Look up the chain by name in `named_filter_chains_`; missing name is also a soft fail (`filter.cc:115`).
   - Run each pre-compiled factory into a `FactoryCallbacksWrapper` (configured as a chain). If the wrapper collected any filters, build a `DelegatedFilterChain` and bump `filter_delegation_success_`.

2. **Inline filter chain** (`composite_action.isFilterChain()`, `filter.cc:143`):
   - Call `composite_action.createFilters(wrapper)` to instantiate the inline chain, reporting `filter_delegation_error_` on any collected errors.
   - Same wrap-as-`DelegatedFilterChain` path.

3. **Single filter** (`filter.cc:174`): the wrapper holds a `filter_to_inject_` `absl::variant`; an `Overloaded` visitor picks the right overload for `StreamDecoderFilterSharedPtr` / `StreamEncoderFilterSharedPtr` / `StreamFilterSharedPtr` and stores it as `delegated_filter_` (wrapping the half-filter variants in `StreamFilterWrapper`).

After any of these, `delegated_filter_` gets both sets of callbacks plus any access loggers collected by the wrapper. `updateFilterState` (`filter.cc:206`) records which action matched under `MatchedActionsFilterStateKey = "envoy.extensions.filters.http.composite.matched_actions"` (as `MatchedActionInfo`, `filter.h:33`), but only for downstream filters — upstream skips filter-state recording (`filter.cc:208`).

`DelegatedFilterChain`'s iteration short-circuits on the first non-`Continue` status: decodeHeaders returns early on `StopIteration`/`StopAllIteration...` (`filter.cc:319`), and likewise for data/trailers/metadata.

## Configuration
`Composite` proto is essentially empty for the filter itself; the interesting logic is in the match tree on the HCM side and in the `ExecuteFilterAction` placed at match leaves. `ExecuteFilterAction` (`action.h:26`) has three constructors:
- Single filter via `typed_config` or dynamic config discovery (`action.h:33`).
- Inline filter chain (list of `FilterFactoryCb`, `action.h:41`).
- Named filter chain lookup (`action.h:49`).

All three accept an optional `sample_percent` runtime fraction + the `Runtime::Loader`, checked by `actionSkip()` so a match can probabilistically skip delegation.

Named filter chains are compiled at config time by `CompositeFilterFactory::compileNamedFilterChains` (`config.cc:14`): each entry in `named_filter_chains` is translated to a vector of `FilterFactoryCb`s so that matching threads only pay instantiation cost, not parsing cost.

## Stats
Prefix `<stats_prefix>composite.` (`config.cc:66`, `config.cc:84`). From `ALL_COMPOSITE_FILTER_STATS` (`filter.h:25`):
- `filter_delegation_success` — a match successfully installed a delegated filter/chain.
- `filter_delegation_error` — a match fired but the factory callback returned errors.

Note: silent-skip paths (named chain not found, sample rolled negative, no filter collected) do **not** increment either counter.

## Factory
`CompositeFilterFactory` extends `Common::DualFactoryBase<Composite>` (`config.h:25`). Two entry points:
- `createFilterFactoryFromProto(Message, prefix, FactoryContext&)` (`config.cc:54`) — downstream path; compiles named filter chains (needs full `FactoryContext`) and returns a callback adding the filter + registering it as an access log handler.
- `createFilterFactoryFromProtoTyped(..., DualInfo, ServerFactoryContext&)` (`config.cc:78`) — upstream path; named filter chains are not supported here (comment at `config.cc:82`).

Registered twice (`config.cc:98-100`): under `Server::Configuration::NamedHttpFilterConfigFactory` for downstream and `UpstreamHttpFilterConfigFactory` for upstream, aliased via `using UpstreamCompositeFilterFactory = CompositeFilterFactory` (`config.h:50`).
