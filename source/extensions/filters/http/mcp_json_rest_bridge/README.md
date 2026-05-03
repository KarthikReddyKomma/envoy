# MCP JSON REST Bridge (`envoy.filters.http.mcp_json_rest_bridge`)

Transcodes MCP JSON-RPC 2.0 requests received at `/mcp` into REST/JSON HTTP calls to a backend API
and translates the backend's response back into a JSON-RPC envelope. Handles the MCP initialize
handshake locally, forwards `tools/list` to a configured GET endpoint and wraps its body as a
JSON-RPC `result`, and maps each `tools/call` to a per-tool `HttpRule` (path/method/body) via
`HttpRequestBuilder`. Non-`/mcp` traffic is passed through unchanged.

Proto: `envoy.extensions.filters.http.mcp_json_rest_bridge.v3.McpJsonRestBridge`.

## Files
- `config.h/cc` — `McpJsonRestBridgeFilterConfigFactory`; validates that any configured
  `tool_list_http_rule` is a GET with no body.
- `mcp_json_rest_bridge_filter.h/cc` — `McpJsonRestBridgeFilterConfig`, `McpJsonRestBridgeFilter`,
  MCP response builders, response-code helpers, protocol-version validation.
- `http_request_builder.h/cc` — translates `HttpRule` + JSON `arguments` into a concrete
  `HttpRequest{url, method, body}`.

## Lifecycle
- Factory produces `McpJsonRestBridgeFilter` via `addStreamFilter` (`config.cc:30-32`). Filter is
  a `PassThroughFilter` subclass (`mcp_json_rest_bridge_filter.h:50`) overriding
  `decodeHeaders`, `decodeData`, `encodeHeaders`, `encodeData`, `encodeTrailers`.
- `McpJsonRestBridgeFilterConfig` constructor (`mcp_json_rest_bridge_filter.cc:122-128`)
  indexes the `tool_config.tools()` repeated field into `tool_to_http_rule_` keyed by tool name,
  and caches `fallback_protocol_version_` from `server_info.fallback_protocol_version`.

## Decode path
- `decodeHeaders` (`mcp_json_rest_bridge_filter.cc:145`):
  - Strips query string from `:path`. If path != `/mcp`, returns `Continue` — filter is inert for
    non-MCP routes (`mcp_json_rest_bridge_filter.cc:146-155`).
  - Sets `mcp_operation_ = Undecided` and captures `:authority` as `server_name_` for the
    initialize response.
  - If method isn't POST, sends `405 Method Not Allowed` with `Allow: POST` and response-code
    details `mcp_json_rest_bridge_filter_not_post` (`mcp_json_rest_bridge_filter.cc:161-167`).
  - Always returns `StopIteration` for `/mcp` — the filter must buffer the body before upstream
    routing decisions can proceed.

- `decodeData` (`mcp_json_rest_bridge_filter.cc:172`):
  - `Unspecified` state (non-`/mcp` traffic) ⇒ `Continue`.
  - Accumulates chunks into `request_body_`; waits for `end_stream`.
  - On `end_stream`, linearizes the buffer and calls `nlohmann::json::parse(...,
    allow_exceptions=false)`. Discard ⇒ `sendErrorResponse(400, ...,
    generateErrorJsonResponse(-32700, "JSON parse error"))` (`mcp_json_rest_bridge_filter.cc:187-194`).
  - Dispatches `handleMcpMethod(json_rpc, request_headers)` and replaces the upstream body with
    `request_body_str_` (the bytes chosen by the dispatcher)
    (`mcp_json_rest_bridge_filter.cc:196-198`).
  - Returns `StopIterationNoBuffer` when the operation is handled locally (initialize /
    initialized-ack / failed); else `Continue` so the rewritten request flows upstream.

