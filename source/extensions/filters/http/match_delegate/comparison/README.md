# match_delegate: v1 (eager) vs v2 (lazy) comparison

These are **read-only reference copies** for understanding the lazy-initialization
change. They are NOT wired into any `BUILD` target and do not affect compilation.
The real, compiled sources are `../config.h` and `../config.cc`.

Both v1 and v2 are formatted with the repo `.clang-format`
(`UseTab: Always`, `TabWidth/IndentWidth: 4`, `ColumnLimit: 300`) so a side-by-side
diff shows only real logic differences.

| File            | Version | Behavior |
|-----------------|---------|----------|
| `v1_config.h`   | eager   | Original implementation (matches `git HEAD`). |
| `v1_config.cc`  | eager   | Nested filter constructed up-front. |
| `v2_config.h`   | lazy    | Current on-disk implementation. |
| `v2_config.cc`  | lazy    | Nested filter constructed only after a non-skip match. |

## Key differences

### v1 - eager
- `DelegatingFactoryCallbacks` wraps each nested filter as it is added, so the nested
  decoder/encoder filter instance is created immediately at filter-chain build time.
- `base_filter_` is captured in the `DelegatingStreamFilter` constructor.
- `onMatchCallback` is dispatched directly to the already-existing `base_filter_`.
- Access loggers of the nested filter are registered unconditionally, so they run even
  when the match tree resolves to `SkipFilter`.

### v2 - lazy
- A single `DelegatingStreamFilter` is registered as both a stream filter and an
  access-log handler; the nested `Http::FilterFactoryCb` is captured, not invoked.
- The nested filter is created only once the match tree evaluates to a non-skip
  result (`ensureFilterCreated()` / `maybeCreateAndDispatch()`).
- Pending custom match actions are stored (`pending_match_action_`) and dispatched to
  the nested filter after it is created.
- Nested filters and their access loggers are collected via an internal
  `FilterCreationCallbacks`, so access loggers only run when the filter actually ran.

See `../LAZY_INIT_CHANGES.md` for the full write-up.
