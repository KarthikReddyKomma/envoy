# 02 — RBAC matchers

Part of the [shared RBAC docs](README.md). This page covers the `Matcher` interface, how `Permission` / `Principal` protos become matcher trees, `PolicyMatcher`, and each built-in matcher type.

## Matcher interface

```mermaid
classDiagram
    class Matcher {
        <<interface>>
        +matches(connection, headers, info) bool
        +create(Permission, validation, context)$ MatcherConstPtr
        +create(Principal, context)$ MatcherConstPtr
    }

    Matcher <|-- AlwaysMatcher
    Matcher <|-- AndMatcher
    Matcher <|-- OrMatcher
    Matcher <|-- NotMatcher
    Matcher <|-- HeaderMatcher
    Matcher <|-- IPMatcher
    Matcher <|-- PortMatcher
    Matcher <|-- PortRangeMatcher
    Matcher <|-- AuthenticatedMatcher
    Matcher <|-- PolicyMatcher
    Matcher <|-- MetadataMatcher
    Matcher <|-- FilterStateMatcher
    Matcher <|-- RequestedServerNameMatcher
    Matcher <|-- PathMatcher
    Matcher <|-- UriTemplateMatcher
```

`matches(...)` is pure and side-effect free (aside from logging in some extensions). Empty headers are used for L4 evaluation.

## Factory dispatch

```mermaid
flowchart TD
    subgraph Permission["Matcher::create(Permission)"]
        P0{rule_case}
        P0 -->|and_rules| AND1[AndMatcher]
        P0 -->|or_rules| OR1[OrMatcher]
        P0 -->|not_rule| NOT1[NotMatcher]
        P0 -->|any| ANY1[AlwaysMatcher]
        P0 -->|header| HDR1[HeaderMatcher]
        P0 -->|destination_ip| IP1[IPMatcher DownstreamLocal]
        P0 -->|destination_port| PORT[PortMatcher]
        P0 -->|destination_port_range| PR[PortRangeMatcher]
        P0 -->|requested_server_name| SNI[RequestedServerNameMatcher]
        P0 -->|url_path| PATH1[PathMatcher]
        P0 -->|uri_template| URI[UriTemplateMatcher]
        P0 -->|metadata / sourced_metadata| MD1[MetadataMatcher]
        P0 -->|matcher| EXT1[MatcherExtensionFactory]
    end

    subgraph Principal["Matcher::create(Principal)"]
        I0{identifier_case}
        I0 -->|and_ids| AND2[AndMatcher]
        I0 -->|or_ids| OR2[OrMatcher]
        I0 -->|not_id| NOT2[NotMatcher]
        I0 -->|any| ANY2[AlwaysMatcher]
        I0 -->|authenticated| AUTH[AuthenticatedMatcher]
        I0 -->|source_ip| IP2[IPMatcher ConnectionRemote]
        I0 -->|direct_remote_ip| IP3[IPMatcher DownstreamDirectRemote]
        I0 -->|remote_ip| IP4[IPMatcher DownstreamRemote]
        I0 -->|header| HDR2[HeaderMatcher]
        I0 -->|url_path| PATH2[PathMatcher]
        I0 -->|metadata / sourced_metadata| MD2[MetadataMatcher]
        I0 -->|filter_state| FS[FilterStateMatcher]
        I0 -->|custom| EXT2[PrincipalExtensionFactory]
    end
```

Unset oneof → `PANIC_DUE_TO_CORRUPT_ENUM` (should be caught by proto validation earlier).

## Composite matchers

```mermaid
flowchart LR
    subgraph AND
        A1[m1] --> A2{all true?}
        A3[m2] --> A2
        A4[mN] --> A2
        A2 -->|yes| AT[true]
        A2 -->|first false| AF[false short-circuit]
    end

    subgraph OR
        O1[m1] --> O2{any true?}
        O3[m2] --> O2
        O4[mN] --> O2
        O2 -->|first true| OT[true short-circuit]
        O2 -->|none| OF[false]
    end

    subgraph NOT
        N1[inner] --> N2[negate]
    end
```

- `AndMatcher` / `OrMatcher` recurse via `Matcher::create` for each child.
- Policy lists use **OR** semantics: `Policy.permissions` and `Policy.principals` are wrapped in `OrMatcher`.

## PolicyMatcher

```mermaid
flowchart TD
    PM[PolicyMatcher::matches] --> P[permissions_.matches<br/>OrMatcher]
    P -->|false| F[false]
    P -->|true| R[principals_.matches<br/>OrMatcher]
    R -->|false| F
    R -->|true| C{CEL expr_ present?}
    C -->|No| T[true]
    C -->|Yes| E[expr_->matches info, headers]
    E -->|ok| T
    E -->|fail| F
```

