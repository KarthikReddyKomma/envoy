# Mutation Rules (shared filter infrastructure)

Small helper library that evaluates a proposed HTTP header mutation (SET / APPEND / REMOVE) against a `HeaderMutationRules` proto and returns one of `OK`, `IGNORE`, or `FAIL`. It is used by filters that let an external party (ext_authz, ext_proc) rewrite request/response headers, so they can safely bound what the external service is allowed to touch.

## Files
- `mutation_rules.h` — public enums `CheckOperation` / `CheckResult` and the `Checker` class (`mutation_rules.h:18`, `mutation_rules.h:26`, `mutation_rules.h:32`).
- `mutation_rules.cc` — rule evaluation + internal `ExtraRoutingHeaders` singleton that lists headers which influence routing (`mutation_rules.cc:22`).
- `BUILD` — single `envoy_cc_library` target `mutation_rules_lib`.

## Public interface
- `enum class CheckOperation { SET, APPEND, REMOVE }` (`mutation_rules.h:18`).
- `enum class CheckResult { OK, IGNORE, FAIL }` (`mutation_rules.h:26`).
- `Checker(const envoy::config::common::mutation_rules::v3::HeaderMutationRules&, Regex::Engine&)` — compiles `allow_expression` / `disallow_expression` regexes and caches the boolean flags (`mutation_rules.cc:38`).
- `CheckResult check(CheckOperation, const Http::LowerCaseString& name, absl::string_view value) const` — the only method filters call (`mutation_rules.h:41`, `mutation_rules.cc:57`).

## Implementation logic
`Checker::check` first calls `isAllowed` and, for non-removes, `isValidValue`; if both pass it returns `OK`, otherwise it returns `FAIL` when `disallow_is_error_` is set or `IGNORE` to silently drop the change (`mutation_rules.cc:59`).

`isAllowed` order of precedence (`mutation_rules.cc:69`):
1. `REMOVE` of a non-modifiable header (`:`-prefixed or `host`) is always rejected via `HeaderUtility::isModifiableHeader` (`mutation_rules.cc:70`).
2. `disallow_expression_` regex match - always rejected (`mutation_rules.cc:74`).
3. `allow_expression_` regex match - always accepted (`mutation_rules.cc:78`).
4. `disallow_all_` flag - always rejected (`mutation_rules.cc:82`).
5. `allow_all_routing_` == false blocks modifications to host/:authority/:method/:scheme, the set built in `ExtraRoutingHeaders` ctor (`mutation_rules.cc:24`, `mutation_rules.cc:86`).
6. `:`-prefixed headers are blocked on `APPEND` (HTTP/2 pseudo headers are single-valued) or when `disallow_system_` is set (`mutation_rules.cc:91`).
7. `allow_envoy_` == false blocks the `x-envoy-*` prefix returned by `Http::Headers::get().prefix()` (`mutation_rules.cc:97`).

`isValidValue` enforces general RFC validity via `HeaderUtility::headerValueIsValid` for non-`:` headers, `authorityIsValid` for `host`/host-legacy, `schemeIsValid` for `:scheme`, and a numeric `>= 200` check for `:status` (`mutation_rules.cc:104`-`131`). The `ExtraRoutingHeaders` singleton is lazy-built via `CONSTRUCT_ON_FIRST_USE` (`mutation_rules.cc:134`).

## Consumers
- `source/extensions/filters/http/ext_proc` — external processing filter, stored in `ExtProcFilter` and consulted before applying header mutations returned by the ext_proc gRPC server (`ext_proc.cc`, `mutation_utils.h`, `mutation_utils.cc`).
- `source/extensions/filters/http/ext_authz` — external authorization filter, consults the `Checker` before applying header overrides from the authz response (`ext_authz.cc`, `ext_authz.h`).

No network-filter consumer today; the library is HTTP-only because it operates on `Http::LowerCaseString`.

## Stats / errors / failure modes
No stats are emitted by this library; callers own counters for rejected mutations. The library signals failure through `CheckResult`:
- `IGNORE` - caller should drop the mutation and continue.
- `FAIL` - caller should abort the HTTP transaction (typically respond with an error).

Regex compilation errors surface during `Checker` construction via `THROW_OR_RETURN_VALUE(Regex::Utility::parseRegex...)` and propagate to the filter factory (`mutation_rules.cc:47`, `mutation_rules.cc:52`).
