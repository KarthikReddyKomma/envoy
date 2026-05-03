# Geoip (`envoy.filters.http.geoip`)

Decoder-only filter that performs an IP geolocation lookup against a configured provider driver and injects the resulting geo headers (e.g. country, city, ASN) into the request headers before continuing decoding. The lookup source is either a custom header, X-Forwarded-For, or the downstream connection remote address.

Proto: `envoy.extensions.filters.http.geoip.v3.Geoip`.

## Files
- `config.h/cc` — factory `GeoipFilterFactory` (ExceptionFree factory base); validates that `xff_config` and `custom_header_config` are mutually exclusive, resolves the `GeoipProviderFactory` via `Envoy::Config::Utility::getAndCheckFactory` and builds the driver, then creates the filter via `callbacks.addStreamDecoderFilter`.
- `geoip_filter.h/cc` — `GeoipFilterConfig` (thread-shared config holding `use_xff_`, `xff_num_trusted_hops_`, `ip_address_header_` and a symbolized stat name set) and `GeoipFilter` (the `StreamDecoderFilter`). `GeoipFilter` inherits `std::enable_shared_from_this` so the async driver callback can safely re-enter the filter.

## Lifecycle
- `decodeHeaders` (geoip_filter.cc:41): captures `headers` into `request_headers_`, resolves the client IP (see decision points below), asserts the provider driver is present, then calls `driver_->lookup(...)` with a callback that posts back to the decoder dispatcher and invokes `onLookupComplete` through a `weak_from_this()` guard (geoip_filter.cc:76-85). Returns `StopAllIterationAndWatermark` so downstream filters do not see the request until the lookup finishes.
- `decodeData` (geoip_filter.cc:92): no-op, returns `Continue`.
- `decodeTrailers` (geoip_filter.cc:96): no-op, returns `Continue`.
- `setDecoderFilterCallbacks` (geoip_filter.cc:100): stashes `decoder_callbacks_`.
- `onDestroy` (geoip_filter.cc:39): empty; the `weak_ptr` in the posted lambda handles the race where the stream is torn down before the lookup completes.
- `onLookupComplete` (geoip_filter.cc:104): on dispatcher thread, iterates `LookupResult` entries and calls `request_headers_->setCopy(...)` for each non-empty value, increments the `total` counter via `config_->incTotal()`, then `decoder_callbacks_->continueDecoding()`.

## Decision / logic
- IP source precedence (geoip_filter.cc:46-69):
  1. If `custom_header_config` is set, extract the first value of `ip_address_header_` and parse with `Network::Utility::parseInternetAddressNoThrow`. Parse failure or empty header just logs at debug and falls through.
  2. Else if `use_xff_ && xff_num_trusted_hops_ > 0`, use `Http::Utility::getLastAddressFromXFF(headers, xff_num_trusted_hops_).address_`.
  3. Else (fallback) use `decoder_callbacks_->streamInfo().downstreamAddressProvider().remoteAddress()`.
- Config validation enforces exclusivity (config.cc:17): `has_xff_config() && has_custom_header_config()` returns `InvalidArgumentError`.
- Results are applied by iterating `LookupResult` and only setting headers whose value is non-empty (geoip_filter.cc:109).

## Configuration
- `xff_config.xff_num_trusted_hops` — enables XFF-based extraction; if zero, XFF is not consulted even when `xff_config` is set (geoip_filter.cc:61).
- `custom_header_config.header_name` — custom header to read the client IP from.
- `provider` — typed provider config resolved at factory time to a `Geolocation::GeoipProviderFactory` which yields the `DriverSharedPtr` used by the filter.
- No per-route config is supported.

## Stats
- `<stat_prefix>geoip.total` — counter incremented once per completed lookup in `onLookupComplete` via `GeoipFilterConfig::incTotal` / `incCounter` (geoip_filter.cc:29, 113). The stat name set pre-remembers `"total"` (geoip_filter.cc:26).
- No additional counters or gauges; provider drivers expose their own stats independently.

## Factory
- `REGISTER_FACTORY(GeoipFilterFactory, NamedHttpFilterConfigFactory){"envoy.geoip"}` (config.cc:53) — registers with both the canonical name (`envoy.filters.http.geoip`) and the legacy alias `envoy.geoip`.
