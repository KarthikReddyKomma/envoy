# Kill Request (`envoy.filters.http.kill_request`)

A testing / chaos filter: when an incoming request (or outgoing response, depending on configured direction) carries a kill-request header and a probabilistic roll succeeds, the filter deliberately crashes Envoy via `RELEASE_ASSERT(false, ...)`. Useful for validating crash-recovery / restart behaviour in non-production environments.

Proto: `envoy.extensions.filters.http.kill_request.v3.KillRequest`.

## Files
- `kill_request_config.h/cc` — `KillRequestFilterFactory` (`FactoryBase`) that constructs a `KillRequestFilter` per stream and a `KillSettings` per-route config; registered under `envoy.filters.http.kill_request` (`kill_request_config.cc:33`).
- `kill_request_filter.h/cc` — `KillRequestHeaderNameValues` (derives the default header `<prefix>-kill-request` from the runtime prefix singleton), `KillRequestFilter` (`Http::StreamFilter`), and `KillSettings` (`Router::RouteSpecificFilterConfig`).

## Lifecycle
- `KillRequestFilter` implements the full `Http::StreamFilter` interface (`kill_request_filter.h:32-90`) but only the two header callbacks perform work.
- `decodeHeaders` (`kill_request_filter.cc:46-70`): checks for the kill header; if present, resolves per-route `KillSettings` and potentially crashes when the configured direction is `REQUEST`.
- `encodeHeaders` (`kill_request_filter.cc:72-83`): only runs when the filter-level direction is not `REQUEST`; if the response carries the kill header and the probability roll passes, it crashes.
- All other callbacks (`decodeData`, `decodeTrailers`, `encode1xxHeaders`, `encodeData`, `encodeTrailers`, `encodeMetadata`, `onDestroy`) are `Continue` no-ops (`kill_request_filter.h:48-77`).
- `setDecoderFilterCallbacks` caches the decoder callbacks (for per-route resolution); `setEncoderFilterCallbacks` is a no-op (`kill_request_filter.h:79`).

## Decision / logic
- Header detection — `isKillRequest` (`kill_request_filter.cc:27-44`):
  - The header name is `kill_request.kill_request_header` when set, otherwise the default `KillRequestHeaders::get().KillRequest` (prefix from `ThreadSafeSingleton<Http::PrefixValue>`).
  - Only the first header value is read (header is untrusted). The value is parsed by `absl::SimpleAtob` and thus accepts `true/t/yes/y/1` (case-insensitive). Missing / unparsable -> not a kill request.
- Probability — `isKillRequestEnabled` (`kill_request_filter.cc:22-25`): `ProtobufPercentHelper::evaluateFractionalPercent(probability, random_generator_.random())`. The random generator is wired in via the factory (`kill_request_config.cc:19`).
- Per-route override (`kill_request_filter.cc:54-62`): `resolveMostSpecificPerFilterConfig<KillSettings>` supplies a route-specific `{probability, direction}`. When present, the probability is copied into `kill_request_.mutable_probability()` so later calls to `isKillRequestEnabled` use the route value even if the `KillSettings` is freed; the direction drives `is_correct_direction`.
- Crash conditions:
  - Decode: `is_kill_request && is_correct_direction && isKillRequestEnabled()` triggers `RELEASE_ASSERT(false, "KillRequestFilter is crashing Envoy!!!")` (`kill_request_filter.cc:64-67`).
  - Encode: only evaluated when `direction != REQUEST`; `isKillRequest(response_headers) && isKillRequestEnabled()` triggers the same `RELEASE_ASSERT` (`kill_request_filter.cc:77-80`).
- Early exit on decode when the kill header is absent (`kill_request_filter.cc:49-51`).

## Configuration
- `probability` (`envoy.type.v3.FractionalPercent`) — how often a kill-request header actually crashes Envoy.
- `kill_request_header` — overrides the default header name.
- `direction` — `REQUEST` (default) or `RESPONSE`; selects whether `decodeHeaders` or `encodeHeaders` is allowed to crash.
- Per-route: `KillSettings` copies `probability` and `direction` (`kill_request_filter.cc:19-20`); the per-route config is an instance of the same `KillRequest` proto.

## Stats
None. The filter emits no counters or gauges.
