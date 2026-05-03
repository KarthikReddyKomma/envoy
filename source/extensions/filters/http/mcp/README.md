# MCP (`envoy.filters.http.mcp`)

Validates, classifies, and annotates Model Context Protocol (MCP) HTTP traffic before it reaches
the upstream. Accepts MCP DELETE (session terminate), SSE GET, and JSON-RPC 2.0 POST; for POSTs it
streams the body through a bounded-size JSON path parser, extracts metadata (method, id, params),
can inject trace-context/baggage from `params._meta` into request headers, and either passes the
stream through or rejects non-MCP traffic based on the configured `TrafficMode`. Parsed metadata
can be propagated downstream via dynamic metadata and/or filter state so later filters
(e.g., `mcp_router`) can consume it.

Proto: `envoy.extensions.filters.http.mcp.v3.Mcp` (+ `McpOverride` per-route).

## Files
- `config.h/cc` — `McpFilterConfigFactory` (downstream registration and per-route config).
- `mcp_filter.h/cc` — `McpFilter`, `McpFilterConfig`, `McpOverrideConfig`, stats struct,
  trace/baggage injection helpers.
- `mcp_json_parser.h/cc` — streaming JSON path parser used to extract JSON-RPC fields incrementally
  from the body buffer (owned by the filter; not a public interface).

## Lifecycle
- Built as a stream filter via `callbacks.addStreamFilter(std::make_shared<McpFilter>(config))`
  (`config.cc:20`, `config.cc:31`). Subclass of `Http::PassThroughFilter`
  (`mcp_filter.h:130`), so only `decodeHeaders`, `decodeData`, and `setDecoderFilterCallbacks` are
  overridden.
- `McpFilterConfig` constructor (`mcp_filter.cc:92-96`) reads `traffic_mode`, `clear_route_cache`,
  trace/baggage propagation configs, `max_request_body_size` (default 8192 bytes),
  `request_storage_mode`, the static metadata namespace, and builds a `ParserConfig` from
  `parser_config` (or default). Stats are generated under `<prefix>mcp.` (`mcp_filter.cc:43-46`).
- `McpOverrideConfig` (`mcp_filter.h:104-123`) exposes per-route `traffic_mode` and
  `max_request_body_size`.

## Decode path
- `decodeHeaders` (`mcp_filter.cc:191`):
  1. `isValidMcpDeleteRequest` (`mcp_filter.cc:98`): DELETE + non-empty `Mcp-Session-Id` header
     ⇒ set `is_mcp_request_ = true`, `Continue` (`mcp_filter.cc:192-196`).
  2. `isValidMcpSseRequest` (`mcp_filter.cc:106`): GET + Accept contains `text/event-stream` or
     `*/*` ⇒ `is_mcp_request_ = true`, `Continue` (`mcp_filter.cc:198-202`).
  3. `isValidMcpPostRequest` (`mcp_filter.cc:128`): POST with `Content-Type: application/json` and
     an Accept set containing both `application/json` and `text/event-stream` (or `*/*`). If
     `end_stream` is true here (POST with no body), treated as not-MCP; otherwise sets
     `is_json_post_request_ = true`, applies `decoder_callbacks_->setBufferLimit(max_size)` and
     returns `StopIteration` to buffer the body (`mcp_filter.cc:204-222`).
  4. Else if not MCP and `shouldRejectRequest()` returns true, increments `requests_rejected` and
     calls `sendLocalReply(400, "Only MCP traffic is allowed", ...)` with response-code-details
     `mcp_filter_reject_no_mcp` (`mcp_filter.cc:225-230`).
  5. Otherwise pass through with `Continue`.

