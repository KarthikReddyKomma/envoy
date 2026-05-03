# GeoIP (L4) (`envoy.filters.network.geoip`)

A minimal `Network::ReadFilter` that performs a geolocation lookup for the downstream client at connection time and stores the resolved attributes (country, city, ASN, etc.) into `streamInfo().filterState()` under the key `envoy.geoip`. Subsequent filters, route matchers and access logs can then read those fields via the filter-state field API.

Proto: `envoy.extensions.filters.network.geoip.v3.Geoip`.

## Files
- `config.h/cc` — `GeoipFilterFactory` (extends `Common::FactoryBase`). Parses the optional `client_ip` substitution formatter, instantiates `GeoipFilterConfig`, resolves the pluggable `Geolocation::GeoipProviderFactory` named in `provider`, builds its driver, and returns a factory callback that adds a `GeoipFilter` to the filter chain (`config.cc:15-45`). Uses `createFilterFactoryFromProtoTyped` which can return `absl::StatusOr` for invalid formatter strings (`config.cc:15, 25-28`).
- `geoip_filter.h/cc` — `GeoipInfo` (filter-state object), `GeoipFilterConfig`, and the `GeoipFilter` read filter.

## Lifecycle
- `initializeReadFilterCallbacks(callbacks)` stores the read callbacks pointer only (`geoip_filter.h:103-105`).
- `onNewConnection()` kicks off an asynchronous lookup and returns `Continue` — data is never held up by the filter (`geoip_filter.cc:55-102`). Logic:
  1. `ASSERT(driver_)`.
  2. If `client_ip_formatter_` is configured, format against the current stream-info. A non-empty, non-`"-"` result is parsed with `Network::Utility::parseInternetAddressNoThrow`; a failed parse or empty result falls back to the connection remote address (`geoip_filter.cc:60-80`).
  3. Otherwise (or as fallback) uses `connectionInfoProvider().remoteAddress()` (`geoip_filter.cc:82-85`).
  4. Grabs `weak_from_this()` so the filter can be safely deleted (e.g. LDS update) while the lookup is outstanding. Calls `driver_->lookup(LookupRequest{remote_address}, ...)`; the provider callback marshals the result back onto the connection dispatcher via `dispatcher.post(...)` before locking the `weak_ptr` and invoking `onLookupComplete` (`geoip_filter.cc:87-99`).
- `onData(data, end_stream)` returns `Continue` unconditionally (`geoip_filter.h:99-101`) — this filter never buffers or blocks traffic.
- `onWrite` — not implemented (read-only filter).

## Lookup completion
`onLookupComplete(result)` (`geoip_filter.cc:104-127`):
- Empty result → only `total` is incremented, no filter-state is set (`geoip_filter.cc:105-109`).
- Otherwise builds a `GeoipInfo` and copies all non-empty `(key, value)` pairs via `setField` (`geoip_filter.cc:111-116`).
- Stores the `GeoipInfo` on `filterState()` under `envoy.geoip` (`GeoipFilterStateKey`, `geoip_filter.h:24`) as `ReadOnly` with `LifeSpan::Connection` (`geoip_filter.cc:119-123`).
- Always bumps `total` at the end (`geoip_filter.cc:126`).

`GeoipInfo` (`geoip_filter.h:29-54`, `geoip_filter.cc:15-36`) implements:
- `serializeAsProto` → `Protobuf::Struct` (string values) for access logging.
- `serializeAsString` → JSON encoding of the struct.
- `hasFieldSupport` / `getField` → per-key lookup returned as `absl::string_view`, enabling `%FILTER_STATE(envoy.geoip:FIELD:<key>)` access-log substitution and matchers.

## Configuration
`Geoip` proto fields consumed (`geoip_filter.cc:38-45`, `config.cc:18-40`):
- `stat_prefix` — used verbatim; the stat is published as `<stat_prefix>geoip.total` (`geoip_filter.cc:42-43`).
- `client_ip` — optional substitution-format string. When present, the filter uses the formatted output as the lookup IP before falling back to the remote address. Validated at factory time (`config.cc:22-29`).
- `provider` — required `TypedExtensionConfig` selecting the `GeoipProviderFactory` (the upstream `maxmind` driver is the canonical provider). The factory translates the opaque config and constructs a `Geolocation::DriverSharedPtr` shared across all connections (`config.cc:34-40`).

No per-route configuration; no processing-mode switch.

## Stats
Stats are registered through a `StatNameSet` rather than the fixed macro:
- `<stat_prefix>geoip.total` — incremented once per lookup completion (hit, empty, or fallback), via `GeoipFilterConfig::incTotal()` (`geoip_filter.h:65`, `geoip_filter.cc:47-50`).

Additional per-field counters are provider-driven: drivers typically register their own "hit" stats via the shared `stat_name_set_` using the stored `stats_prefix_`.
