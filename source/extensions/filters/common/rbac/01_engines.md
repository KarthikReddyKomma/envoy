# 01 — RBAC engines

Part of the [shared RBAC docs](README.md). This page covers construction (`createEngine` / `createShadowEngine`), the two concrete engines, ALLOW/DENY/LOG semantics, shadow mode, and dynamic metadata.

## Construction

```mermaid
flowchart TD
    CFG[Filter proto<br/>HTTP or Network RBAC] --> CE[createEngine]
    CFG --> CSE[createShadowEngine]

    CE --> M{has_matcher?}
    M -->|Yes| ME[MatcherEngineImpl<br/>EnforcementMode::Enforced]
    M -->|No| R{has_rules?}
    R -->|Yes| RE[EngineImpl<br/>Enforced]
    R -->|No| N1[nullptr]

    CSE --> SM{has_shadow_matcher?}
    SM -->|Yes| SME[MatcherEngineImpl<br/>EnforcementMode::Shadow]
    SM -->|No| SR{has_shadow_rules?}
    SR -->|Yes| SRE[EngineImpl<br/>Shadow]
    SR -->|No| N2[nullptr]
```

Rules from `utility.h`:

- If both `matcher` and `rules` are set → **matcher wins**, rules ignored with a warn (same for shadow).
- Returned `nullptr` means “no engine”; filters treat that as allow-by-default.

`ActionValidationVisitor` (alias of `MatchTreeValidationVisitor<HttpMatchingData>`) is passed into matcher-engine construction so each filter can restrict which matcher **inputs** are legal (network filter allow-lists L4 inputs only).

## Interface

```mermaid
classDiagram
    class RoleBasedAccessControlEngine {
        <<interface>>
        +handleAction(connection, headers, info, effective_policy_id*) bool
        +handleAction(connection, info, effective_policy_id*) bool
    }

    note for RoleBasedAccessControlEngine "true  = allow traffic\nfalse = deny traffic\neffective_policy_id filled on match"
```

Header-less overload always forwards to the header overload with `Http::StaticEmptyHeaders::request_headers` (`engine_impl.cc`).

`EnforcementMode { Enforced, Shadow }` is stored on each engine and only affects `generateLog` (shadow skips writing `access_log_hint`).

## Legacy rules engine (`RoleBasedAccessControlEngineImpl`)

Built from `envoy.config.rbac.v3.RBAC`:

1. Optionally create a shared CEL arena builder if any policy has `condition` **without** `cel_config` (compat optimization).
2. For each `policies` map entry → one `PolicyMatcher`.
3. Store global `action_` (`ALLOW` / `DENY` / `LOG`).

### Evaluation

```mermaid
flowchart TD
    A[handleAction] --> B[checkPolicyMatch]
    B --> C[Iterate policies_ in map order]
    C --> D{PolicyMatcher.matches?}
    D -->|Yes| E[Record policy name<br/>matched = true<br/>stop]
    D -->|No more| F[matched = false]
    E --> G{action_}
    F --> G

    G -->|ALLOW| H{matched?}
    H -->|Yes| I[return true]
    H -->|No| J[return false]

    G -->|DENY| K{matched?}
    K -->|Yes| J
    K -->|No| I

    G -->|LOG| L[generateLog mode_, matched]
    L --> M[return true always]
```

`checkPolicyMatch` (`engine_impl.cc`): first matching policy wins; lexicographic map order of policy names.

### Policy match (see also [02 — Matchers](02_matchers.md))

A `PolicyMatcher` matches iff:

```text
permissions_ (OR of Permission)  AND
principals_  (OR of Principal)   AND
(CEL condition if present)
```

Empty `rules` with `ALLOW` → no policy can match → deny everyone. Absent rules (no engine) is different from empty rules.

## Matcher-tree engine (`RoleBasedAccessControlMatcherEngineImpl`)

Built from `xds.type.matcher.v3.Matcher` via `MatchTreeFactory<HttpMatchingData, ActionContext>`.

