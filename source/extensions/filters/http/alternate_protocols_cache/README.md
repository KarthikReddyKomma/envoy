# Alternate Protocols Cache (`envoy.filters.http.alternate_protocols_cache`)

Encoder-only filter that scans upstream responses for the RFC 7838 `Alt-Svc` header and records the advertised alternate protocols (typically HTTP/3) in Envoy's `HttpServerPropertiesCache`. Subsequent requests to the same upstream host can then upgrade transparently to the advertised protocol.

Proto: `envoy.extensions.filters.http.alternate_protocols_cache.v3.FilterConfig`.

## Files
- `filter.h/cc` — `Filter` (encoder-only) and `FilterConfig`.
- `config.h/cc` — `AlternateProtocolsCacheFilterFactory`, registered as `envoy.filters.http.alternate_protocols_cache`.

## Lifecycle
Extends `Http::PassThroughEncoderFilter` (`filter.h:39`); there is no decode-side work at all.

- `encodeHeaders(headers, end_stream)` (`filter.cc:35`) — the only real override.
  - Looks up `Alt-Svc` (`CustomHeaders::get().AltSvc`). If absent, `Continue` (`filter.cc:36-39`).
  - Resolves the per-cluster `HttpServerPropertiesCacheOptions` via `upstreamClusterInfo()->httpProtocolOptions().alternateProtocolsCacheOptions()`; if the cluster did not opt in, `Continue` without writing (`filter.cc:41-50`).
  - Parses each `Alt-Svc` header value with `HttpServerPropertiesCacheImpl::alternateProtocolsFromString`. A single unparseable value aborts the update for this response (`Continue`, nothing cached) (`filter.cc:53-64`).
  - Derives the origin: scheme `https`, hostname taken from `upstreamHost()->hostname()` unless the upstream TLS session has a non-empty SNI (SNI wins), and port from `host->address()->ip()->port()` defaulting to 443 (`filter.cc:70-82`).
  - Writes `cache->setAlternatives(origin, protocols)` for the upstream host (`filter.cc:83`). Note the cache key is the upstream host, not `:authority`, so a load-balanced cluster can have per-host protocol negotiation state (`filter.cc:66-69`).
- `onDestroy()` is an empty override (`filter.cc:30`).

## Decision / logic
- Fast exit when the response lacks `Alt-Svc` (`filter.cc:36-39`).
- Fast exit when the upstream cluster has no `alternateProtocolsCacheOptions` — this is how clusters opt out (`filter.cc:48-50`).
- SNI overrides configured hostname when both are available (`filter.cc:73-78`). This matches the cache key used by HTTP/3 establishment.
- Port falls back to 443 when the upstream host description has no address (`filter.cc:80`).
- The deprecated `alternate_protocols_cache_options` proto field emits a `warn` log and is otherwise ignored (`filter.cc:24-27`).

## Configuration
- Proto `FilterConfig` has one deprecated field (`alternate_protocols_cache_options`) that is warned about and ignored.
- Actual configuration lives on the cluster: `Cluster.typed_extension_protocol_options` `HttpProtocolOptions.alternate_protocols_cache_options`.
- No per-route configuration.

## Stats
None. The filter itself does not emit stats; the underlying `HttpServerPropertiesCache` has its own accounting.

## Factory
`AlternateProtocolsCacheFilterFactory` implements both the normal and `ServerContext`-only factory methods (`config.cc:14`, `config.cc:31`). Each constructs a `FilterConfig` with the server-wide `HttpServerPropertiesCacheManager` and the main-thread dispatcher's time source, then returns a callback that installs an encoder-only filter bound to the worker dispatcher via `addStreamEncoderFilter` (`config.cc:25-28`, `config.cc:41-44`). The dispatcher is passed in so the filter can hand it to `cache_manager.getCache(...)` which may need to schedule thread-local work. Registered via `REGISTER_FACTORY(AlternateProtocolsCacheFilterFactory, NamedHttpFilterConfigFactory)` (`config.cc:50`).
