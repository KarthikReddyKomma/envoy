# Header Mutation (`envoy.filters.http.header_mutation`)

A stream filter that applies formatter-driven mutations (add / set / remove with substitution support) to request headers, response headers, request trailers, response trailers, and URL query parameters. It is a `PassThroughFilter` and supports per-route overrides at route, virtual-host, and route-table scope with configurable specificity ordering. The same factory is registered for both the downstream HTTP filter chain and the upstream HTTP filter chain.

Proto: `envoy.extensions.filters.http.header_mutation.v3.HeaderMutation` (and `HeaderMutationPerRoute`).

## Files
- `config.h/cc` — `HeaderMutationFactoryConfig` (a `DualFactoryBase`) that builds the shared config and registers both the downstream and upstream named-HTTP-filter factories (`config.cc:53-55`). Also produces the per-route config.
- `header_mutation.h/cc` — `Mutations` (header/trailer/query-param mutation engine), `HeaderMutationConfig` (global config), `PerRouteHeaderMutation` (route-specific config), and the `HeaderMutation` stream filter.

## Lifecycle
- `HeaderMutation` extends `Http::PassThroughFilter` (`header_mutation.h:135-158`). The filter chain attaches it as a stream filter so it runs in both decode and encode paths.
- `decodeHeaders` (`header_mutation.cc:185-197`): builds a `Formatter::Context` with the request headers and active tracing span, runs the global request-header/query-param mutations, then lazily initializes the ordered list of per-route configs and applies each.
- `encodeHeaders` (`header_mutation.cc:199-215`): builds a context with cached request headers and the response headers, applies global response-header mutations, then re-attempts route-config initialization (covers the case where an earlier filter sent a local reply so `decodeHeaders` never ran) and applies per-route response mutations.
- `decodeTrailers` (`header_mutation.cc:235-249`): applies global and per-route request-trailer mutations. (`Context` does not currently carry request trailers — TODO at `header_mutation.cc:236-237`.)
- `encodeTrailers` (`header_mutation.cc:217-233`): applies global and per-route response-trailer mutations with a context containing request headers, response headers, and the trailer map.

All four callbacks return `Continue` / `Continue` — the filter never blocks the stream.

## Decision / logic
- Route-config discovery is memoized: `maybeInitializeRouteConfigs` (`header_mutation.cc:159-183`) guards with `route_configs_initialized_` so the same list is used across decode and encode even when empty. It calls `Http::Utility::getAllPerFilterConfig<PerRouteHeaderMutation>` which returns configs ordered route-table -> virtual-host -> route.
- Specificity control: if `most_specific_header_mutations_wins` is false the vector is reversed (`header_mutation.cc:180-182`) so earlier entries win over later evaluation; otherwise the default evaluation order already lets the most-specific config run last.
- Request header mutation also rewrites the `:path` query string only when query mutations exist and a path header is present (`header_mutation.cc:117-128`).
- `QueryParameterMutationAppend::mutateQueryParameter` (`header_mutation.cc:26-53`) switches over `KeyValueAppendAction`: `APPEND_IF_EXISTS_OR_ADD`, `ADD_IF_ABSENT`, `OVERWRITE_IF_EXISTS`, `OVERWRITE_IF_EXISTS_OR_ADD`, with URL-encoding gated by runtime flag `envoy.reloadable_features.header_mutation_url_encode_query_params` (`header_mutation.cc:17-23`).
- `Mutations` validation (`header_mutation.cc:77-109`) enforces: exactly one of `append`/`remove`, an `append.record` with a string value; the record value compiles through `Formatter::FormatterImpl::create`.

## Configuration
- `mutations` — grouped `request_mutations`, `response_mutations`, `request_trailers_mutations`, `response_trailers_mutations`, and `query_parameter_mutations` (built by `Mutations` ctor, `header_mutation.cc:55-110`). Header mutations are wrapped by `Http::HeaderMutations` (a formatter-backed engine in `source/common/http/header_mutation.h`).
- `most_specific_header_mutations_wins` — controls evaluation order of per-route configs (see above).
- Per route: `HeaderMutationPerRoute.mutations` — same `Mutations` message (`header_mutation.h:106-117`). Configs are collected at route-table, virtual-host, and route levels and all are applied.

## Stats
None. The filter does not emit counters or gauges; the stats prefix passed into the factory is unused.
