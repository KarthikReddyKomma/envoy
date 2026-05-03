# Network Match Delegate (`envoy.filters.network.match_delegate`)

A meta-filter that wraps another network filter and skips it when a match tree evaluates to a `SkipAction`. The delegate intercepts `addReadFilter` / `addWriteFilter` / `addFilter` on the `FilterManager`, replacing each real filter with a `DelegatingNetworkFilter` that evaluates its configured `xds::type::matcher::v3::Matcher` once at `onNewConnection()` time against `Network::MatchingData` (socket + filter state + dynamic metadata). If the match yields a `SkipAction` (or no match tree is configured), all subsequent `onData` / `onWrite` calls short-circuit to `Continue` without invoking the wrapped filter.

Proto: `envoy.extensions.common.matching.v3.ExtensionWithMatcher` (matcher action: `envoy.extensions.filters.common.matcher.action.v3.SkipFilter`).

## Files
- `config.h` / `config.cc` — all logic lives here. Contains `SkipAction` (matcher action base), `DelegatingNetworkFilter` (the wrapper `Network::Filter`), `DelegatingNetworkFilterManager` (a `FilterManager` shim), `SkipActionFactory`, `MatchTreeValidationVisitor`, and `MatchDelegateConfig` (the network filter factory).

## Lifecycle
- `DelegatingNetworkFilter::onNewConnection()` (`config.cc:134-146`) — lazily populates `MatchingDataImpl` from the read callbacks' socket, `StreamInfo::filterState()`, and `StreamInfo::dynamicMetadata()` via `FilterMatchState::onConnection()` (`config.h:48-54`); then calls `evaluateMatchTree()`; if skipped, returns `Continue` without touching the inner filter; otherwise delegates to `read_filter_->onNewConnection()`.
- `DelegatingNetworkFilter::onData()` (`config.cc:148-155`) — if `skipFilter()` is set, returns `Continue`; else forwards to the wrapped read filter. Note the match is only evaluated in `onNewConnection`; `onData` relies on the cached decision.
- `DelegatingNetworkFilter::onWrite()` (`config.cc:163-170`) — same short-circuit pattern for the write filter path.
- `initializeReadFilterCallbacks` / `initializeWriteFilterCallbacks` (`config.cc:157-161`, `172-176`) — store callbacks and forward them to the inner filter so it sees the real `FilterManager` / `Connection`.

## Decision / logic
- `FilterMatchState::evaluateMatchTree()` (`config.cc:94-125`) — runs once (guard `match_tree_evaluated_`). If `has_match_tree_` is false, treats it as an implicit skip (`config.cc:100-104`). Otherwise calls `Matcher::evaluateMatch<Network::MatchingData>()`. On a complete match, inspects the action's `typeUrl`: `SkipAction` (or `nullptr`) sets `skip_filter_ = true` (`config.cc:112-115`). Any other action type is logged at warn level and treated as "do not skip" (`config.cc:117-122`) — composite actions are a future extension.
- `DelegatingNetworkFilterManager` (`config.cc:62-90`) intercepts `addReadFilter` / `addWriteFilter` / `addFilter` and wraps each incoming filter in a `DelegatingNetworkFilter` bound to the same `match_tree_` before delegating to the underlying manager. `addFilter` produces a single `DelegatingNetworkFilter` that is both the read and write filter (`config.cc:79-82`). `removeReadFilter` is intentionally a no-op (`config.cc:84`) and `initializeReadFilters()` returns false (`config.cc:86`) because the real manager owns lifecycle.
- `MatchDelegateConfig::createFilterFactoryFromProtoTyped` (`config.cc:178-188`) requires `extension_config` and looks up the wrapped `NamedNetworkFilterConfigFactory` via `Config::Utility::getAndCheckFactory`.
- `MatchDelegateConfig::createFilterFactory` (`config.cc:190-226`) translates the `Any` config, builds the inner `FilterFactoryCb`, constructs a `MatchTreeValidationVisitor` seeded with the wrapped factory's `matchingRequirements()` (type-URL allowlist in `config.cc:30-58`), then builds the match tree via `Matcher::MatchTreeFactory<Network::MatchingData, NetworkFilterActionContext>` (`config.cc:203-209`). Validation errors throw (`config.cc:212-215`). The returned `FilterFactoryCb` instantiates a `DelegatingNetworkFilterManager` around the real manager and runs the inner factory against it (`config.cc:222-225`).
- `SkipActionFactory` (`config.cc:18-28`) registers the `"skip"` matcher action producing a `SkipAction`. Registered via `REGISTER_FACTORY(SkipActionFactory, Matcher::ActionFactory<NetworkFilterActionContext>)` (`config.cc:60`).
- `MatchDelegateConfig` factory registration: `REGISTER_FACTORY(MatchDelegateConfig, Server::Configuration::NamedNetworkFilterConfigFactory)` (`config.cc:231`).

## Configuration
- `extension_config` (required) — the wrapped network filter's `TypedExtensionConfig`. Must resolve to a registered `NamedNetworkFilterConfigFactory`.
- `xds_matcher` — optional `xds.type.matcher.v3.Matcher`. When absent, the delegate behaves as an always-skip (wrapped filter is never invoked beyond construction).
- The wrapped factory's `matchingRequirements().data_input_allow_list()` constrains which `DataInput` type URLs may appear in `xds_matcher` (`config.cc:33-54`).

## Stats
The filter itself emits no stats. All stats come from the wrapped filter when it is invoked.
