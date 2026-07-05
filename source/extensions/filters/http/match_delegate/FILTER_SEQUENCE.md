# Filter Sequence: Regular vs. Match-Delegate

Most of the confusion comes from two unrelated things both called "callbacks":

- **`FilterChainFactoryCallbacks`** — used *at chain-construction time* so a filter can register
  itself into the chain (`addStreamDecoderFilter`, `addStreamEncoderFilter`, `addStreamFilter`).
- **`Stream{Decoder,Encoder}FilterCallbacks`** — used *at runtime* by a filter to drive the stream
  (`streamInfo()`, `continueDecoding()`, `sendLocalReply()`, ...). These are pushed into the filter
  via `set{Decoder,Encoder}FilterCallbacks`.

match-delegate wraps the **first** kind (to insert a wrapper filter) and merely **forwards** the
second kind.

Two timing facts that the diagrams depend on (verified in code):

- The `FilterFactoryCb` and the filter-level `match_tree` are built **once at config load**; the
  `match_tree` is **shared** across all streams.
- `set{Decoder,Encoder}FilterCallbacks` is called **synchronously while the chain is being built**
  — the `ActiveStreamDecoderFilter`/`ActiveStreamEncoderFilter` constructor calls it immediately
  (`filter_manager.h:252`, `:342`). For match-delegate that is where `matching_data_` is created
  (`onStreamInfo`), so it exists before any `decodeHeaders`.

---

## 1. Regular filter

A normal filter's config factory returns a `FilterFactoryCb`. Per stream the filter manager runs
it with the real `FilterChainFactoryCallbacks`, and the filter registers **itself**. The manager
wraps it in an `ActiveStream*Filter`, whose constructor immediately hands back the runtime callbacks.

```mermaid
sequenceDiagram
    autonumber
    participant Cfg as Filter config factory
    participant FM as FilterManager (per stream)
    participant F as MyFilter

    rect rgb(235, 245, 255)
    note over Cfg: Config load (once)
    Cfg->>Cfg: createFilterFactoryFromProto(...)
    Cfg-->>FM: FilterFactoryCb (closure)
    end

    rect rgb(235, 255, 240)
    note over FM,F: Chain construction (per stream)
    FM->>Cfg: FilterFactoryCb(real callbacks)
    Cfg->>F: make_shared<MyFilter>(...)
    Cfg->>FM: callbacks.addStreamFilter(MyFilter)
    note over FM: wraps in ActiveStream*Filter, ctor calls set*FilterCallbacks
    FM->>F: setDecoderFilterCallbacks(cb)
    FM->>F: setEncoderFilterCallbacks(cb)
    end

    rect rgb(255, 250, 235)
    note over FM,F: Request / response (per stream)
    FM->>F: decodeHeaders(headers, end_stream)
    F-->>FM: FilterHeadersStatus
    FM->>F: encodeHeaders(headers, end_stream)
    F-->>FM: FilterHeadersStatus
    end
```

The object in the chain **is** `MyFilter`; it uses the runtime callbacks directly.

---

## 2. Match-delegate filter

The config factory builds the inner filter's factory **and** the shared `match_tree` once. Its
`FilterFactoryCb` wraps the real callbacks in `DelegatingFactoryCallbacks`, so when the inner filter
registers itself the wrapper substitutes a `DelegatingStreamFilter(match_tree, inner)`. At runtime
the delegate evaluates the match tree and either **skips** the inner filter or **delegates** to it.

### 2a. Config load + chain construction

