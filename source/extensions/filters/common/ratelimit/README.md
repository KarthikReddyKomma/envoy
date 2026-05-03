# Rate Limit (shared filter infrastructure)

Shared async client + stat-name helpers that talk to an external global rate-limit service (RLS — Lyft's gRPC `envoy.service.ratelimit.v3.RateLimitService.ShouldRateLimit`). It is the single implementation used by all HTTP and network filters that want to consult a central RLS; each filter constructs a `Client` per call site and delivers the result back via `RequestCallbacks::complete`.

## Files
- `ratelimit.h` — abstract `Client` interface, `RequestCallbacks`, `LimitStatus` enum, and descriptor/metadata aliases (`ratelimit.h:26`, `ratelimit.h:43`, `ratelimit.h:68`).
- `ratelimit_impl.h/cc` — `GrpcClientImpl` (the only concrete client) + the `rateLimitClient(...)` factory (`ratelimit_impl.h:43`, `ratelimit_impl.h:85`, `ratelimit_impl.cc:132`).
- `stat_names.h` — per-process `StatNames` struct (ok / error / failure_mode_allowed / over_limit) built into the symbol table (`stat_names.h:15`).
- `BUILD` — two `envoy_cc_library` targets: the pure-interface `ratelimit_client_interface` and the gRPC implementation `ratelimit_lib`, plus a header-only `ratelimit_stat_names_lib`.

## Public interface
- `enum class LimitStatus { OK, Error, OverLimit }` (`ratelimit.h:26`).
- `class RequestCallbacks { virtual void complete(status, descriptor_statuses, response_headers_to_add, request_headers_to_add, response_body, dynamic_metadata) = 0; }` (`ratelimit.h:43`, `ratelimit.h:58`).
- `class Client { cancel(); detach(); limit(cbs, domain, descriptors, span, stream_info, hits_addend); }` (`ratelimit.h:68`). `limit()` may invoke `complete()` synchronously on the same stack frame — callers must tolerate that (`ratelimit.h:90`).
- `ClientPtr rateLimitClient(FactoryContext&, GrpcServiceConfigWithHashKey, timeout)` — constructs a `GrpcClientImpl` backed by a shared raw async client from the cluster manager (`ratelimit_impl.cc:132`).
- `struct StatNames(SymbolTable&, stat_prefix)` — interns `ratelimit.<prefix>.{ok,error,failure_mode_allowed,over_limit}` once per process so filters don't re-intern per request (`stat_names.h:15`).

## Implementation logic
Lifecycle of a single `GrpcClientImpl::limit` call (`ratelimit_impl.cc:68`):
1. `ASSERT(callbacks_ == nullptr)` — the client is single-inflight; callers must own one client per outstanding request (`ratelimit_impl.cc:71`, TODO at `ratelimit_impl.h:40`).
2. Build `RateLimitRequest` via `createRequest`, copying `domain`, `hits_addend`, and every `Descriptor` entry into `envoy::extensions::common::ratelimit::v3::RateLimitDescriptor`. Per-descriptor `limit`, `hits_addend`, and `is_negative_hits` are carried through (`ratelimit_impl.cc:43`-`66`).
3. Dispatch via `async_client_->send(service_method_, request, *this, parent_span, options)` — options carry `timeout_` and a `ParentContext` with the caller's `StreamInfo` (`ratelimit_impl.cc:78`).
4. The returned inflight handle is stashed in `request_` if non-null.

Completion paths clear state before invoking the user callback (important because callbacks in the stream-done path can destroy the client):
- `onSuccess` (`ratelimit_impl.cc:85`): asserts response is not UNKNOWN, maps OVER_LIMIT to `LimitStatus::OverLimit` and tags span `ratelimit_status=over_limit` or `ok` (`ratelimit_impl.cc:90`). Copies `response_headers_to_add` and `request_headers_to_add` into fresh `Http::HeaderMapImpl`s (`ratelimit_impl.cc:97`-`109`). Builds `descriptor_statuses` and optional `dynamic_metadata`, then caches `callbacks_` locally, clears `callbacks_`/`request_`, and finally calls `complete(...)` — the local handle avoids use-after-free if the callback destroys the client (`ratelimit_impl.cc:115`).
- `onFailure` (`ratelimit_impl.cc:121`): logs at debug and calls `complete(LimitStatus::Error, nullptr, nullptr, nullptr, EMPTY_STRING, nullptr)` using the same local-handle pattern.

`cancel()` (`ratelimit_impl.cc:27`) asserts a callback is registered, cancels the gRPC request, and clears state; `detach()` (`ratelimit_impl.cc:36`) keeps the callback alive but detaches the inflight request — the calling filter is then responsible for ensuring the callback outlives the response (`ratelimit.h:78`).

The factory `rateLimitClient` obtains a `RawAsyncClientSharedPtr` via `grpcAsyncClientManager().getOrCreateRawAsyncClientWithHashKey(config_with_hash_key, scope, /*skip_cluster_check=*/true)` and throws on status error (`ratelimit_impl.cc:135`).

## Consumers
- `source/extensions/filters/http/ratelimit` — HTTP global rate-limit filter (`ratelimit.{h,cc}`, `config.{h,cc}`, `ratelimit_headers.h`).
- `source/extensions/filters/http/local_ratelimit` — uses `StatNames` and the common `Descriptor` types for local + shadow reporting (`local_ratelimit.h`).
- `source/extensions/filters/network/ratelimit` — TCP network filter (`config.{h,cc}`, `ratelimit.h`).
- `source/extensions/filters/network/thrift_proxy/filters/ratelimit` — Thrift sub-filter (`ratelimit.{h,cc}`, `config.{h,cc}`).

All four share the same `GrpcClientImpl` and descriptor wire format; only the descriptor-generation logic differs per protocol.

## Stats / errors / failure modes
Stats are recorded by the consuming filter, not here, using `StatNames::{ok_,error_,over_limit_,failure_mode_allowed_}` (`stat_names.h:29`). Failure handling:
- gRPC error (timeout, unavailable, stream reset) -> `LimitStatus::Error`; the consumer chooses whether to treat the request as `failure_mode_allowed`.
- Response `OverLimit` -> `LimitStatus::OverLimit`; consumer typically returns 429 + the RLS-supplied `response_body`/`response_headers_to_add`.
- `UNKNOWN` `overall_code` is rejected by `ASSERT` in `onSuccess` (`ratelimit_impl.cc:87`) so unconfigured RLS responses crash debug builds.
- Destructor `ASSERT(!callbacks_)` catches consumers that fail to call `cancel()` before tearing down the filter (`ratelimit_impl.cc:25`).
