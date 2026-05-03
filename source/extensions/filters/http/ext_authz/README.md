# External Authorization Filter (`envoy.filters.http.ext_authz`)

Delegates the allow/deny decision for every request to an external
authorization service over **gRPC** or **HTTP**. The filter pauses the request,
asks the service, then either forwards, rewrites, denies, or fails-open based
on the response.

Proto: `envoy.extensions.filters.http.ext_authz.v3.ExtAuthz`.

## Transport

- **gRPC** — `envoy.service.auth.v3.Authorization/Check`. Default timeout 200 ms
  (`grpc_service.timeout`). Client built at `config.cc:65`.
- **HTTP** — raw HTTP request to `http_service.server_uri`. Default timeout
  200 ms. Client built at `config.cc:41`.

## Request lifecycle

All line references are `ext_authz.cc` unless noted.

1. **`decodeHeaders()` (line 288)**
   - Honours per-route `disabled` / `deny_at_disable`.
   - If `with_request_body` is configured, sets `buffer_data_` and returns
     `StopIteration` so the body can accumulate.
2. **`decodeData()` (line 343)**
   - Accumulates until `end_stream` or the buffer hits `max_request_bytes_`
     (line 345).
   - `allow_partial_message=false` + overflow → reject before calling out;
     otherwise calls `initiateCall()`.
3. **`decodeTrailers()` (line 360)** — flushes buffering if still pending.
4. **`initiateCall()` (line 191)**
   - Builds the `CheckRequest` via
     `CheckRequestUtils::createHttpCheck()` (line 272).
   - Merges per-route and route-level `context_extensions`.
   - Calls `client_->check(...)` (line 284). State = `Calling`,
     `filter_return_ = StopDecoding`.
5. **`onComplete(response)` (line 502)** — the central decision branch.

## Decision handling

### OK (line 539)

- Apply header mutations from the auth response:
  `headers_to_set` / `headers_to_add` / `headers_to_append` (lines 550–639).
- Delete headers in `headers_to_remove` (line 642).
- If `validate_mutations` and the mutation is invalid → 500 +
  `UnauthorizedExternalService` response flag (`rejectResponse()` line 422).
- Optional query-parameter rewrites.
- If `clear_route_cache` and headers were modified → invalidate the cached
  route (line 545).
- Response-side injections (`allowed_upstream_headers`, etc.) are stashed for
  the encode phase (lines 684–730).
- Shadow mode: decision stored in `FilterState`, request proceeds anyway
  (line 777).
- Increment `stats_.ok_`, `continueDecoding()` (line 780).

### Denied (line 784)

- Build a local reply using `response->status_code`, `body`, and
  `headers_to_set` (line 821).
- Shadow mode: record and continue instead (line 798).
- Increment `stats_.denied_`.

### Error (line 846)

- `failure_mode_allow=true`
  - Increment `stats_.failure_mode_allowed_`.
  - Optionally add `x-envoy-auth-failure-mode-allowed` (line 878).
  - `continueDecoding()` (line 880).
- `failure_mode_allow=false`
  - Send local reply using the response status or `status_on_error`
    (line 884).
- Shadow mode: record and continue (line 854).
- Truncate body to `max_denied_response_body_bytes` (line 814).
- Increment `stats_.error_`.

Response header limits (`enforce_response_header_limits`) are checked at
lines 384/396/410; overflow → `responseHeaderLimitsReached()` (line 897)
sends a 500.

## Header-mutation safety

- Decoder-phase mutations pass through `decoder_header_mutation_rules`
  (`FilterConfig`, lines 195–200).
- Invalid mutations are either silently dropped or converted to 500, depending
  on `validate_mutations`.

## Metadata / logging

- Dynamic metadata from the auth response is merged into the `StreamInfo`
  (line 534).
- `ExtAuthzLoggingInfo` stores latency, gRPC status, bytes sent/received
  (lines 449–483) for access logs.

## Per-route config

`FilterConfigPerRoute` (`ext_authz.h:328`):

- `disabled` — skip the check entirely (`decodeHeaders` line 290).
- `check_settings` — override `with_request_body` /
  `disable_request_body_buffering`.
- `grpc_service` / `http_service` — per-route client override.
- `context_extensions` — merged into the `CheckRequest`.

Runtime gates: `filter_enabled` (fractional %) and `filter_enabled_metadata`
(metadata matcher).

## Stats (`ext_authz.h:42`)

`ok`, `denied`, `error`, `disabled`, `failure_mode_allowed`, `invalid`,
`request_header_limits_reached`, `response_header_limits_reached`,
`shadow_denied`, `shadow_error`, `ignored_dynamic_metadata`,
`filter_state_name_collision`, `omitted_response_headers`.

## Files

- `config.{h,cc}` — proto → client setup (HTTP at `config.cc:41`, gRPC at
  `config.cc:65`).
- `ext_authz.{h,cc}` — `Filter` (`ext_authz.h:384`), `FilterConfig`
  (`ext_authz.h:169`), `ShadowDecisionObject` (`ext_authz.h:69`).
