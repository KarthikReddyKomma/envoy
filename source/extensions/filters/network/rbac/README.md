# RBAC Network Filter (`envoy.filters.network.rbac`)

Connection-level role-based access control. Evaluates the shared RBAC engine (policies, permissions, principals, or a matcher tree) against the connection's source/destination IP, port, SNI, network namespace, SSL peer identity, filter state, and dynamic metadata. Allowed connections are forwarded; denied connections are closed (optionally after a configurable delay). A parallel shadow engine records "what would have happened" into dynamic metadata and shadow stats without affecting traffic.

Proto: `envoy.extensions.filters.network.rbac.v3.RBAC`.

## Files
- `config.h` / `config.cc` — `RoleBasedAccessControlNetworkFilterConfigFactory`. Validates that the configured rules contain no HTTP-header principals/permissions (unsupported at L4, `config.cc:19-75`) and adds a `RoleBasedAccessControlFilter` read filter.
- `rbac_filter.h` / `rbac_filter.cc` — `ActionValidationVisitor` (allowed matcher input types), `RoleBasedAccessControlFilterConfig` (wraps the primary and shadow engines and stat sink), and the filter itself.

## Lifecycle
- `onNewConnection()` — returns `Continue` unchanged (`rbac_filter.h:92`). L4 RBAC decides at `onData` time so that downstream address information (e.g., original destination) and SSL peer certificate information are populated.
- `onData()` (`rbac_filter.cc:75-154`) — the decision point:
  1. If `is_delay_denied_`, return `StopIteration` (connection is awaiting the delay timer).
  2. Emit verbose debug line with SNI / addresses / SSL SANs / dynamic metadata (`rbac_filter.cc:80-105`).
  3. Run shadow engine, then enforced engine. In `CONTINUOUS` mode both run on every `onData`; in `ONE_TIME_ON_FIRST_BYTE` mode each is cached by its `Unknown` sentinel (`rbac_filter.cc:110-129`).
  4. `Allow` -> `Continue`; `Deny` -> set connection termination details, optionally start delay timer, return `StopIteration`; no engine -> `Continue`.
- `onEvent()` (`rbac_filter.cc:167-172`) — on close, disables and drops `delay_timer_`.

## Decision / logic
- `checkEngine()` (`rbac_filter.cc:185-225`) fetches the engine for the requested `EnforcementMode`, calls `engine->handleAction(connection, streamInfo, &effective_policy_id)`, and updates stats (`allowed`/`denied` or `shadow_allowed`/`shadow_denied`). For shadow hits it writes `shadow_effective_policy_id` and `shadow_engine_result` into dynamic metadata via `setDynamicMetadata()` (`rbac_filter.cc:174-183`).
- Delay-deny path (`rbac_filter.cc:137-148`): when `delay_deny > 0`, the filter creates a timer in the connection dispatcher, flips `is_delay_denied_`, calls `readDisable(true)`, and on expiry `closeConnection()` sends `NoFlush` close with reason `rbac_deny_close`.
- Matcher inputs are restricted by `ActionValidationVisitor::performDataInputValidation` (`rbac_filter.cc:18-59`) to a fixed allow-list: `DestinationIPInput`, `DestinationPortInput`, `SourceIPInput`, `SourcePortInput`, `DirectSourceIPInput`, `ServerNameInput`, `NetworkNamespaceInput`, SSL `UriSanInput`/`DnsSanInput`/`SubjectInput`, and `FilterStateInput`.
- Config-time validation rejects HTTP-header-based rules because L4 has no headers (`config.cc:19-23`).

## Configuration
- `stat_prefix` — required; used as the base stat prefix.
- `rules` — enforced `envoy.config.rbac.v3.RBAC` (policy map or matcher tree).
- `shadow_rules`, `shadow_rules_stat_prefix` — non-enforcing engine that still emits metrics and dynamic metadata.
- `enforcement_type` — `ONE_TIME_ON_FIRST_BYTE` (default) or `CONTINUOUS` (`rbac_filter.cc:110-129`).
- `delay_deny` — duration to hold a denied connection before closing (`rbac_filter.h:72`, `rbac_filter.cc:73`).

## Stats
Generated via `Filters::Common::RBAC::generateStats(stat_prefix, "", shadow_rules_stat_prefix, scope)`:
- `<prefix>.allowed` / `<prefix>.denied` — enforced engine.
- `<shadow_prefix>.shadow_allowed` / `<shadow_prefix>.shadow_denied` — shadow engine.

## Dynamic metadata
Under `envoy.filters.network.rbac` (`rbac_filter.cc:182`): `<shadow_prefix>.shadow_effective_policy_id` and `<shadow_prefix>.shadow_engine_result` (`allowed`/`denied`).

## Factory
`RoleBasedAccessControlNetworkFilterConfigFactory` is registered with `REGISTER_FACTORY` as a `NamedNetworkFilterConfigFactory` (`config.cc:99-100`). `createFilterFactoryFromProtoTyped` validates both `rules` and `shadow_rules`, then captures a `RoleBasedAccessControlFilterConfigSharedPtr` in a lambda that adds the read filter (`config.cc:77-94`).
