# Custom Response (`envoy.filters.http.custom_response`)

Replaces or mutates upstream responses that match a configured `Matcher::MatchTree<HttpMatchingData>` using pluggable "custom response policies" (e.g. a local body, a redirect to another URI, or a fully recreated upstream request). Policies are registered via `Matcher::ActionFactory<CustomResponseActionFactoryContext>` and invoked on the encode path. The filter supports per-route/per-virtual-host match trees that compose with the listener-level tree.

Proto: `envoy.extensions.filters.http.custom_response.v3.CustomResponse`.

## Files
- `custom_response_filter.h/cc` — `CustomResponseFilter` (both decoder and encoder sides).
- `config.h/cc` — `FilterConfig` holds the stats prefix and the compiled `MatchTreePtr`; exposes `getPolicy()`.
- `factory.h/cc` — `CustomResponseFilterFactory`, main filter factory plus route-specific factory (`factory.cc:22`). `REGISTER_FACTORY` at `factory.cc:31`.
- `policy.h` — `Policy` base class (`policy.h:22`), `CustomResponseFilterState` (`policy.h:34`), `CustomResponseActionFactoryContext`, and the `PolicyMatchActionFactory<PolicyConfig>` helper plus `REGISTER_CUSTOM_RESPONSE_POLICY_FACTORY` macro.

## Lifecycle
`CustomResponseFilter` extends `Http::PassThroughFilter` (`custom_response_filter.h:18`). One `FilterConfig` is built per listener in `createFilterFactoryFromProtoTyped` (`factory.cc:10`) and shared with every stream; per-route `FilterConfig`s are built in `createRouteSpecificFilterConfigTyped` (`factory.cc:21`) and plumbed through `RouteSpecificFilterConfig`.

- `decodeHeaders` (`custom_response_filter.cc:17`): looks up `CustomResponseFilterState` in the stream's filter state. If present it means a prior policy already recreated the stream, so the filter must not capture request headers for route-specific config lookup. Otherwise it stashes `&header_map` in `downstream_headers_`. Always `Continue`.
- `encodeHeaders` (`custom_response_filter.cc:35`): the main decision point.
  1. If filter state is set (this is the custom response itself), delegate directly to `filter_state->policy->encodeHeaders` (`custom_response_filter.cc:43-45`) and return.
  2. Walk the per-filter config hierarchy (least-specific → most-specific) via `Http::Utility::getAllPerFilterConfig<FilterConfig>` and let each match; the most specific non-empty match wins (`custom_response_filter.cc:50-60`). Empty matches are ignored so they don't overwrite earlier hits.
  3. Fall back to the listener-level `config_->getPolicy` (`custom_response_filter.cc:62-64`).
  4. If no policy matched, pass through.
  5. Otherwise invoke `policy->encodeHeaders(headers, end_stream, *this)` to perform the mutation/redirect/recreate.
- `onLocalReply` (`custom_response_filter.h:35`): sets `on_local_reply_called_ = true` so policies can observe that the terminal reply path was entered and choose to no-op when needed. Always returns `Continue`.
- `setDecoderFilterCallbacks` / `setEncoderFilterCallbacks` cache the callback pointers used by policies through the filter handle.

`FilterConfig::getPolicy` (`config.cc:52`) builds an `HttpMatchingDataImpl` from `stream_info`, calls `data.onResponseHeaders(headers)`, evaluates the tree with `Matcher::evaluateMatch`, and dynamic-casts the matched action back to `Policy`.

## Decision / logic
- Filter state short-circuit: `custom_response_filter.cc:27-33` and `custom_response_filter.cc:40-46` — if a policy previously set `CustomResponseFilterState::kFilterStateName`, the current response is the synthetic/recreated one and must only be mutated by that same policy.
- Per-filter config traversal uses `getAllPerFilterConfig` so more specific configs can override less specific ones, but only if they actually produce a match (`custom_response_filter.cc:51-60`).
- Listener-level fallback is checked only when no per-route config produced a match (`custom_response_filter.cc:62-64`).
- Matchers are optional: `createMatcher` returns an empty tree if `custom_response_matcher` is unset (`config.cc:40-42`), which makes `getPolicy` return an empty shared_ptr (`config.cc:54-56`).
- Policy lookup uses the Matcher API with a `CustomResponseMatchActionValidationVisitor` that unconditionally accepts data inputs (`config.cc:20-27`).

## Configuration
- `custom_response_matcher` — `Matcher::MatchTree` keyed on `HttpMatchingData` (response headers + stream info). Each leaf action must be a registered custom response policy (see `REGISTER_CUSTOM_RESPONSE_POLICY_FACTORY`, `policy.h:74-78`).
- Per-route / per-virtual-host override via `createRouteSpecificFilterConfigTyped` (`factory.cc:21-27`): the same proto message compiled into a route-scoped `FilterConfig`. Multiple levels are composed by `encodeHeaders`.
- Built-in policy types (separate directories) plug in via `PolicyMatchActionFactory<PolicyConfig>` whose `createAction` forwards the config + `ServerFactoryContext` + stats prefix to `createPolicy` (`policy.h:55-59`).

## Stats
No counters or gauges are declared in this filter's `FilterConfig`. A `Stats::StatName stats_prefix_` is threaded through to `CustomResponseActionFactoryContext` (`policy.h:47`) so individual policy implementations can publish their own stats scoped under that prefix.
