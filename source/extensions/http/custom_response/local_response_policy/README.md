# Local Response Policy

A policy plugin consumed by the `custom_response` HTTP filter
(`source/extensions/filters/http/custom_response`). When the filter's matcher
picks this policy for an upstream response, the policy synthesizes a local
reply (optionally formatted from a data source / format string) instead of
letting the original response pass through. This is the "serve a canned error
page" branch of custom response handling.

Proto: `envoy.extensions.http.custom_response.local_response_policy.v3.LocalResponsePolicy`.

## Files
- `local_response_policy.h/cc` - policy implementation.
- `local_response_factory.h/cc` - factory registering with the custom
  response policy registry.

## Interface
- Implements `Extensions::HttpFilters::CustomResponse::Policy` (its
  `encodeHeaders` is invoked by the custom response filter for matched
  responses).
- Factory is a `CustomResponsePolicyFactory` registered via
  `REGISTER_CUSTOM_RESPONSE_POLICY_FACTORY`.

## Logic
- Constructor materializes three optional pieces from proto:
  `local_body_` (read eagerly from the configured `DataSource`),
  `status_code_`, `formatter_` (a `SubstitutionFormatter` when
  `body_format` is set), and a `HeaderParser` for
  `response_headers_to_add`.
- `encodeHeaders` is the entry point. It first asserts no custom response
  filter state exists yet (this policy never chains), picks the final
  status code via `getStatusCodeForLocalReply`, formats the body, then calls
  `encoder_callbacks->sendLocalReply` with the body and a mutator closure
  that applies `response_headers_to_add`. It always returns `StopIteration`
  because the local reply path replaces the upstream response.
- `formatBody` starts from the configured body (if any) and runs it through
  the formatter when configured; the formatter has access to the original
  request/response headers, stream info, and the active span.
- `getStatusCodeForLocalReply` prefers the configured `status_code` and
  falls back to the upstream status, finally defaulting to 500.

## Key decision points
- `local_response_policy.cc:21` - body `DataSource` is read at construction
  (single file/inline fetch), which avoids I/O on hot path.
- `local_response_policy.cc:48` - ENVOY_BUG asserts this policy is not
  stacked under another custom response filter state.
- `local_response_policy.cc:56` - returns `StopIteration` because
  `sendLocalReply` has supplied the response.
- `local_response_policy.cc:59` - status code fallback order.

## Configuration
- `status_code` - override for the outgoing status.
- `body` - `DataSource` read at load time.
- `body_format` - `SubstitutionFormatString` used to template the body.
- `response_headers_to_add` - standard header mutations applied before the
  local reply is sent.

## Stats / errors
No dedicated stats; the surrounding custom response filter owns counters.
Invalid `body_format` or `response_headers_to_add` throw at config load via
`THROW_OR_RETURN_VALUE`.
