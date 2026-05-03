# Redirect Policy

A policy plugin consumed by the `custom_response` HTTP filter
(`source/extensions/filters/http/custom_response`). When the filter's matcher
selects this policy for a response, the request is rewritten and the stream
recreated so that it is re-routed to a redirect target (either a literal URI
or a template built from a `RedirectAction`). The policy is also invoked a
second time on the response from the new target so it can apply final status
code / header mutations.

Proto: `envoy.extensions.http.custom_response.redirect_policy.v3.RedirectPolicy`.

## Files
- `redirect_policy.h/cc` - policy plus `ModifyRequestHeadersAction`
  extension point.
- `redirect_factory.h/cc` - factory registering with the custom response
  policy registry.

## Interface
- Implements `Extensions::HttpFilters::CustomResponse::Policy` (the custom
  response filter calls `encodeHeaders`).
- Exposes a nested `ModifyRequestHeadersActionFactory` (`category()` =
  `envoy.http.custom_response.redirect_policy.modify_request_headers_action`)
  so deployments can plug arbitrary request mutation logic that runs after
  the redirect URI is computed.
- Factory is a `CustomResponsePolicyFactory` registered with
  `REGISTER_CUSTOM_RESPONSE_POLICY_FACTORY`.

## Logic
- Configuration must supply exactly one of `uri` or `redirect_action`
  (asserted in the ctor). `redirect_action` is translated to
  `Http::Utility::RedirectConfig` via `createRedirectConfig`; `regex_rewrite`
  and `prefix_rewrite` are explicitly rejected.
- `encodeHeaders` has two modes:
  1. **First pass (upstream response triggered redirect).** Filter state
     `envoy.filters.http.custom_response` is absent. The policy builds the
     absolute URL (`uri` or `Http::Utility::newUri`), rewrites the downstream
     request's scheme/host/path/method, strips any `#fragment`, clears
     route cache, applies `request_headers_to_add`, invokes any
     `ModifyRequestHeadersAction`, caches the original response code in
     filter state, and calls `decoder_callbacks->recreateStream`. Returns
     `StopIteration`.
  2. **Second pass (response from redirect target).** Filter state is
     populated: the policy applies `response_headers_to_add`, overrides
     the status code (using the configured value or the cached original),
     and returns `Continue` so the response flows downstream.
- A `Cleanup` restores the original host/path/scheme if the redirect is
  aborted (e.g. no matching route).

## Key decision points
- `redirect_policy.cc:78` - ctor-time assertion that exactly one of `uri`
  or `redirect_action` is set.
- `redirect_policy.cc:41` - reject `regex_rewrite` / `prefix_rewrite`
  (not supported for custom response).
- `redirect_policy.cc:87` - reject `#fragment` in `path_redirect`.
- `redirect_policy.cc:143` - early-out if a local reply was already sent
  before decodeHeaders.
- `redirect_policy.cc:158` - cleanup restores original headers on abort.
- `redirect_policy.cc:202` - no-route path increments
  `custom_response_redirect_no_route_` and returns `Continue` so access
  logs reflect the mutated headers.
- `redirect_policy.cc:219` - original response code is cached in filter
  state so pass 2 can surface it.

## Configuration
- `uri` *or* `redirect_action` (exactly one).
- `status_code` - optional override; otherwise the cached upstream code is
  used.
- `request_headers_to_add`, `response_headers_to_add` - standard mutations.
- `modify_request_headers_action` - typed extension invoked after the URI
  is applied to the downstream headers.

## Stats / errors
Counters (`redirect_policy.h:33`):
- `custom_response_redirect_no_route` - resolved URI had no matching route.
- `custom_response_invalid_uri` - URL parse failed at runtime (only
  possible via `redirect_action`; literal `uri` is validated at config
  load and throws instead).
