# SNI Dynamic Forward Proxy (`envoy.filters.network.sni_dynamic_forward_proxy`)

Non-terminal L4 filter that resolves the downstream TLS SNI (or a previously stamped `envoy.upstream.dynamic_host` filter-state string) through the shared dynamic forward proxy DNS cache so a later `tcp_proxy` can connect to the resolved address. It participates in the DNS cache's circuit breaker, pauses reads while resolution is in flight, and optionally stamps the resolved `UpstreamAddress` into filter state for downstream consumers.

Proto: `envoy.extensions.filters.network.sni_dynamic_forward_proxy.v3.FilterConfig`.

## Files
- `proxy_filter.h/cc` — `ProxyFilterConfig` (holds the `DnsCache` handle, port, and `save_upstream_address` flag) and `ProxyFilter` (the `ReadFilter` + `DnsCache::LoadDnsCacheEntryCallbacks`).
- `config.h/cc` — `SniDynamicForwardProxyNetworkFilterConfigFactory`.

## Lifecycle
- `onNewConnection()` (proxy_filter.cc:35) — the only interesting hook. Picks host, enforces the DNS circuit breaker, picks port, kicks off `loadDnsCacheEntry`, and branches on the result.
- `onData()` (proxy_filter.h:47) — inline `return Continue`. The filter does not look at payload.
- `initializeReadFilterCallbacks()` (proxy_filter.h:51) — stores callbacks.
- `onLoadDnsCacheComplete()` (proxy_filter.cc:114) — `DnsCache::LoadDnsCacheEntryCallbacks`. Fires when an async resolution finishes: releases the circuit breaker, optionally writes `UpstreamAddress`, re-enables reads, and calls `continueReading()` to resume the chain.

No `onWrite()`; pure `ReadFilter`.

## Decision / logic
Inside `onNewConnection` (proxy_filter.cc:35):
1. Host selection — prefer the `Router::StringAccessor` filter-state entry keyed `envoy.upstream.dynamic_host`; otherwise fall back to `requestedServerName()` (SNI) (proxy_filter.cc:36-47). Empty host → `Continue` without any resolution (proxy_filter.cc:52-54).
2. Circuit breaker — `config_->cache().canCreateDnsRequest()` returns a RAII handle or null. Null means the pending-request budget is exhausted; the connection is closed with `NoFlush` and `StopIteration` is returned (proxy_filter.cc:56-62).
3. Port selection — check filter-state key `envoy.upstream.dynamic_port` via `StreamInfo::UInt32Accessor`. If present and within `(0, 65535]`, use it; otherwise fall back to the configured `port_value` and also stamp that value back into filter state as mutable (proxy_filter.cc:64-79).
4. DNS lookup — `config_->cache().loadDnsCacheEntry(host, port, false, *this)` (proxy_filter.cc:81). Store the returned `handle_`; when null, reset the circuit-breaker handle immediately (proxy_filter.cc:83-86).
5. Result switch (proxy_filter.cc:88-109):
   - `InCache` — write `UpstreamAddress` if `save_upstream_address` and the host info has a resolved address (proxy_filter.cc:93-96), then `Continue`.
   - `Loading` — `readDisable(true)` to pause the downstream, return `StopIteration`; resumption happens in `onLoadDnsCacheComplete` (proxy_filter.cc:99-103).
   - `Overflow` — close the connection with `NoFlush` and `StopIteration` (proxy_filter.cc:104-108).
   - Any other value trips `PANIC_DUE_TO_CORRUPT_ENUM` (proxy_filter.cc:111).

`onLoadDnsCacheComplete` (proxy_filter.cc:114) mirrors the `InCache` success path: release the circuit breaker (proxy_filter.cc:117-118), optionally stamp `UpstreamAddress` (proxy_filter.cc:120-122), `readDisable(false)` and `continueReading()` (proxy_filter.cc:124-125).

`addHostAddressToFilterState` (proxy_filter.cc:128) is a no-op unless `save_upstream_address_` is true (proxy_filter.cc:132-134); when enabled, it installs `StreamInfo::UpstreamAddress` under its static key with `StateType::Mutable`, `LifeSpan::Connection` (proxy_filter.cc:139-143).

## Configuration
- `port_value` — upstream port default when no `envoy.upstream.dynamic_port` override is present (proxy_filter.cc:23, 75).
- `dns_cache_config` — `DnsCacheConfig`; resolved through the shared `DnsCacheManager` (proxy_filter.cc:26-28). Governs resolver type, TTL, per-host circuit breaker, key format, etc.
- `save_upstream_address` — when true, write `StreamInfo::UpstreamAddress` into filter state once DNS resolves (proxy_filter.cc:25, 132-143).

Per-connection overrides read from filter state:
- `envoy.upstream.dynamic_host` (`Router::StringAccessor`) — overrides SNI (proxy_filter.cc:36-46).
- `envoy.upstream.dynamic_port` (`StreamInfo::UInt32Accessor`) — overrides `port_value` (proxy_filter.cc:64-73).

## Stats
The filter itself emits no counters. DNS-related metrics (cache size, host lookups, circuit-breaker trips) are reported by the shared `DnsCache` under `dns_cache.<name>.*`; circuit-breaker decisions observed here are the same budget enforced there.

## Factory
`SniDynamicForwardProxyNetworkFilterConfigFactory` (config.h:21) extends `Common::ExceptionFreeFactoryBase<FilterConfig>` with the name `NetworkFilterNames::get().SniDynamicForwardProxy` (config.cc:14-16). `createFilterFactoryFromProtoTyped` (config.cc:19) constructs a `DnsCacheManagerFactoryImpl` bound to the factory context, builds a shared `ProxyFilterConfig` (propagating creation errors via `absl::Status`, config.cc:25-29), and returns a lambda that adds a `ProxyFilter` (sharing the same `ProxyFilterConfig`) to the filter manager (config.cc:31-33). Registration is via `REGISTER_FACTORY` (config.cc:39). Non-terminal by default.