- `decodeData` (`mcp_filter.cc:236`):
  - Bails out with `Continue` if the request is not a JSON POST MCP candidate.
  - Lazily constructs `JsonPathParser` from `config_->parserConfig()` (`mcp_filter.cc:241-243`).
  - If `parsing_complete_` already set ⇒ `Continue`.
  - Iterates the raw slices in `data` up to the (possibly capped) byte budget, calling
    `parser_->parse(...)` per slice; bumps `bytes_parsed_` (`mcp_filter.cc:253-275`).
  - Early-exit when `parser_->isAllFieldsCollected()` returns true — calls `completeParsing()`
    (`mcp_filter.cc:265-268`).
  - On parse error: inc `invalid_json` and `sendLocalReply(400, "not a valid JSON", ...)` with
    details `mcp_filter_not_jsonrpc` (`mcp_filter.cc:270-274`).
  - On `end_stream` or size-limit hit: `parser_->finishParse()`. If that fails but all required
    fields are present and only optional fields are missing, proceeds with a partial parse;
    otherwise inc `body_too_large` and `handleParseError` (`mcp_filter.cc:282-295`).
  - Default when body not complete: `StopIterationAndWatermark` (`mcp_filter.cc:297`).

- `completeParsing` (`mcp_filter.cc:308`):
  - Sets `parsing_complete_ = true` and `is_mcp_request_ = parser_->isValidMcpRequest()`.
  - If still not MCP and `shouldRejectRequest()` ⇒ 400 with details `mcp_filter_not_jsonrpc`
    (`mcp_filter.cc:314-317`).
  - Pulls `Protobuf::Struct metadata = parser_->metadata()`.
  - If `parser_config.group_metadata_key` is set, writes the resolved method-group onto metadata
    (`mcp_filter.cc:321-326`).
  - If trace-context / baggage propagation is enabled, reads the `params._meta` struct via
    `parser_->getNestedValue(Paths::PARAMS_META)` and calls `injectTraceContext` /
    `injectBaggage` against request headers (`mcp_filter.cc:329-341`). Trace fields are validated
    with `Envoy::Tracing::isValidTraceParent/State/Baggage` before insertion
    (`mcp_filter.cc:54-89`).
  - If metadata non-empty:
    - `shouldStoreToFilterState()` ⇒ sets a read-only request-lifespan `FilterStateObject` holding
      method, struct, and the `is_mcp_request_` flag (`mcp_filter.cc:344-347`).
    - `shouldStoreToDynamicMetadata()` ⇒ writes the struct (with `is_mcp_request` added) into
      dynamic metadata under `config_->metadataNamespace()` (`mcp_filter.cc:349-353`).
    - `config_->clearRouteCache()` ⇒ `downstreamCallbacks()->clearRouteCache()`
      (`mcp_filter.cc:355-360`).

## Decision / logic
- Content-Type JSON acceptance is exact, tolerating only `;` / space after `application/json`
  to exclude `application/json-patch+json` (`mcp_filter.cc:133-136`).
- `shouldRejectRequest` / `getMaxRequestBodySize` resolve per-route `McpOverrideConfig` via
  `resolveMostSpecificPerFilterConfig` before falling back to the filter config
  (`mcp_filter.cc:171-189`).
- Storage mode (`DYNAMIC_METADATA` / `FILTER_STATE` / combined / unspecified) gates where the
  parsed metadata is emitted (`mcp_filter.h:69-81`).

## Configuration
- `traffic_mode`: `REJECT_NO_MCP` vs permissive (pass through non-MCP).
- `max_request_body_size` (default 8192).
- `request_storage_mode`: `DYNAMIC_METADATA`, `FILTER_STATE`, or both.
- `propagate_trace_context`, `propagate_baggage`: toggle W3C traceparent/tracestate/baggage
  injection from `params._meta`.
- `clear_route_cache`: force route recomputation after metadata emission.
- `parser_config`: custom `ParserConfig` (group metadata key, etc.).
- Per-route: `McpOverride.traffic_mode`, `McpOverride.max_request_body_size`.

## Factory
- `McpFilterConfigFactory` registered as `envoy.filters.http.mcp`
  `NamedHttpFilterConfigFactory` (`config.cc:45`). Supports both factory- and server-context
  constructors (`config.cc:12-33`). Per-route via
  `createRouteSpecificFilterConfigTyped` returning `McpOverrideConfig` (`config.cc:35-40`).

## Stats
Prefix: `<stat_prefix>mcp.`
- `requests_rejected` — counter, non-MCP traffic dropped by `REJECT_NO_MCP`.
- `invalid_json` — counter, body-parse error.
- `body_too_large` — counter, reached size/end-stream without all required fields.
