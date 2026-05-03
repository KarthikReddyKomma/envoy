# RBAC Filter (`envoy.filters.http.rbac`)

Role-based access control at the HTTP layer. Evaluates every request against a
set of policies built from principals (IP, certificate, JWT, headers, filter
state) and permissions (method, path, header, metadata, URL path). Decisions
produce allow / deny, or — in shadow mode — observation-only metrics.

Proto: `envoy.extensions.filters.http.rbac.v3.RBAC`.

## Lifecycle

- Pure decoder filter. Runs in `decodeHeaders()` (`rbac_filter.cc:248`).
- Evaluates the **enforced** engine first (`handleAction()` at
  `rbac_filter.cc:209`). Deny → 403 + `StopIteration`.
- Evaluates the **shadow** engine separately (`rbac_filter.cc:166`). Never
  blocks; only records stats and metadata.

## Configuration

- `rules` — enforced policy set.
- `shadow_rules` — observation-only policy set.
- `rules_stat_prefix` / `shadow_rules_stat_prefix` — metric namespaces.
- `track_per_rule_stats` — per-policy hit counters.
- `matcher` + `shadow_matcher` — matcher-API (xDS matcher) form that replaces
  `rules` for more flexible composition.
  Inputs include `source/dest IP and port`, `server_name`, SSL (URI/DNS SAN,
  subject), HTTP headers, filter state, dynamic metadata
  (`rbac_filter.cc:19–68`).
- `RBACPerRoute` — per-route override (usually to disable the filter or swap
  policies for a specific route).

## Decision

`handleAction()` returns allow / deny and the matched policy id. On deny:

- 403 *Forbidden* via `sendLocalReply`.
- `response_detail = "rbac_access_denied_matched_policy[<id>]"`.

Stats increment at `rbac_filter.cc:222–239` (enforced) and `166–180` (shadow).

## Dynamic metadata

Namespace `envoy.filters.http.rbac`:
- `shadow_engine_result` — `allowed` / `denied`
- `shadow_effective_policy_id`
- `enforced_engine_result`
- `enforced_effective_policy_id`

## Stats

`allowed`, `denied`, `shadow_allowed`, `shadow_denied`. With
`track_per_rule_stats`, per-policy variants are emitted.

## Files

- `config.{h,cc}` — factory + proto parsing.
- `rbac_filter.{h,cc}` — filter and engine invocation.
- Shared engine at `source/extensions/filters/common/rbac/`.
