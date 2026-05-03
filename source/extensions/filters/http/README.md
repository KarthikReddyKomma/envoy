# HTTP Filters — Source Index

This directory contains the source for all HTTP filter extensions. Each
sub-directory is one filter, registered by name in the HTTP connection
manager's `http_filters` list. Per-filter READMEs describe the logic; this
index is a quick map of the critical ones.

## Filter-chain model

- Request decoding runs each filter's `decodeHeaders` / `decodeData` /
  `decodeTrailers` in configured order. Any filter can stop iteration to
  block (auth, buffer, rate limit, external processing) or short-circuit
  with a local reply (fault, cors preflight, rbac deny).
- Response encoding runs `encodeHeaders` / `encodeData` / `encodeTrailers` in
  reverse order.
- The **router** filter is always terminal — it is what actually forwards
  the request upstream.

## Critical filters (documented here)

### Routing
- **[router](router/README.md)** — terminal filter; cluster selection,
  timeouts, retries, hedging, shadowing.
- **[dynamic_forward_proxy](dynamic_forward_proxy/README.md)** — forward to
  arbitrary hosts via per-cluster DNS cache.

### Authentication / authorization
- **[ext_authz](ext_authz/README.md)** — external authorization over gRPC
  or HTTP.
- **[jwt_authn](jwt_authn/README.md)** — JWT validation with JWKS fetch.
- **[oauth2](oauth2/README.md)** — OAuth 2.0 authorization-code flow with
  cookie-based sessions.
- **[rbac](rbac/README.md)** — role-based access control.
- **[csrf](csrf/README.md)** — CSRF origin check.
- **[cors](cors/README.md)** — CORS preflight + response headers.

### Traffic management
- **[ratelimit](ratelimit/README.md)** — global rate limiting via an external
  RLS gRPC service.
- **[local_ratelimit](local_ratelimit/README.md)** — per-Envoy token-bucket
  rate limiting.
- **[fault](fault/README.md)** — delay / abort / response-rate fault
  injection.
- **[health_check](health_check/README.md)** — intercepts health probes.

### Payload handling
- **[buffer](buffer/README.md)** — buffer full request body before
  forwarding.
- **[compressor](compressor/README.md)** — compress request / response
  bodies.
- **[decompressor](decompressor/README.md)** — decompress request / response
  bodies.
- **[cache](cache/README.md)** — RFC 7234 response cache.

### Extensibility / integration
- **[ext_proc](ext_proc/README.md)** — full-duplex gRPC streaming processor.
- **[lua](lua/README.md)** — inline Lua scripting.
- **[grpc_web](grpc_web/README.md)** — gRPC-Web ↔ gRPC transcoding.

## Other filters in this tree

Not documented in detail here; see the respective proto (under
`api/envoy/extensions/filters/http/...`) and source:

- `a2a`, `adaptive_concurrency`, `admission_control`,
  `alternate_protocols_cache`, `api_key_auth`
- `aws_lambda`, `aws_request_signing`
- `bandwidth_limit`, `bandwidth_share`, `basic_auth`
- `cache_v2`, `cdn_loop`, `composite`, `connect_grpc_bridge`
- `credential_injector`, `custom_response`, `dynamic_modules`
- `file_server`, `file_system_buffer`, `gcp_authn`, `geoip`
- `grpc_field_extraction`, `grpc_http1_bridge`, `grpc_http1_reverse_bridge`,
  `grpc_json_reverse_transcoder`, `grpc_json_transcoder`, `grpc_stats`
- `header_mutation`, `header_to_metadata`, `ip_tagging`
- `json_to_metadata`, `kill_request`, `match_delegate`
- `mcp`, `mcp_json_rest_bridge`, `mcp_router`
- `on_demand`, `original_src`, `proto_api_scrubber`,
  `proto_message_extraction`, `rate_limit_quota`
- `set_filter_state`, `set_metadata`, `sse_to_metadata`,
  `stateful_session`, `tap`, `thrift_to_metadata`, `transform`, `wasm`

## Adding documentation

For any new or undocumented filter, a short README should cover:

1. **Purpose** — one paragraph.
2. **Lifecycle** — which decode/encode hooks run, when iteration stops.
3. **Decision logic** — what drives allow / deny / mutate / block.
4. **Configuration** — the shape of the proto.
5. **Stats** — the counters and gauges it emits.
6. **File layout** — entry points with `file.cc:line` references.
