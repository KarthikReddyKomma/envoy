# RBAC (shared filter infrastructure)

Shared RBAC engine used by both the HTTP RBAC filter (`source/extensions/filters/http/rbac`) and the network RBAC filter (`source/extensions/filters/network/rbac`). It compiles an `envoy.config.rbac.v3.RBAC` proto (or the newer `xds::type::matcher::v3::Matcher` form) into a tree of `Matcher` objects and evaluates each request/connection against the policies. It also exposes two extension points — `MatcherExtensionFactory` (permissions) and `PrincipalExtensionFactory` (principals) — so custom checks can plug in by proto.

## Files
- `engine.h` — abstract `RoleBasedAccessControlEngine` interface with the two `handleAction` overloads (network-only vs HTTP) (`engine.h:17`).
- `engine_impl.h/cc` — two concrete engines (`RoleBasedAccessControlEngineImpl` for legacy `rules`, `RoleBasedAccessControlMatcherEngineImpl` for the xDS matcher tree), `Action` + `ActionFactory`, `DynamicMetadataKeys`, and `generateLog` (`engine_impl.h:72`, `engine_impl.h:100`, `engine_impl.cc:14`).
- `matcher_interface.h` — `Matcher` abstract base and its two static `create(...)` factories (`matcher_interface.h:20`).
- `matchers.h/cc` — concrete matchers for permissions/principals: `AndMatcher`, `OrMatcher`, `NotMatcher`, `AlwaysMatcher`, `HeaderMatcher`, `IPMatcher` (LC-Trie), `PortMatcher`/`PortRangeMatcher`, `AuthenticatedMatcher`, `MetadataMatcher`, `FilterStateMatcher`, `RequestedServerNameMatcher`, `PathMatcher`, `UriTemplateMatcher`, `PolicyMatcher` (`matchers.h:29`-`304`).
- `matcher_extension.h` — `MatcherExtensionFactory` (category `envoy.rbac.matchers`) + `BaseMatcherExtensionFactory<M,P>` template (`matcher_extension.h:17`, `matcher_extension.h:36`).
- `principal_extension.h` — `PrincipalExtensionFactory` (category `envoy.rbac.principals`) + `BasePrincipalExtensionFactory` template (`principal_extension.h:16`).
- `utility.h/cc` — `RoleBasedAccessControlFilterStats`, `generateStats`, `createEngine`/`createShadowEngine` template helpers, `responseDetail` (`utility.h:32`, `utility.h:69`, `utility.cc:13`, `utility.cc:33`).
- `matchers/upstream_ip_port.{h,cc}` — built-in `MatcherExtensionFactory` that matches the upstream's resolved IP/port (`upstream_ip_port.h:18`).
- `principals/mtls_authenticated/mtls_authenticated.{h,cc}` — built-in `PrincipalExtensionFactory` that matches validated-mTLS peer certs (`mtls_authenticated.h:16`).

## Public interface
- `class RoleBasedAccessControlEngine { bool handleAction(connection, headers, info, effective_policy_id*); bool handleAction(connection, info, effective_policy_id*); };` (`engine.h:17`).
- `enum class EnforcementMode { Enforced, Shadow };` (`engine_impl.h:32`).
- `class Matcher { virtual bool matches(connection, headers, info) = 0; static MatcherConstPtr create(Permission, ...); static MatcherConstPtr create(Principal, ...); };` (`matcher_interface.h:20`).
- `MatcherExtensionFactory::create(const Protobuf::Message&, ValidationVisitor&) -> MatcherConstPtr`, category `envoy.rbac.matchers` (`matcher_extension.h:24`).
- `PrincipalExtensionFactory::create(const envoy::config::core::v3::TypedExtensionConfig&, CommonFactoryContext&) -> MatcherConstPtr`, category `envoy.rbac.principals` (`principal_extension.h:23`).
- `createEngine<ConfigType>(config, server, validation, action_validation)` / `createShadowEngine<...>` — dispatch to the right engine based on `has_matcher()` / `has_rules()` (`utility.h:70`, `utility.h:90`).
- `generateStats(prefix, rules_prefix, shadow_rules_prefix, scope)` — builds `rbac.<prefix>.{allowed,denied,shadow_allowed,shadow_denied}` counters plus per-policy dynamic-counter helpers (`utility.cc:13`).
- `responseDetail(policy_id)` — returns `rbac_access_denied_matched_policy[<id-with-spaces->_>]` for `StreamInfo::responseCodeDetails()` (`utility.cc:33`).

