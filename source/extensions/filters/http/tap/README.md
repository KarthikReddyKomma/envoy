# Tap (`envoy.filters.http.tap`)

HTTP-level tap filter. Feeds request/response headers, bodies, and trailers into a shared tapping pipeline (admin-attached or static sink) so operators can capture structured traces of HTTP traffic. The filter itself is thin: a per-request `HttpPerRequestTapper` created from the current `HttpTapConfig` receives each event and decides whether to emit a record on stream destroy (via `AccessLog::Instance::log`).

Proto: `envoy.extensions.filters.http.tap.v3.Tap`.

## Files
- `config.h/cc` — `TapFilterFactory` registered as `envoy.filters.http.tap`; installs the filter as both a stream filter and an access-log handler.
- `tap_config.h` — interface for `HttpTapConfig` / `HttpPerRequestTapper` (`createPerRequestTapper`).
- `tap_config_impl.h/cc` — concrete `HttpTapConfigImpl` backed by `Extensions::Common::Tap`.
- `tap_filter.h/cc` — `FilterConfig`/`FilterConfigImpl` plus the `Filter` class.

## Lifecycle
- Registered at `config.cc:51` (`REGISTER_FACTORY`). `createFilterFactoryFromProtoTyped` (`config.cc:32-46`) builds an `HttpTapConfigFactoryImpl` and a `FilterConfigImpl` (which is an `ExtensionConfigBase`, so it can be driven by the admin TapDS machinery). The returned cb installs the same `Filter` instance both as a stream filter and as an access-log handler (`config.cc:42-44`), which is what makes `log()` fire on destroy.
- `FilterConfigImpl` ctor (`tap_filter.cc:10-17`) forwards `common_config` to `ExtensionConfigBase`, stashes the raw proto, and generates the stats block.
- `Filter::setDecoderFilterCallbacks` (`tap_filter.h:100-107`) calls `config_->currentConfig()` once per stream; if a tap config is active it creates a per-request tapper via `createPerRequestTapper(getTapConfig(), callbacks)`, otherwise `tapper_` stays null and every subsequent hook is a no-op.
- Decode path (`tap_filter.cc:30-49`): `decodeHeaders/Data/Trailers` each call `tapper_->onRequestHeaders/Body/Trailers`; `decodeData` skips empty chunks (`tap_filter.cc:38`).
- Encode path (`tap_filter.cc:51-70`): symmetric, calling `onResponseHeaders/Body/Trailers`. `encode1xxHeaders` is passthrough (`tap_filter.h:110-112`), `encodeMetadata` passthrough (`tap_filter.h:117-119`).
- Access log on destroy (`tap_filter.cc:72-76`): `log()` calls `tapper_->onDestroyLog()` — if that returns true the filter bumps `rq_tapped_`.

## Decision / logic
- Tap activation is per stream: `tap_filter.h:102` — dynamic admin config is read once at filter creation. If no config is installed at that instant, the stream runs untapped even if a tap is pushed later.
- Every decode/encode hook is guarded by `tapper_ != nullptr` (`tap_filter.cc:31, 38, 45, 52, 59, 66`).
- `log()` is the only place `rq_tapped_` is incremented and only if the per-request tapper signals that something was actually emitted (`tap_filter.cc:73-74`).

## Configuration
- `common_config` — `envoy.config.common.tap.v3.CommonExtensionConfig` (static, admin, or TapDS); plumbed through `ExtensionConfigBase`.
- No per-route config; the filter operates identically for every stream it runs on. All enablement/disablement is expressed through the TapDS `match` tree.

## Stats
Prefix `<stats_prefix>tap.` (`tap_filter.cc:26`). Counters (`tap_filter.h:23-25`):
- `rq_tapped` — number of streams for which the per-request tapper emitted a log record on destroy.
