# ExtAuthz Client Library (shared filter infrastructure)

C++ library used by the HTTP ext_authz filter (`source/extensions/filters/http/ext_authz`) and the network ext_authz filter (`source/extensions/filters/network/ext_authz`) to talk to an external authorization service. It defines the abstract `Client` interface, the `CheckRequest` builder that snapshots request attributes, and two concrete implementations: a gRPC client that speaks `envoy.service.auth.v3.Authorization.Check` and an HTTP client that turns an Envoy async HTTP request into an authz check.

## Files
- `ext_authz.h` - Public interface: `Client`, `RequestCallbacks`, `CheckStatus`, `Response`, tracing/response-code-detail constants, and the `x-envoy-auth-*` header names.
- `ext_authz_grpc_impl.h/cc` - `GrpcClientImpl` unary gRPC client.
- `ext_authz_http_impl.h/cc` - `ClientConfig` + `RawHttpClientImpl` HTTP client.
- `check_request_utils.h/cc` - `CheckRequestUtils` for building `envoy::service::auth::v3::CheckRequest` from an HTTP stream or TCP read-filter callbacks, plus header matcher helpers (`HeaderKeyMatcher`, `NotHeaderKeyMatcher`).

## Public interface
- `class Client` (`ext_authz.h:153`) - `void check(RequestCallbacks&, const CheckRequest&, Tracing::Span&, const StreamInfo::StreamInfo&)`, `void cancel()`, `StreamInfo* streamInfo() const`. The callback `onComplete(ResponsePtr&&)` (`ext_authz.h:150`) may be invoked synchronously from within `check()`.
- `struct Response` (`ext_authz.h:87`) - carries `CheckStatus` (OK/Error/Denied), eight distinct header-mutation vectors for upstream/downstream (`headers_to_append`, `headers_to_set`, `headers_to_add`, `response_headers_to_add`, `response_headers_to_set`, `response_headers_to_add_if_absent`, `response_headers_to_overwrite_if_exists`, `headers_to_remove`), query param mutations, `dynamic_metadata`, optional `body`/`status_code`, and the upstream `grpc_status`.
- `class GrpcClientImpl(async_client, timeout)` (`ext_authz_grpc_impl.h:46`) - wraps `Grpc::AsyncClient<CheckRequest,CheckResponse>`; one instance per filter stack (unary RPC).
- `class RawHttpClientImpl(cluster_manager, ClientConfigSharedPtr)` (`ext_authz_http_impl.h:149`) - issues an async HTTP request to the configured cluster.
- `class ClientConfig` (`ext_authz_http_impl.h:26`) - parses `HttpService`, holds cluster name, path prefix/override, tracing name, request-header parser, client/upstream/dynamic-metadata header matchers, retry policy.
- `CheckRequestUtils::createHttpCheck(...)` / `createTcpCheck(...)` (`check_request_utils.h:93`, `:116`) - fills the `CheckRequest` with peer, request, TLS-session, and metadata attributes, optionally packing the body as bytes and encoding raw headers.

## Implementation logic
- gRPC `check()` (`ext_authz_grpc_impl.cc:95`) calls `async_client_.send(service_method_, request, *this, parent_span, options)`; responses are decoded in `onSuccess` which branches on `status.code()`, `has_error_response()`, or defaults to Denied with HTTP 403 (`ext_authz_grpc_impl.cc:108`). Header mutations are split between "append" and "set" based on the proto's `append` flag or `append_action` enum (`ext_authz_grpc_impl.cc:31`); unknown actions flip `saw_invalid_append_actions` (`:63`). `onFailure` maps any non-OK gRPC status to `CheckStatus::Error` and fires the callback with an empty response (`:164`).
- HTTP client caches a static `lengthZeroHeader()` and an `errorResponse()` template (`ext_authz_http_impl.cc:30`, `:39`). A `SuccessResponse` helper iterates response headers once and distributes them to upstream/append/response/dynamic-metadata buckets based on the `ClientConfig` matchers (`ext_authz_http_impl.cc:65`). Path prefix and path override are mutually exclusive; both validators enforce a leading `/` (`:113`, `:120`).
- Retry policy is built via `createRetryPolicy` which converts the core retry policy to a route retry policy; the default `retry_on` depends on `envoy.reloadable_features.ext_authz_http_client_retries_respect_user_retry_on` (`ext_authz_http_impl.cc:136`).
- `GrpcClientImpl::cancel()` asserts there is an inflight callback, then calls `request_->cancel()` (`ext_authz_grpc_impl.cc:89`). The destructor asserts `!callbacks_` to catch leaks (`:87`).
- Tracing tags (`ext_authz_status` -> `ok`/`unauthorized`/`error`) are set on the parent span from the gRPC client (`ext_authz_grpc_impl.cc:114`, `:122`, `:136`). HTTP client sets the tracing span name via `ClientConfig::tracing_name_`.

## Consumers
- `source/extensions/filters/http/ext_authz` - links both `ext_authz_grpc_lib` and `ext_authz_http_lib`, picks one based on `config.services_case()`.
- `source/extensions/filters/network/ext_authz` - links only `ext_authz_grpc_lib` (TCP-level authz is gRPC-only).

## Stats / errors / failure modes
- This library emits no stats; counters like `ext_authz.ok/denied/error/failure_mode_allowed` belong to the filter wrappers.
- `CheckStatus::Error` is the only path where `onFailure` fires; the filter decides whether to fail-open (`failure_mode_allow`) based on its own config. Error responses intentionally leave `status_code` unset so callers can apply their `status_on_error` configuration (`ext_authz_grpc_impl.cc:126`, `ext_authz_http_impl.cc:37`).
- Response-code-details values surfaced to the filter: `ext_authz_denied`, `ext_authz_error`, `ext_authz_invalid` (`ext_authz.h:42`).
- `saw_invalid_append_actions` lets callers emit a debug/warn stat when the authz server returns an enum the client cannot honour (`ext_authz.h:115`).