```mermaid
flowchart TD
    A[Ctor] --> B[ActionContext has_log_=false]
    B --> C[MatchTreeFactory.create matcher]
    C --> D[Capture has_log_ from ActionFactory]
    D --> E{validation errors?}
    E -->|Yes| F[throw EnvoyException]
    E -->|No| G[Ready]

    H[handleAction] --> I[HttpMatchingDataImpl from StreamInfo + headers]
    I --> J[evaluateMatch]
    J --> K{isMatch?}
    K -->|Yes| L[Read Action name + RBAC action]
    L --> M{has_log_?}
    M -->|Yes| N[generateLog for LOG actions]
    M -->|No| O[skip]
    N --> P{action}
    O --> P
    P -->|ALLOW / LOG| Q[return true]
    P -->|DENY| R[return false]
    K -->|No| S[Default DENY<br/>return false]
```

Differences from the rules engine:

| | Rules engine | Matcher engine |
|---|---|---|
| Match unit | Named `Policy` | Matcher-tree leaf `Action` |
| Global action | One `rules.action` for all policies | Per-leaf ALLOW/DENY/LOG |
| No match | Depends on global action | **Always deny** |
| Inputs | Permission/Principal fields | `HttpMatchingData` inputs (validated per filter) |

`ActionFactory` (`envoy.filters.rbac.action`) builds `Action(name, action)` and sets `ActionContext::has_log_` if any leaf is `LOG`.

## LOG action and `generateLog`

```mermaid
sequenceDiagram
    participant Engine
    participant StreamInfo
    participant MD as dynamic metadata

    Engine->>Engine: matched? / leaf action == LOG?
    alt mode == Shadow
        Note over Engine: Do not write access_log_hint
    else mode == Enforced
        Engine->>MD: envoy.common.access_log_hint = true/false
        Note over StreamInfo: Shared namespace used by access loggers
    end
    Note over Engine: Rules LOG always returns allow<br/>Matcher LOG leaf returns allow
```

Keys (`DynamicMetadataKeys`):

| Constant | Value |
|---|---|
| `AccessLogKey` | `access_log_hint` |
| `CommonNamespace` | `envoy.common` |
| `ShadowEffectivePolicyIdField` | `shadow_effective_policy_id` |
| `ShadowEngineResultField` | `shadow_engine_result` |
| `EngineResultAllowed` / `Denied` | `allowed` / `denied` |

Shadow effective-policy / result fields are written by the **filters**, not by the shared engine.

## Shadow vs enforced

```mermaid
flowchart LR
    REQ[Request / connection] --> S[Shadow engine]
    S --> SS[shadow_allowed / shadow_denied]
    S --> SM[shadow_* dynamic metadata]
    REQ --> E[Enforced engine]
    E --> ES[allowed / denied]
    E --> OUT{Continue or deny}
```

| | Enforced | Shadow |
|---|---|---|
| Affects traffic | Yes | No |
| Stats | `allowed` / `denied` | `shadow_allowed` / `shadow_denied` |
| `access_log_hint` | Yes (on LOG) | Never |
| Typical use | Production policy | Dry-run candidate policy |

## CEL conditions (rules engine only)

When a policy has `condition`:

- With `cel_config` → use cached CEL builder from factory context.
- Without `cel_config` → share one arena-backed `BuilderInstance` across those policies (`ExprBuilderWithArena`).
- Compile failure → `Expr::CelException` at engine construction.

`PolicyMatcher::matches` evaluates `expr_->matches(info, headers)` as the third conjunct.

## Failure modes

| Phase | Failure | Effect |
|---|---|---|
| Ctor (rules) | Bad CIDR / port range / CEL | Throw / status error — config NACK |
| Ctor (matcher) | Input validation errors | `EnvoyException` with first error |
| Runtime | Pipe / missing address | IP matchers return `false` |
| Runtime | No route + ROUTE metadata | Metadata matcher returns `false` |
| Runtime | Corrupt enum | `PANIC_DUE_TO_CORRUPT_ENUM` |

## Next

- Matcher tree details → [02 — Matchers](02_matchers.md)
- Stats, plugins, consumers → [03 — Extensions & utility](03_extensions_and_utility.md)
