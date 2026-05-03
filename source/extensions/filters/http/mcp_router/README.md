# MCP Router (`envoy.filters.http.mcp_router`)

Terminal HTTP filter that acts as a multi-backend router for MCP JSON-RPC requests. It consumes
the parsed MCP method/params previously written to dynamic metadata by `envoy.filters.http.mcp`,
decodes any composite `mcp-session-id` (session + backend sub-sessions) via `SessionCodec`,
optionally validates the session subject against an authenticated identity, and then, per MCP
method, either fans out to all configured backends (initialize, `tools/list`, `resources/list`,
`prompts/list`, `resources/templates/list`, notifications) aggregating their responses, routes a
single call to the backend implied by a `<backend>__<name>` or `<backend>+<scheme>://` prefix, or
answers locally (`ping`). Responses are emitted as either JSON or SSE depending on the flow.

Proto: `envoy.extensions.filters.http.mcp_router.v3.McpRouter`.

## Files
- `config.h/cc` — `McpRouterFilterConfigFactory`; terminal filter (`config.h:23-26`).
- `filter_config.h/cc` — `McpBackendConfig`, `SessionIdentityConfig`, stats, `McpRouterConfig`
  interface, `McpRouterConfigImpl` (proto-driven) and `McpRouterClusterConfigImpl`
  (cluster-metadata override at request time).
- `mcp_router.h/cc` — `McpRouterFilter`, `McpMethod` enum, per-method handlers, name/URI parsers,
  body rewriters, response aggregators.
- `backend_stream.h/cc` — async upstream stream glue (`BackendStreamCallbacks`) used through
  `Http::MuxDemux` / `MultiStream` to drive one or many concurrent backend HTTP/2 streams.
- `session_codec.h/cc` — `SessionCodec::decode` / `parseCompositeSessionId` — encodes
  `{route, subject, backend_sessions}` into the single `mcp-session-id` header value.

## Lifecycle
- Terminal decoder filter (`config.cc:17`): `addStreamDecoderFilter(std::make_shared<McpRouterFilter>(config))`.
- Constructor creates a `Http::MuxDemux` bound to the factory context (`mcp_router.cc:80`).
- `onDestroy` (`mcp_router.cc:84-92`) resets `multistream_`, callbacks, and aggregation state to
  prevent post-destruction delivery.

## Decode path
- `decodeHeaders` (`mcp_router.cc:114`):
  - Rejects `GET` with 405 (`mcp_router.cc:116-119`).
  - Captures `request_headers_` pointer for later upstream header cloning.
  - Reads cluster metadata via `getClusterConfig()` (`mcp_router.cc:94-112`); when the selected
    cluster carries `envoy.clusters.mcp_multicluster` typed metadata, wraps the base config in
    `McpRouterClusterConfigImpl` so backend list is cluster-derived (`mcp_router.cc:123-126`).
  - Extracts `mcp-session-id` header into `encoded_session_id_`.
  - Rejects end-of-stream (no body) with 400 "Missing request body".
  - Returns `StopIteration` to wait for body.

- `decodeData` (`mcp_router.cc:143`):
  - On first chunk (`initialized_ == false`):
    - inc `rq_total`.
    - `readMetadataFromMcpFilter` (`mcp_router.cc:264-338`) pulls the dynamic metadata struct
      under `config_->metadataNamespace()` and populates `method_`, `request_id_`, and one of
      `tool_name_` / `resource_uri_` / `prompt_name_` / (`completion_ref_type_` +
      `prompt_name_`/`resource_uri_`) depending on method. Missing/unknown method ⇒
      inc `rq_invalid`, 400.
    - If session id present, `decodeAndParseSession` (`mcp_router.cc:340-366`) decodes it via
      `SessionCodec`, populates `route_name_`, `session_subject_`, `backend_sessions_`; failures
      inc `rq_session_invalid`, 400. `validateSubjectIfRequired` (`mcp_router.cc:400-423`) — when
      `shouldEnforceValidation()` — resolves the authenticated subject from header or metadata
      via `getAuthenticatedSubject` (`mcp_router.cc:368-398`) and 403s on mismatch, incrementing
      `rq_auth_failure`.
    - Dispatches to `handleInitialize` / `handleToolsList` / `handleToolsCall` /
      `handleResources*` / `handlePrompts*` / `handleCompletionComplete` /
      `handleLoggingSetLevel` / `handlePing` / `handleNotification(...)`
      (`mcp_router.cc:162-231`). Unknown ⇒ inc `rq_invalid`, 400.
    - If `needs_body_rewrite_`, inc `rq_body_rewrite` and run the matching rewriter
      (`rewriteToolCallBody` / `rewriteResourceUriBody` / `rewritePromptsGetBody` /
      `rewriteCompletionCompleteBody`) on `data` to strip the backend prefix so the upstream
      sees the canonical name/URI (`mcp_router.cc:237-249`).
  - `streamData(data, end_stream)` — pushes the (possibly rewritten) buffer into the established
    `multistream_` so each backend stream receives identical bytes; always returns
    `StopIterationNoBuffer` (`mcp_router.cc:252-254`).