```mermaid
sequenceDiagram
    autonumber
    participant Cfg as MatchDelegateConfig
    participant FM as FilterManager (per stream)
    participant DCB as DelegatingFactoryCallbacks
    participant IF as Inner filter
    participant DF as DelegatingStreamFilter
    participant MS as match_state_ (matching_data_)

    rect rgb(235, 245, 255)
    note over Cfg: Config load (once)
    Cfg->>Cfg: createFilterFactory(...)
    Cfg->>Cfg: build inner filter_factory
    Cfg->>Cfg: build shared match_tree
    Cfg-->>FM: FilterFactoryCb capturing [filter_factory, match_tree]
    end

    rect rgb(235, 255, 240)
    note over FM,MS: Chain construction (per stream)
    FM->>Cfg: FilterFactoryCb(real callbacks)
    Cfg->>DCB: DelegatingFactoryCallbacks(real callbacks, match_tree)
    Cfg->>IF: filter_factory(delegating_callbacks)
    IF->>DCB: addStreamFilter(InnerFilter)
    DCB->>DF: make DelegatingStreamFilter(match_tree, Inner, Inner)
    DCB->>FM: real callbacks.addStreamFilter(DelegatingStreamFilter)
    note over FM: wraps DF in ActiveStream*Filter, ctor calls set*FilterCallbacks
    FM->>DF: setDecoderFilterCallbacks(cb)
    DF->>MS: onStreamInfo(cb.streamInfo()) -> create matching_data_
    DF->>IF: setDecoderFilterCallbacks(cb)  (forwarded)
    FM->>DF: setEncoderFilterCallbacks(cb)
    DF->>MS: onStreamInfo(cb.streamInfo())  (no-op if already set)
    DF->>IF: setEncoderFilterCallbacks(cb)  (forwarded)
    end
```

### 2b. Request processing (decode path)

```mermaid
sequenceDiagram
    autonumber
    participant FM as FilterManager
    participant DF as DelegatingStreamFilter
    participant MS as match_state_
    participant MT as match_tree (shared)
    participant IF as Inner filter

    FM->>DF: decodeHeaders(headers, end_stream)
    DF->>DF: resolveMostSpecificPerFilterConfig<FilterConfigPerRoute>
    opt per-route config present
        DF->>MS: setMatchTree(per_route tree)  (overrides filter-level tree)
    end
    DF->>MS: evaluateMatchTree(onRequestHeaders)

    alt match_tree_ == nullptr
        MS-->>DF: skip_filter_ = true
    else evaluate against matching_data_
        MS->>MT: evaluateMatch(match_tree_, matching_data_)
        alt complete & match => SkipAction
            MT-->>MS: skip_filter_ = true
        else complete & match => other action
            MT-->>MS: base_filter_->onMatchCallback(action)
        else incomplete
            MT-->>MS: defer (re-evaluated on later hooks)
        end
    end

    alt skipFilter() == true
        DF-->>FM: Continue (inner filter NOT called)
    else
        DF->>IF: decodeHeaders(headers, end_stream)
        IF-->>DF: FilterHeadersStatus
        DF-->>FM: FilterHeadersStatus
    end

    note over DF,IF: decodeData / decodeMetadata only check cached skipFilter()
    note over DF,IF: decodeTrailers re-runs evaluateMatchTree(onRequestTrailers)
    note over DF,IF: Encode path mirrors this (onResponseHeaders / onResponseTrailers)
```

---

## Side-by-side

| Aspect | Regular filter | Match-delegate |
| --- | --- | --- |
| Built once at config load | `FilterFactoryCb` | `FilterFactoryCb` + shared `match_tree` + inner `filter_factory` |
| Factory callbacks | real, used directly | wrapped in `DelegatingFactoryCallbacks` to intercept registration |
| Object added to chain | the filter itself | `DelegatingStreamFilter` wrapping the inner filter |
| `set*FilterCallbacks` | consumed by the filter | consumed by delegate: creates `matching_data_` (`onStreamInfo`), then forwarded to inner |
| Per-stream allocations | one filter | one inner filter + one `DelegatingStreamFilter` |
| Per-request extra logic | none | evaluate `match_tree` → skip or delegate each hook |
| Match evaluation | n/a | evaluated + cached (`match_tree_evaluated_`); re-tried on trailers if incomplete |

## Code references

- Delegate factory + per-stream wrapping closure: `comparison/v1_config.cc:213-246`.
- `DelegatingFactoryCallbacks` (registration interception): `comparison/v1_config.h:88-116`.
- `DelegatingStreamFilter` hooks (skip vs delegate): `comparison/v1_config.cc:97-195`.
  - `decodeHeaders` (per-route resolve + evaluate): `:97-111`.
  - `set{Decoder,Encoder}FilterCallbacks` (`onStreamInfo` creates `matching_data_`): `:141-145`, `:191-195`.
- `evaluateMatchTree` (feed data, evaluate, cache, skip/`onMatchCallback`): `comparison/v1_config.cc:56-84`.
- Runtime callbacks pushed during chain build: `source/common/http/filter_manager.h:252`, `:342`,
  `addStream*Filter` at `:999-1019`.
