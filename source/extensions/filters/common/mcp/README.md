# MCP Shared Constants and Filter State (shared filter infrastructure)

Header-only library shared by the Model Context Protocol (MCP) filter family - the HTTP MCP filter, the MCP JSON/REST bridge, and the MCP router. It centralizes the protocol version strings, JSON-RPC field names, method/method-group names, JSON extraction paths, and the `FilterStateObject` that carries a parsed MCP request from an earlier filter (`mcp`) to later filters (`mcp_json_rest_bridge`, `mcp_router`) without reparsing the body.

## Files
- `constants.h` - All `constexpr absl::string_view` constants. Three nested namespaces under `McpConstants`: `Methods` (wire-level method identifiers), `MethodGroups` (rollup labels for metrics/routing), and `Paths` (JSON Pointer-style paths for attribute extraction).
- `filter_state.h` - `FilterStateObject` inheriting `StreamInfo::FilterState::Object`, plus `metadataNamespace()` runtime-feature shim.

## Public interface
- `McpConstants::McpfilterNamespace` = `"envoy.filters.http.mcp"` and `McpConstants::LegacyMetadataNamespace` = `"mcp_proxy"` (`constants.h:16`).
- JSON-RPC field keys: `JSONRPC_FIELD`, `METHOD_FIELD`, `ID_FIELD`, `RESULT_FIELD`, `PARAMS_FIELD`, `ARGUMENTS_FIELD`, `ERROR_CODE_FIELD`, `ERROR_MESSAGE_FIELD`, content envelope: `TYPE_FIELD`, `TEXT_FIELD`, `CONTENT_FIELD`, `IS_ERROR_FIELD`, `ERROR_FIELD`, and `JSONRPC_VERSION` = `"2.0"` (`constants.h:20`).
- Initialize negotiation: `LATEST_SUPPORTED_MCP_VERSION`, `FALLBACK_PROTOCOL_VERSION`, `MCP_VERSION_2024_11_05`, `MCP_VERSION_2025_06_18`, plus server-info fields `NAME_FIELD`, `VERSION_FIELD`, `DEFAULT_SERVER_VERSION` (`constants.h:37`).
- HTTP headers: `MCP_SESSION_ID_HEADER` = `"mcp-session-id"`, `MCP_PROTOCOL_VERSION_HEADER` = `"mcp-protocol-version"` (`constants.h:56`).
- `IS_MCP_REQUEST` dynamic-metadata key (`constants.h:53`) written by the MCP filter so downstream filters can branch on "is this actually an MCP request."
- `McpConstants::Methods::*` (`constants.h:60`) - exhaustive list of wire method names (`tools/call`, `tools/list`, `resources/read`, `prompts/get`, `sampling/createMessage`, `initialize`, `ping`, and all `notifications/*`).
- `McpConstants::MethodGroups::*` (`constants.h:107`) - coarse classifications (`lifecycle`, `tool`, `resource`, `prompt`, `notification`, `logging`, `sampling`, `completion`, `unknown`).
- `McpConstants::Paths::*` (`constants.h:120`) - JSON paths consumed by attribute extractors: `params.name`, `params.uri`, `params.level`, `params.ref`, `params.protocolVersion`, `params.clientInfo.name`, `params.requestId`, `params.progressToken`, `params.progress`, `params._meta`.
- `FilterStateObject` (`filter_state.h:27`):
  - `FilterStateKey` = `"envoy.filters.http.mcp.request"`.
  - Constructed from either `(method, Json::ObjectSharedPtr, is_mcp_request)` or `(method, const Protobuf::Struct&, is_mcp_request)` - the proto-struct overload loads it through `Json::Factory::loadFromProtobufStruct`.
  - Accessors: `method()` returning `absl::optional<absl::string_view>` (nullopt for empty), `json()` for the parsed body, `isMcpRequest()`.
  - `serializeAsString()` emits the JSON body so access loggers can record it; returns nullopt when empty.
- `metadataNamespace()` (`filter_state.h:17`) returns `McpfilterNamespace` when `envoy.reloadable_features.mcp_filter_use_new_metadata_namespace` is enabled, otherwise `LegacyMetadataNamespace` - handy shim for migrating filter-metadata consumers.

## Implementation logic
- All constants are `constexpr absl::string_view`, giving zero-cost string sharing and letting filters compare with `==` rather than `strcmp`.
- `FilterStateObject` stores the JSON body as an `Json::ObjectSharedPtr` so multiple downstream filters can re-inspect it without a clone. The proto-struct constructor defers to `Json::Factory::loadFromProtobufStruct` which performs structural validation.
- `serializeAsString()` short-circuits when the object is null or empty - empty returns `absl::nullopt`, which tells the access logger framework to omit the field rather than emit `""`.
- `metadataNamespace()` is inline header-only and cheap enough to call on the request hot path; it reads the runtime feature each time so flag flips propagate to new requests without restart.

## Consumers
- HTTP filters:
  - `source/extensions/filters/http/mcp` - primary producer: parses the JSON-RPC body, classifies method, writes the `FilterStateObject` under `FilterStateKey` and sets the `is_mcp_request` dynamic-metadata bit.
  - `source/extensions/filters/http/mcp_json_rest_bridge` - consumer: reads the filter state to know which tool/resource call to translate into a REST upstream request.
  - `source/extensions/filters/http/mcp_router` - consumer: routes on `Methods::*` and `Paths::*` values plus the `mcp-session-id` header.

## Stats / errors / failure modes
- No stats. No errors emitted from this library.
- `FilterStateObject` never throws - malformed JSON is handled by the factory that constructs it (the `mcp` filter), and an empty `json_` simply yields `nullopt` from `serializeAsString()`.
- Consumers must defensively handle both legacy and new metadata namespaces during the `mcp_filter_use_new_metadata_namespace` rollout by calling `metadataNamespace()` instead of hard-coding the literal.
- `FilterStateKey` is not a runtime-flagged name; changing it would silently break downstream filters that pull their input from the filter state chain.