- `decodeTrailers` (`mcp_router.cc:257`): multicasts trailers via `multistream_->multicastTrailers`.

## SSE stream handler (upstream → downstream)
`McpRouterFilter` also implements `SseStreamHandler` (`mcp_router.h:56-61`):
- `pushSseHeaders` emits the response status/headers the first time a backend produces them.
- `pushSseData` forwards raw SSE bytes downstream.
- `pushSseEvent` (per-backend event_type) is used for mux/demux aggregation of `message` events
  from multiple backends when a single logical reply is needed.
- `onStreamingError` / `onStreamingComplete` terminate the downstream stream and update stats.

## Decision / logic
- Backend selection:
  - `parseToolName` (`mcp_router.cc:425`): splits on `__`; verifies prefix against
    `findBackend`; falls back to `defaultBackendName()` when not multiplexing or prefix absent.
  - `parseResourceUri` (`mcp_router.cc:448`): format `<backend>+<scheme>://<path>`; rewrites URI to
    `<scheme>://<path>` for the backend when prefix resolves.
  - `parsePromptName` (`mcp_router.cc:490`): same `__` convention as tools.
- Aggregation helpers (`aggregateInitialize`, `aggregateToolsList`, `aggregateResourceItems`,
  `aggregateResourcesList`, `aggregateResourcesTemplatesList`, `aggregatePromptsList`) merge JSON
  result arrays across backends while preserving `jsonrpc`/`id` envelope (`mcp_router.h:117-123`).
- `extractJsonRpcFromResponse` strips SSE wrapping around a JSON-RPC payload so aggregation works
  whether backend replied with `application/json` or `text/event-stream` (`mcp_router.h:125`).
- `initializeFanout` triggers `rq_fanout`; `initializeSingleBackend` has streaming/non-streaming
  variants for SSE (`mcp_router.h:128-132`).
- `copyRequestHeaders` (`mcp_router.cc:36-48`) forwards downstream headers to each backend except
  the skip-list (`:method`, `:path`, `:authority`, `host`, `content-type`, `accept`,
  `mcp-session-id`).
- Constants pinned: `kGatewayName = envoy-mcp-gateway`, `kGatewayVersion = 1.0.0`,
  `kProtocolVersion = 2025-06-18` (`mcp_router.cc:29-31`).

## Configuration
- `backends[]`: repeated `{name, cluster, path, timeout (default 5s), host_rewrite_literal}`
  (`filter_config.h:24-30`).
- `default_backend_name`: used when multiplexing and the request doesn't carry a backend prefix.
- Session identity (`filter_config.h:32-55`): subject source is either a header (`HeaderSubjectSource`)
  or a dynamic-metadata path (`MetadataSubjectSource`); `validation_mode ∈ {Disabled, Enforce}`.
- `metadataNamespace()` — the dynamic-metadata namespace written by the `mcp` filter.
- `McpRouterClusterConfigImpl` (`filter_config.h:136-158`) overrides `backends[]` and
  `default_backend_name` from per-cluster typed metadata at request time while preserving the base
  session-identity config.

## Factory
- `McpRouterFilterConfigFactory` registered as `envoy.filters.http.mcp_router`
  `NamedHttpFilterConfigFactory` (`config.cc:23`); declared terminal via
  `isTerminalFilterByProtoTyped => true` (`config.h:23-26`).

## Stats
Prefix: `<stat_prefix>` (set by the HCM), counters defined by `MCP_ROUTER_STATS`
(`filter_config.h:60-72`):
- `rq_total` — every dispatched MCP request.
- `rq_fanout` — requests broadcast to all backends.
- `rq_direct_response` — answered locally (`ping`, initialize short-circuit, aggregated).
- `rq_body_rewrite` — body mutated to strip backend prefix from name/URI.
- `rq_invalid` — bad metadata / unsupported method.
- `rq_unknown_backend` — resolved prefix not in `backends`.
- `rq_backend_failure` — individual backend errored.
- `rq_fanout_failure` — fanout aggregate failed.
- `rq_session_invalid` — bad session-id decode/parse.
- `rq_auth_failure` — enforced subject mismatch / no subject resolvable.