Constructor compiles `policy.condition()` when present (see [01 — Engines](01_engines.md#cel-conditions-rules-engine-only)).

Example logical shape:

```mermaid
graph TD
    subgraph Policy["policy: admin_api"]
        PERM[permissions OR]
        PERM --> H[header :method POST]
        PERM --> PATH[url_path prefix /admin]

        PRIN[principals OR]
        PRIN --> A1[authenticated SPIFFE admin]
        PRIN --> A2[source_ip 10.0.0.0/8]

        CEL[condition CEL optional]
    end

    PERM --- AND((AND))
    PRIN --- AND
    CEL --- AND
```

## IPMatcher (LC Trie)

```mermaid
flowchart TD
    C[IPMatcher::create ranges] --> V{valid CIDRs?}
    V -->|No| E[InvalidArgumentError]
    V -->|Yes| T[Build LcTrie bool]
    T --> M[matches]
    M --> X[extractIpAddress by Type]
    X --> G{address has IP?}
    G -->|No pipe/null| F[false]
    G -->|Yes| Q[trie_->getData address]
    Q --> R{empty?}
    R -->|No| OK[true]
    R -->|Yes| F
```

| `IPMatcher::Type` | Address source | Proto field |
|---|---|---|
| `ConnectionRemote` | `connection.remoteAddress()` | `Principal.source_ip` |
| `DownstreamLocal` | `downstreamAddressProvider.localAddress()` | `Permission.destination_ip` |
| `DownstreamDirectRemote` | `directRemoteAddress()` | `Principal.direct_remote_ip` |
| `DownstreamRemote` | `remoteAddress()` | `Principal.remote_ip` |

Performance: LC Trie → roughly O(log n) over configured prefixes.

## Port matchers

| Class | Proto | Match |
|---|---|---|
| `PortMatcher` | `destination_port` | Local IP port == exact value |
| `PortRangeMatcher` | `destination_port_range` | `start <= port < end` (half-open); ctor validates `0…65536` and `start < end` |

Both read `downstreamAddressProvider().localAddress()`; non-IP → `false`.

## AuthenticatedMatcher (principal)

```mermaid
flowchart TD
    A[matches] --> S{connection.ssl()?}
    S -->|No| F[false]
    S -->|Yes| M{principal_name matcher set?}
    M -->|No| T[true — any authenticated peer]
    M -->|Yes| U[Try URI SANs]
    U -->|hit| T
    U -->|miss| D[Try DNS SANs]
    D -->|hit| T
    D -->|miss| Sub[Match subject DN]
    Sub --> Out[true/false]
```

This is the classic SPIFFE / SAN principal. For “must be validated mTLS”, prefer the [`MtlsAuthenticated`](03_extensions_and_utility.md#mtlsauthenticatedmatcher) extension.

## HeaderMatcher

- Built from `envoy.config.route.v3.HeaderMatcher`.
- Always fails on connections without usable headers (L4 empty map).
- Runtime flag `envoy.reloadable_features.rbac_match_headers_individually` chooses `matchesHeadersIndividually` vs `matchesHeaders` at construction.

Network RBAC rejects header permissions/principals at filter-config time so they never reach the engine there.

## Path & URI template

| Matcher | Input | Notes |
|---|---|---|
| `PathMatcher` | `:path` | Query/fragment stripped by path matcher helper; missing Path → false |
| `UriTemplateMatcher` | path value | Via `Router::PathMatcherFactory` from `uri_template` typed config |

## Metadata & filter state

```mermaid
flowchart TD
    MD[MetadataMatcher] --> SRC{metadata_source}
    SRC -->|DYNAMIC default| DM[info.dynamicMetadata]
    SRC -->|ROUTE| RT{info.route()?}
    RT -->|Yes| RM[route->metadata]
    RT -->|No| F[false]

    FS[FilterStateMatcher] --> FS2[info.filterState]
```

`sourced_metadata` selects source explicitly; legacy `metadata` field always uses `DYNAMIC`.

## RequestedServerNameMatcher

Matches `connection.requestedServerName()` (typically TLS SNI) with a `StringMatcher`.

## AlwaysMatcher

`any: true` → always `true`. Used for “any permission” / “any principal” wildcards.

## HTTP-only vs L4-safe

| Matcher | HTTP | Network L4 |
|---|---|---|
| Header / Path / UriTemplate | Yes | No (empty headers / config-rejected) |
| IP / Port / SNI / Authenticated / FilterState / Metadata | Yes | Yes |
| Upstream IP/port extension | After upstream chosen | Same (needs filter state) |

## Next

- Engines / actions → [01 — Engines](01_engines.md)
- Extensions, stats, consumers → [03 — Extensions & utility](03_extensions_and_utility.md)