- `handleMcpMethod` (`mcp_json_rest_bridge_filter.cc:254`):
  - `validateJsonRpcIdAndMethod` (`mcp_json_rest_bridge_filter.cc:455`): sets `session_id_` from
    the request `id`; emits `-32601` (missing/non-string method) or `-32600` (missing id, unless
    it's `notifications/initialized`) as appropriate.
  - `validateRequestMcpVersion` (`mcp_json_rest_bridge_filter.cc:103`): checks
    `MCP-Protocol-Version` header (or fallback) against the supported set
    (`mcp_json_rest_bridge_filter.cc:33-41`); initialize requests are exempt.
  - Method dispatch (`mcp_json_rest_bridge_filter.cc:266-313`):
    - `tools/list`: if `ToolConfig.tool_list_http_rule` is set, rewrite `:path` to `rule.get()`,
      `:method = GET`, strip body headers, set `Accept-Encoding: identity`, `clearRouteCache()`.
      Otherwise `mcp_operation_ = Unspecified` and forward the body unchanged.
    - `initialize`: sends a local reply with `generateInitializeResponse` (negotiates protocol
      version against the supported set — `mcp_json_rest_bridge_filter.cc:65-83`).
    - `notifications/initialized`: sends local 202 Accepted.
    - `tools/call`: `mapMcpToolToApiBackend` (see below).
    - Else: `-32601` "Method … is not supported".

- `mapMcpToolToApiBackend` (`mcp_json_rest_bridge_filter.cc:384`):
  - Validates `params` (object), `params.name` (string), and `params.arguments` (object if
    present) — returns `-32602` on each failure
    (`mcp_json_rest_bridge_filter.cc:384-413`).
  - Looks up `HttpRule` via `config_->getHttpRule(tool_name)`; missing ⇒ `-32602` "Unknown tool".
  - `buildHttpRequest(*http_rule, arguments)` ⇒ fills `request_body_str_`, rewrites `:path`,
    `:method`, content-length/chunked, sets `Content-Type: application/json`,
    `Accept-Encoding: identity`, then `clearRouteCache()` so the new path re-routes
    (`mcp_json_rest_bridge_filter.cc:418-446`).

## Encode path
- `encodeHeaders` (`mcp_json_rest_bridge_filter.cc:207`): for `Unspecified`, `Undecided`,
  `Initialization`, `InitializationAck` ⇒ `Continue` (nothing to rewrite). Otherwise returns
  `StopIteration` unless this is an already-terminal headers-only response (a TODO notes headers-
  only upstream responses aren't fully handled yet).
- `encodeData` (`mcp_json_rest_bridge_filter.cc:228`): buffers upstream response into
  `response_body_`. On `end_stream`, calls `encodeJsonRpcData` and replaces the response body
  with the rewritten JSON-RPC bytes (`mcp_json_rest_bridge_filter.cc:228-243`).
- `encodeJsonRpcData` (`mcp_json_rest_bridge_filter.cc:316`):
  - `ToolsList`: parse upstream body; on parse failure or HTTP >= 400 emit a JSON-RPC `error`
    `-32000 "Server error"`; else wrap as `{jsonrpc, id, result: tools}`.
  - `ToolsCall`: check UTF-8 with `utf8_range::IsStructurallyValid`; build
    `{jsonrpc, id, result:{content:[{type:"text", text:body}], isError:(status>=400)}}`
    via `translateJsonRestResponseToJsonRpc` (`mcp_json_rest_bridge_filter.cc:53-63`).
  - `OperationFailed`: parse the previously emitted error body and rewrap under `error` field with
    the original session id (`mcp_json_rest_bridge_filter.cc:352-366`).
  - Adjusts `Content-Length` vs chunked, forces `Content-Type: application/json`
    (`mcp_json_rest_bridge_filter.cc:372-381`).
- `encodeTrailers` (`mcp_json_rest_bridge_filter.cc:245`): pass-through; transcoding with trailers
  is documented as unsupported.

## Decision / logic
- Supported protocol versions: `LATEST_SUPPORTED_MCP_VERSION`, `FALLBACK_PROTOCOL_VERSION`,
  `MCP_VERSION_2024_11_05`, `MCP_VERSION_2025_06_18` (`mcp_json_rest_bridge_filter.cc:33-41`).
- `McpOperation` enum (`mcp_json_rest_bridge_filter.h:79-93`) drives the encode-side switch; any
  failure path calls `sendErrorResponse` which sets `mcp_operation_ = OperationFailed`
  (`mcp_json_rest_bridge_filter.cc:449-453`).

## Configuration
- `tool_config.tools[]`: `name` + `HttpRule` — used for `tools/call`.
- `tool_config.tool_list_http_rule`: must be GET with empty body (validated in
  `config.cc:19-26`); rewritten target for `tools/list`.
- `server_info.fallback_protocol_version`: used when request lacks `MCP-Protocol-Version` header.
- No per-route config.

## Factory
- `McpJsonRestBridgeFilterConfigFactory` registered as
  `envoy.filters.http.mcp_json_rest_bridge` (`config.h:21`, `config.cc:38-39`). Uses
  `ExceptionFreeFactoryBase` so construction returns `absl::StatusOr`.

## Stats
- No stats emitted. Errors are surfaced via `ENVOY_STREAM_LOG` and `sendLocalReply`
  response-code-details (`mcp_json_rest_bridge_filter_*`).
