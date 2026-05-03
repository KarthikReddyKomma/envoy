# Router Filter (`envoy.filters.http.router`)

The router is the **terminal decoder filter** — it always sits at the end of the
HTTP filter chain. It is the filter that actually forwards the decoded request
to an upstream cluster and streams the response back downstream.

Proto: `envoy.extensions.filters.http.router.v3.Router`.

Most of the interesting logic lives in `source/common/router/` (shared with
upstream filter chains). This directory is a thin factory that wires the shared
code into an HTTP filter.

## Why it must be last

Every filter ahead of the router gets to mutate or short-circuit the request
(auth, rate limit, header mutation, fault injection, etc.). The router is the
point where the final, fully-mutated request is committed to the network.
No filter after the router can run on the decode path, because by the time the
router is done the request has already been written upstream.

## Request lifecycle

1. **`decodeHeaders()`** — `common/router/router.cc:393`
   - Look up the matched route via `callbacks_->routeSharedPtr()`.
   - Direct-response / redirect routes: return a synthesized response and stop
     (`router.cc:435–464`).
   - Resolve the cluster via `cm_.getThreadLocalCluster()`.
   - Compute final timeouts with `FilterUtility::finalTimeout()`
     (`router.cc:145`), merging route config, `x-envoy-upstream-rq-timeout-ms`,
     `grpc-timeout`, and `x-envoy-expected-rq-timeout-ms`.
   - Compute hedging params (`FilterUtility::finalHedgingParams()`,
     `router.cc:278`).
2. **Host selection** — `LoadBalancerContextBase` hooks
   (`computeHashKey`, `metadataMatchCriteria`, `shouldSelectAnotherHost`).
   Host selection can be asynchronous (`router.cc:1969`).
3. **Upstream request** — an `UpstreamRequest`
   (`common/router/upstream_request.h:67`) is created; the connection-pool
   callback streams headers/data/trailers upstream. Optional upstream HTTP
   filter chain runs inside the `UpstreamRequest`.
4. **Response path** — the router implements the upstream callbacks:
   - `onUpstream1xxHeaders()` (`router.h:434`)
   - `onUpstreamHeaders()` (`router.cc:1475`) — also the retry decision point.
   - `onUpstreamData()` (`router.cc:1656`)
   - `onUpstreamTrailers()` (`router.cc:1679`)
   - `onUpstreamReset()` — on reset, calls `maybeRetryReset()`
     (`router.cc:1266`).
5. **Completion** — `onUpstreamComplete()` (`router.cc:1710`) flushes access
   logs and cancels any still-outstanding hedged attempts.

## Key responsibilities

| Concern             | Where                                                                                             |
|---------------------|---------------------------------------------------------------------------------------------------|
| Global timeout      | `response_timeout_` timer; `onGlobalTimeout` → 504.                                                |
| Per-try timeout     | `UpstreamRequest::setupPerTryTimeout()`; may trigger hedging or retry.                             |
| Retries             | `RetryState` decides; `doRetry()` (`router.cc:1937`) / `continueDoRetry()` (`router.cc:1988`) re-submit the buffered request on a new host. |
| Hedging             | `finalHedgingParams` + per-try-timeout; first response wins, losers are reset.                      |
| Shadow traffic      | `FilterUtility::shouldShadow` + async `shadow_writer_` — fire-and-forget mirror.                    |
| gRPC                | `grpc_request_` flag; gRPC timeout from `grpc-timeout`; grpc-status → HTTP code (`router.cc:1486`). |
| Upgrades / CONNECT  | Route config + `setupRouteTimeoutForWebsocketUpgrade()`.                                           |
| Internal redirect   | `convertRequestHeadersForInternalRedirect()`; restart the request with the new target.             |
| Stats               | `chargeUpstreamCode()` (`router.cc:305`) emits per-code / per-zone / canary counters.               |

## Response headers set by the router

Unless `suppress_envoy_headers=true`:

- `x-envoy-attempt-count` — retry attempt number
- `x-envoy-upstream-service-time` — upstream wall time
- any route-level `request_headers_to_add` / `response_headers_to_add`

## Configuration knobs

| Field                                                        | Effect |
|--------------------------------------------------------------|--------|
| `dynamic_stats`                                              | Emit per-cluster/code stats. Default `true`; turn off for very high QPS. |
| `start_child_span` *(deprecated)*                            | Prefer `HCM.tracing.spawn_upstream_span`. |
| `upstream_log` / `upstream_log_options`                      | One access-log entry per upstream try (retries produce multiple). |
| `strict_check_headers`                                       | Validates `x-envoy-*-timeout-ms`, `x-envoy-max-retries`, `x-envoy-retry-on`, `x-envoy-retry-grpc-on`. |
| `respect_expected_rq_timeout`                                | Honour `x-envoy-expected-rq-timeout-ms` from an egress Envoy. |
| `suppress_envoy_headers`                                     | Do not emit `x-envoy-*` request/response headers. |
| `suppress_grpc_request_failure_code_stats`                   | Skip 5xx HTTP stat when the request is gRPC. |
| `upstream_http_filters`                                      | Terminal codec filter + any upstream-only filters. |
| `reject_connect_request_early_data`                          | Reject CONNECT payloads before the 200 response. |

## Files

- `config.cc` / `config.h` — registration + factory (`config.cc:14`).
- `common/router/router.{h,cc}` — main `Filter` class (`router.h:261`).
- `common/router/upstream_request.{h,cc}` — per-try state machine.
- `common/router/config_impl.{h,cc}` — route / retry / hedge policy parsing.
