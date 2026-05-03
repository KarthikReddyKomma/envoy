# Stateful Session (`envoy.filters.http.stateful_session`)

Pins a request stream to a specific upstream host using a session token that is created / consumed by a pluggable `SessionStateFactory` (e.g. cookie-based). On request, the filter asks the session state for an upstream address and, if present, calls `setUpstreamOverrideHost` on the router. On response, it lets the session state mutate outbound headers to persist the chosen host back to the client. Supports strict (fail-closed) mode and per-route overrides including full disablement.

Proto: `envoy.extensions.filters.http.stateful_session.v3.StatefulSession` (and `StatefulSessionPerRoute`).

## Files
- `config.h/cc` — `StatefulSessionFactoryConfig` (main + upstream + per-route factory entry points).
- `stateful_session.h/cc` — `StatefulSessionConfig`, `PerRouteStatefulSession`, and the `StatefulSession` filter (a `PassThroughFilter`).

## Lifecycle
- Registered at `config.cc:45` (`REGISTER_FACTORY`). `createFilterFactoryFromProtoTyped` (`config.cc:14-22`) and the server-context variant (`config.cc:24-34`) build a `StatefulSessionConfig` and install the filter via `addStreamFilter`. Per-route configs are built by `createRouteSpecificFilterConfigTyped` (`config.cc:36-43`).
- `StatefulSessionConfig` ctor (`stateful_session.cc:24-42`) loads `strict` and the strict-mode status code, optionally constructs stats (only if `stat_prefix` is non-empty — `stateful_session.cc:27`), and resolves `session_state` via `SessionStateFactoryConfig`. Absent `session_state` defaults to a no-op `EmptySessionStateFactory` (`stateful_session.cc:17-20, 33`).
- `PerRouteStatefulSession` ctor (`stateful_session.cc:44-51`): if override case is `kDisabled`, marks the route entry disabled; otherwise constructs an inner `StatefulSessionConfig` with an empty stats prefix so per-route entries never emit stats (comment at `stateful_session.cc:49`).
- `decodeHeaders` (`stateful_session.cc:53-76`) resolves the most specific per-filter config; if disabled, returns `Continue` and leaves `filter_active_ = false`. Otherwise it creates a `SessionStatePtr` from request headers and, if the state surfaces an `upstreamAddress()`, sets the load-balancer override via `decoder_callbacks_->setUpstreamOverrideHost({addr, strict, statusOnMissingStrictDestination})`.
- `encodeHeaders` (`stateful_session.cc:78-119`) either records `no_session_` if the filter was active but no session token existed (only when upstream was actually reached — `stateful_session.cc:83-87`), or delegates `onUpdate(host, response_headers)` so the state plugin can emit/update cookies. It then inspects whether the host changed to classify outcomes (see below).

## Decision / logic
- Per-route resolution: `stateful_session.cc:55` — `resolveMostSpecificPerFilterConfig<PerRouteStatefulSession>`; disabled routes short-circuit at `stateful_session.cc:58-60`.
- No session → `Continue`: `stateful_session.cc:68-70`.
- Override host set: `stateful_session.cc:72-74` — third arg to `OverrideHost` is the `strict` flag and the response code to return if strict host is unreachable.
- Encode classification (`stateful_session.cc:91-107`):
  - `host_changed == true` (fallback host was used) and `!strict` → `failed_open_`.
  - `host_changed == false` (override honored) → `routed_`.
- Strict fail-closed: when no upstream info is present and the stream has `NoHealthyUpstream` set with `strict_` true → `failed_closed_` (`stateful_session.cc:111-115`).
- Status on missing strict destination: `stateful_session.cc:25` — uses `status_on_strict_destination_not_found` when strict; otherwise defaults to `503 Service Unavailable`.

## Configuration
- `session_state` — `TypedExtensionConfig` resolved via `SessionStateFactoryConfig` at `stateful_session.cc:37-41`.
- `strict` — boolean; forces status-code failure when the pinned host is unavailable.
- `status_on_strict_destination_not_found` — HTTP code used in strict mode (default 503).
- `stat_prefix` — empty disables stats emission entirely (`stateful_session.cc:27-30`).
- Per-route (`StatefulSessionPerRoute`): oneof between `disabled` (fully skip the filter) and a nested `stateful_session` config used in place of the listener config (`stateful_session.cc:45-50`). Per-route configs do not emit stats.

## Stats
Prefix `<stats_prefix>stateful_session.<stat_prefix>.` (only when `stat_prefix` is set; `stateful_session.cc:28`). Counters (`stateful_session.h:30-34`):
- `routed` — override host was honored (upstream did not change).
- `failed_open` — non-strict mode; request fell back to a different host.
- `failed_closed` — strict mode; no healthy upstream and request rejected.
- `no_session` — filter active but no session token was extracted for a request that reached upstream.
