# Original Src (`envoy.filters.http.original_src`)

An HTTP filter that propagates the downstream client's source IP onto upstream connections. On every request the filter reads the stream's `downstreamRemoteAddress()` and attaches socket options (via the shared `Filters::Common::OriginalSrc` helper) so that the upstream socket is bound to that address, allowing back-end services to observe the real client IP. The filter is a strict decode-only pass-through; it does not modify headers or bodies.

Proto: `envoy.extensions.filters.http.original_src.v3.OriginalSrc`.

## Files
- `config.h/cc` — `Config` value object holding the single `mark_` field parsed from the proto (`config.cc:10-11`).
- `original_src.h/cc` — `OriginalSrcFilter` implementing `Http::StreamDecoderFilter`.
- `original_src_config_factory.h/cc` — `OriginalSrcConfigFactory` extending `Common::FactoryBase` and registered via `REGISTER_FACTORY` (`original_src_config_factory.cc:27`).

## Lifecycle
Installed as a decoder-only filter (`original_src_config_factory.cc:20`: `callbacks.addStreamDecoderFilter(...)`). The filter keeps a copy of `Config` and a raw `Http::StreamDecoderFilterCallbacks*` acquired in `setDecoderFilterCallbacks` (`original_src.cc:43-45`).

Overridden callbacks:
- `decodeHeaders(...)` (`original_src.cc:15-33`) — performs the work once per request.
- `decodeData(...)` (`original_src.cc:35-37`) — returns `Continue`, no buffering.
- `decodeTrailers(...)` (`original_src.cc:39-41`) — returns `Continue`.
- `onDestroy()` (`original_src.cc:13`) — no-op.

No encoder-side overrides; the filter is purely upstream-facing.

## Decision / logic
`decodeHeaders` branch points:
- `original_src.cc:16-18`: reads `streamInfo().downstreamAddressProvider().remoteAddress()` and asserts non-null.
- `original_src.cc:20-23`: if the address is not `Network::Address::Type::Ip` (e.g. AF_UNIX), the filter logs nothing special and returns `Continue` without touching socket options. Non-IP downstreams simply pass through as if the filter were not installed.
- `original_src.cc:29-31`: calls `Filters::Common::OriginalSrc::buildOriginalSrcOptions(address, mark)` to construct the platform-specific socket options, then pushes them onto the upstream via `callbacks_->addUpstreamSocketOptions(options_to_add)`.

The work is idempotent per request but runs on every request; the returned options partition the upstream connection pool by `(source-ip, mark)`.

## Configuration
- `mark` (`uint32`) — SO_MARK value applied to upstream sockets, copied into `Config::mark_` (`config.cc:10-11`). A value of `0` means no mark. Consumed by the socket-option builder and also mixed into the connection-pool partition key.

There is no per-route configuration, no dynamic route-specific override, and no stat prefix handling (the factory ignores `stat_prefix` at `original_src_config_factory.cc:17`).

## Stats
None. This filter emits no counters, gauges, or histograms; the only observable signal is the `ENVOY_LOG(debug, ...)` trace in `decodeHeaders` (`original_src.cc:25-27`).

## Factory
`OriginalSrcConfigFactory` (`original_src_config_factory.h:15`):
- Inherits `Common::FactoryBase<envoy::extensions::filters::http::original_src::v3::OriginalSrc>` with name `"envoy.filters.http.original_src"` (`original_src_config_factory.h:18`).
- `createFilterFactoryFromProtoTyped` builds a `Config` and returns a lambda that adds a fresh `OriginalSrcFilter` per stream (`original_src_config_factory.cc:15-22`).
- Registered as a `NamedHttpFilterConfigFactory` at static init (`original_src_config_factory.cc:27`).
