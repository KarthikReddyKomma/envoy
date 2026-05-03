# Set Metadata (`envoy.filters.http.set_metadata`)

A decode-only HTTP filter that injects untyped (`Struct`) and/or typed (`Any`) entries into the stream's dynamic metadata map at the start of each request. Each configured entry targets a namespace; if that namespace already has metadata, the filter either merges/overwrites it or leaves it alone based on `allow_overwrite`. Useful for tagging requests with routing hints, auth context, or diagnostic data consumed by later filters, access logs, or listeners.

Proto: `envoy.extensions.filters.http.set_metadata.v3.Config`.

## Files
- `config.h/cc` — `SetMetadataConfig` factory with server- and factory-context variants; `REGISTER_FACTORY` at `config.cc:41`.
- `set_metadata_filter.h/cc` — defines the per-listener `Config` (parses proto into `untyped_`/`typed_` entry vectors and generates stats) and `SetMetadataFilter` (`PassThroughDecoderFilter` performing the mutation).

## Lifecycle
Installed as a decoder filter (`config.cc:24-25`, `config.cc:36-37`). One `Config` instance is shared across streams; each stream gets a fresh `SetMetadataFilter`.

Overridden callbacks:
- `decodeHeaders(RequestHeaderMap&, bool)` (`set_metadata_filter.cc:50-95`) — applies all untyped then all typed entries, then returns `Continue`.
- `decodeData(Buffer::Instance&, bool)` (`set_metadata_filter.cc:97-99`) — `Continue`, no buffering.
- `setDecoderFilterCallbacks(...)` (`set_metadata_filter.cc:101-103`) — stashes the callbacks pointer.
- `onDestroy()` (`set_metadata_filter.h:64`) — no-op inline.

No encoder-side hooks.

## Decision / logic
`Config` construction (`set_metadata_filter.cc:16-39`):
- `set_metadata_filter.cc:19-23`: if the deprecated `value`/`metadata_namespace` pair is set, a single `UntypedMetadataEntry{allow_overwrite=true, ...}` is pushed first (preserves legacy behavior where the single-entry form always merged).
- `set_metadata_filter.cc:25-38`: iterates `proto_config.metadata()` and pushes an `UntypedMetadataEntry` when `has_value()`, a `TypedMetadataEntry` when `has_typed_value()`, or logs a warning if neither.

`decodeHeaders` branch points:
- `set_metadata_filter.cc:52-73` — untyped path: grabs `streamInfo().dynamicMetadata().mutable_filter_metadata()` and for each entry:
  - `set_metadata_filter.cc:58-60`: namespace absent -> insert wholesale.
  - `set_metadata_filter.cc:61-67`: namespace present and `allow_overwrite` true -> call `StructUtil::update(orig_fields, to_merge)` to merge field-by-field (not a full replace).
  - `set_metadata_filter.cc:68-71`: namespace present and `allow_overwrite` false -> increment `overwrite_denied_` and skip.
- `set_metadata_filter.cc:75-92` — typed path: grabs `mutable_typed_filter_metadata()` and for each entry:
  - `set_metadata_filter.cc:81-83`: namespace absent -> insert.
  - `set_metadata_filter.cc:84-86`: namespace present and `allow_overwrite` true -> assignment replaces the whole `Any` (no merge since typed metadata has no generic merge).
  - `set_metadata_filter.cc:87-90`: namespace present and `allow_overwrite` false -> increment `overwrite_denied_`.

Always returns `FilterHeadersStatus::Continue`.

## Configuration
- `metadata_namespace` + `value` (deprecated) — single untyped entry, always merge-on-collision (`set_metadata_filter.cc:19-23`).
- `metadata[]` — list of `Metadata` messages, each with:
  - `metadata_namespace` (string)
  - `allow_overwrite` (bool)
  - oneof `value` (`google.protobuf.Struct`) or `typed_value` (`google.protobuf.Any`)

`Config` extends `Router::RouteSpecificFilterConfig` (`set_metadata_filter.h:37`), so the proto can be attached at virtual-host or route level in `typed_per_filter_config`, though the filter itself does not currently read per-route configs in `decodeHeaders` — only the listener-level `config_` is consulted. If you need per-route metadata today, combine this filter with another mechanism.

## Stats
Namespaced at `<stats_prefix>set_metadata.` (`set_metadata_filter.cc:42`):
- `overwrite_denied` (counter) — incremented once per entry whose target namespace already existed while `allow_overwrite` was false (`set_metadata_filter.cc:70`, `set_metadata_filter.cc:89`).

Macro definition at `set_metadata_filter.h:21`:
```
#define ALL_SET_METADATA_FILTER_STATS(COUNTER) COUNTER(overwrite_denied)
```

## Factory
`SetMetadataConfig` (`config.h:16`):
- `FactoryBase<envoy::extensions::filters::http::set_metadata::v3::Config>` with name `"envoy.filters.http.set_metadata"` (`config.h:19`).
- `createFilterFactoryFromProtoTyped` builds `Config` from `FactoryContext::scope()` and stats_prefix (`config.cc:17-27`).
- `createFilterFactoryFromProtoWithServerContextTyped` mirrors the above using `ServerFactoryContext::scope()` (`config.cc:29-39`).
- `REGISTER_FACTORY(SetMetadataConfig, NamedHttpFilterConfigFactory)` at `config.cc:41`.