## Implementation logic
### Legacy `rules`-based engine (`RoleBasedAccessControlEngineImpl`)
- Constructor walks `rules.policies()`, builds one `PolicyMatcher` per entry. If any policy has a CEL `condition` but no `cel_config`, an arena-based CEL `BuilderInstance` is built once and shared across those policies (backwards-compat optimization) (`engine_impl.cc:50`-`66`).
- `handleAction(conn, info, id*)` delegates to `handleAction(conn, StaticEmptyHeaders::request_headers, info, id*)` — network filters use this overload (`engine_impl.cc:69`).
- `handleAction(conn, hdrs, info, id*)` (`engine_impl.cc:76`) calls `checkPolicyMatch`: iterates `policies_` in map order, stops at the first `PolicyMatcher::matches` success, writes the policy name into `*effective_policy_id` (`engine_impl.cc:97`). Then the RBAC action decides the boolean:
  - `ALLOW` -> return matched.
  - `DENY` -> return !matched.
  - `LOG` -> `generateLog(info, mode_, matched)` sets `envoy.common.access_log_hint` dynamic metadata (only when not Shadow) and always returns `true` so filter chain continues (`engine_impl.cc:88`, `engine_impl.cc:33`).
- `PANIC_DUE_TO_CORRUPT_ENUM` on unreachable paths.

### Matcher-tree engine (`RoleBasedAccessControlMatcherEngineImpl`)
- Constructor uses `Envoy::Matcher::MatchTreeFactory<HttpMatchingData, ActionContext>` to turn the xDS matcher into a tree; captures whether any action is `LOG` via `ActionContext::has_log_`; throws on validation errors (`engine_impl.cc:115`-`129`).
- `handleAction` wraps `StreamInfo` + headers in `HttpMatchingDataImpl`, calls `evaluateMatch` which asserts completeness, reads the matched `Action` (name, RBAC action), optionally calls `generateLog` with `mode_` and returns `ALLOW`/`LOG` -> true, `DENY` -> false. Non-match -> default DENY (`engine_impl.cc:139`-`171`).

### Matchers
`Matcher::create(Permission)` dispatches on `rule_case()` to the concrete type; same for `Principal::create` on `identifier_case()` (`matchers.cc:18`, `matchers.cc:73`). Composite forms (`AndMatcher`, `OrMatcher`, `NotMatcher`) recurse. `IPMatcher` builds an `Network::LcTrie::LcTrie<bool>` for O(log n) CIDR matching and extracts one of four addresses per the `Type` enum: connection remote, downstream local, downstream direct remote, downstream remote (`matchers.cc:207`-`274`). `AuthenticatedMatcher` pulls SANs (URI first, then DNS) falling back to the certificate subject (`matchers.cc:326`-`353`). `PolicyMatcher::matches` requires **all three**: `permissions_`, `principals_`, and (if present) the CEL `expr_` (`matchers.cc:375`).

Both `HeaderMatcher` flavors are toggled by runtime feature `envoy.reloadable_features.rbac_match_headers_individually` at construction (`matchers.cc:191`-`204`).

### Built-in extensions
- `UpstreamIpPortMatcher` — reads the chosen upstream host from the connection's stream info to match after load-balancing (`upstream_ip_port.h:18`).
- `MtlsAuthenticatedMatcher` — rejects if `ssl()` missing or `peerCertificateValidated()` false; if configured, matches SANs via `Ssl::SanMatcher`; requires at least one of `san_matcher` or `any_validated_client_certificate` to be set (throws otherwise) (`mtls_authenticated.cc:12`-`41`).

### Stats helper
`RoleBasedAccessControlFilterStats` exposes 4 static counters plus `incPolicyAllowed/Denied/ShadowAllowed/ShadowDenied(name)` which intern stat names dynamically under `rbac.<prefix>.policy.<name>.<suffix>` (`utility.h:32`-`62`).

## Consumers
- `source/extensions/filters/http/rbac/rbac_filter.{h,cc}` — HTTP filter; uses `createEngine` + `createShadowEngine` to build two engines (enforce + shadow), calls the `(conn, headers, info, id*)` overload per request, reports stats + `responseDetail(policy_id)` on deny.
- `source/extensions/filters/network/rbac/rbac_filter.{h,cc}` — network filter; same engines but calls the header-less overload per new connection.

Both filters link the matcher-extension libraries (e.g. `upstream_ip_port`) via explicit `deps` so those `REGISTER_FACTORY`ed matchers are available when building permissions.

## Stats / errors / failure modes
- Counters: `rbac.<rules_prefix>.{allowed,denied}` and `rbac.<shadow_rules_prefix>.{shadow_allowed,shadow_denied}`, plus `...policy.<policy_name>.{allowed,denied,shadow_allowed,shadow_denied}` dynamic variants (`utility.h:18`-`54`).
- Config errors: invalid CIDR / empty IP-range / out-of-bounds port range -> `absl::InvalidArgumentError` or `EnvoyException` at engine-construction time (`matchers.cc:212`, `matchers.cc:232`, `matchers.cc:302`). CEL compile failure -> `Expr::CelException` from `PolicyMatcher` ctor (`matchers.h:220`). Matcher-tree validation errors throw `EnvoyException` with the first error message (`engine_impl.cc:127`).
- Runtime: `IPMatcher::matches` returns `false` for pipe addresses instead of crashing (`matchers.cc:281`). `MetadataMatcher` with `ROUTE` source returns `false` when the route is null (`matchers.cc:359`).
- Shadow mode: the shadow engine never emits access-log metadata and its `allow/deny` decisions are counted under the shadow counters only — the enforce engine is the source of truth for the request outcome.
