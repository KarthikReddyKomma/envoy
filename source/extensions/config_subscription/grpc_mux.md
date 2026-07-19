# gRPC xDS — Deep Dive

The `grpc/` subfolder is the heart of xDS: a resilient bidirectional gRPC stream that multiplexes
one or more resource types and fans responses out to the right consumers. This document covers the
class layering, the mux family, `GrpcStream`, `WatchMap`, the factories, and the response-dispatch
path.

## 1. Class layering

```
GrpcSubscriptionImpl   (Config::Subscription adapter, per resource type)
   | holds GrpcMuxSharedPtr + a GrpcMuxWatch
   v
GrpcMux (interface) — implemented by:
   - GrpcMuxImpl           legacy SotW ADS mux        grpc/grpc_mux_impl.*
   - NewGrpcMuxImpl        legacy Delta mux           grpc/new_grpc_mux_impl.*
   - XdsMux::GrpcMuxSotw   unified SotW mux           grpc/xds_mux/grpc_mux_impl.*
   - XdsMux::GrpcMuxDelta  unified Delta mux          grpc/xds_mux/grpc_mux_impl.*
   | each owns:
   v
GrpcStream<Req,Resp>   bidi stream wrapper (retries/backoff/rate limit)   grpc/grpc_stream.h
WatchMap               resource-name -> interested watches fan-out        grpc/watch_map.*
```

The **legacy** (`GrpcMuxImpl` / `NewGrpcMuxImpl`) vs **unified** (`XdsMux::*`) split is gated by
the runtime flag `envoy.reloadable_features.unified_mux`. The unified mux merges SotW and Delta
into one template parameterized by a `SubscriptionState`. The long-term direction is convergence
onto the unified implementation.

## 2. `GrpcSubscriptionImpl` — the per-type adapter

This adapts the typed `Subscription` to the untyped `GrpcMux`, and tracks per-API stats. It is
both a `Subscription` and (internally) the `SubscriptionCallbacks` the mux talks to:

```cpp
void GrpcSubscriptionImpl::start(const absl::flat_hash_set<std::string>& resources) {
  if (init_fetch_timeout_.count() > 0) { /* enable init-fetch timeout timer */ }
  watch_ = grpc_mux_->addWatch(type_url_, resources, *this, resource_decoder_, options_);
  stats_.update_attempt_.inc();
  // ADS batches initial requests for all types, so aggregated users must NOT call start();
  // non-ADS users must.
  if (!is_aggregated_) {
    grpc_mux_->start();
  }
}
```

- `start()` registers a watch via `grpc_mux_->addWatch(...)`. For **non-ADS** it then calls
  `grpc_mux_->start()`; for **ADS** (`is_aggregated_`) it does not, so the shared mux can batch the
  initial request for every type onto the one stream.
- `updateResourceInterest()` → `watch_->update(names)`.
- Both `onConfigUpdate` overloads disable the init-fetch timer, forward to `callbacks_`, and record
  success/version/duration stats. `onConfigUpdateFailed` maps the reason to stats and forwards
  timeout/rejected failures to the consumer.
- `pause()` returns a `ScopedResume` from `grpc_mux_->pause(type_url_)`.

`GrpcCollectionSubscriptionImpl` is the xdstp:// collection variant — its `start()` ignores
resource names and watches a single encoded collection-locator URL.

## 3. The mux: multiplexing types over one stream

### Per-type state

A single stream carries many resource types; each keeps independent state (current request, list
of watches, pause counters, TTL manager, nonce/version, control-plane id). In the legacy SotW mux
this is `ApiState` keyed by type_url; in the delta and unified muxes it's a `SubscriptionState`
paired with a `WatchMap`.

### Watches

`addWatch()` creates a watch, marks the type subscribed (recording dependency ordering), and
queues a discovery request:

```cpp
GrpcMuxWatchPtr GrpcMuxImpl::addWatch(const std::string& type_url, ...) {
  auto watch = std::make_unique<GrpcMuxWatchImpl>(resources, callbacks, decoder, type_url, *this, ...);
  if (!apiStateFor(type_url).subscribed_) {
    apiStateFor(type_url).request_.set_type_url(type_url);
    apiStateFor(type_url).request_.mutable_node()->MergeFrom(local_info_.node());
    apiStateFor(type_url).subscribed_ = true;
    subscriptions_.emplace_back(type_url);
  }
  queueDiscoveryRequest(type_url);
  return watch;
}
```

### Pause / resume

`pause(type_url)` increments a pause counter and returns a `ScopedResume` that, on destruction,
decrements and re-queues any request elided during the pause. This lets a consumer batch a burst
of interest changes into a single request, and lets the mux hold same-type requests while
processing a response.

### Request building & rate-limited draining

`sendDiscoveryRequest()` rebuilds `resource_names` from the union of all watches of that type,
attaches `node` only when required (first request / not skippable), and sends. `drainRequests()`
drains the queue while `grpc_stream_->checkRateLimitAllowsDrain()` permits.

## 4. `GrpcStream` — the resilient bidi stream

`GrpcStream<RequestProto, ResponseProto>` (`grpc/grpc_stream.h`) is the connection layer shared by
SotW and Delta. Responsibilities:

| Concern | Mechanism |
|---------|-----------|
| Connect | `establishNewStream()` → `async_client_->start(method, *this)`; on success `onStreamEstablished()`, on failure `onEstablishmentFailure(true)` + retry timer |
| Retries/backoff | `JitteredExponentialBackOffStrategy`; `setRetryTimer()` arms with `nextBackOffMs()`; backoff reset on a received message |
| Remote close | `onRemoteClose()` logs, nulls the stream, sets `connected_state` gauge to 0, and re-arms retry unless intentionally closed |
| Log noise | `isNonRetriableFailure()` quiets transient `DeadlineExceeded`/`ResourceExhausted`/`Unavailable` |
| Rate limiting | optional `TokenBucketImpl` (default 100 tokens, 10/s) gates draining; `checkRateLimitAllowsDrain()` consumes a token or arms `drain_request_timer_` |
| Response | `onReceiveMessage()` forwards the decoded response to `callbacks_->onDiscoveryResponse(...)` |

When `envoy.restart_features.xds_failover_support` is enabled, the mux wraps the stream in a
`GrpcMuxFailover` (primary + failover sources, the secondary using a fixed backoff).

## 5. `WatchMap` — fan-out to interested watches

`WatchMap` is the "several consumers want the same resource; give each its own watch but issue one
real subscription" layer. It keeps `watch_interest_: name -> {watches}` plus a set of
`wildcard_watches_`. The core query returns wildcard watches ∪ exact-name matches (with xdstp
normalization and namespace/glob fallbacks):

```cpp
absl::flat_hash_set<Watch*> WatchMap::watchesInterestedIn(const std::string& resource_name) {
  absl::flat_hash_set<Watch*> ret;
  if (!use_namespace_matching_) ret = wildcard_watches_;
  // ... exact match on watch_interest_; else namespace/xdstp-glob fallback ...
  return ret;
}
```

`updateWatchInterest()` returns set-level `{added_, removed_}` deltas so the mux knows what to
add/remove from the *real* subscription when the first/last watch on a name appears/disappears.

> Note: the legacy SotW `GrpcMuxImpl` is the **only** mux that does *not* use `WatchMap` — it
> dispatches by directly iterating `ApiState.watches_`. The delta mux and both unified muxes route
> everything through `WatchMap`.

## 6. The response-dispatch path

### Legacy SotW (`GrpcMuxImpl`) — iterate `ApiState.watches_`

```mermaid
sequenceDiagram
    participant S as GrpcStream
    participant M as GrpcMuxImpl
    participant W as each GrpcMuxWatchImpl
    participant C as consumer callbacks

    S->>M: onDiscoveryResponse(DiscoveryResponse)
    M->>M: lookup ApiState by type_url; pause same-type
    M->>M: decode resources; build name -> DecodedResourceRef map
    loop each watch of this type
        alt wildcard (empty resources_)
            M->>C: onConfigUpdate(all_resources, version)
        else named
            M->>C: onConfigUpdate(found_resources, version)
        end
    end
    M->>S: queueDiscoveryRequest (ACK: version + nonce)
```

A thrown `EnvoyException` during processing fires every watch's
`onConfigUpdateFailed(UpdateRejected, ...)` and sends a **NACK** (writes `error_detail`, does not
advance the version).

### Delta + unified muxes — via `WatchMap`

The mux hands the raw response to a `SubscriptionState` (which owns the protocol bookkeeping —
nonces, ACKs, per-resource versions), then routes the resulting added/removed decoded resources
through the type's `WatchMap`, which builds per-watch bundles and even synthesizes empty updates to
wildcard watches when appropriate. In the unified mux this is generic:
`onDiscoveryResponse → genericHandleResponse(type_url, msg)` → `subscriptionStateFor(type_url)` +
`watchMapFor(type_url)`.

## 7. The factories

| Factory | Name | Builds |
|---------|------|--------|
| `GrpcConfigSubscriptionFactory` | `envoy.config_subscription.grpc` | non-ADS SotW: dedicated mux (`GrpcMuxImpl` or `XdsMux::GrpcMuxSotw`), `sotwGrpcMethod`, non-aggregated `GrpcSubscriptionImpl` |
| `DeltaGrpcConfigSubscriptionFactory` | `envoy.config_subscription.delta_grpc` | non-ADS Delta: `NewGrpcMuxImpl` or `XdsMux::GrpcMuxDelta`, `deltaGrpcMethod` |
| `AdsConfigSubscriptionFactory` | `envoy.config_subscription.ads` | reuses the shared `data.ads_grpc_mux_`; aggregated `GrpcSubscriptionImpl` (no mux created) |
| `GrpcMuxFactory` (`MuxFactory` registry) | `envoy.config_mux.grpc_mux_factory` | the cluster-manager-owned shared ADS mux (`GrpcMuxImpl`, `StreamAggregatedResources`, with an `EdsResourcesCacheImpl`) |
| collection factories | `...delta_grpc_collection`, `...aggregated_grpc_collection`, `...ads_collection` | xdstp:// collection locators |

The resource type URL isn't registered per factory — it flows through `SubscriptionData.type_url_`
(set by the consumer) and selects the gRPC method via `sotwGrpcMethod()` / `deltaGrpcMethod()` from
`source/common/config/type_to_endpoint.h`.

## 8. Supporting infrastructure

Worth knowing, in `grpc/`:

| File | Role |
|------|------|
| `pausable_ack_queue.*` | Ordered ACK queue with per-type pause (delta) |
| `update_ack.h` | `UpdateAck` = nonce + error_detail |
| `eds_resources_cache_impl.*` | EDS fallback cache (ADS only) |
| `grpc_mux_failover.h` | Primary/failover stream multiplexing |
| `xds_source_id.*` | `XdsConfigSourceId` key for the resources delegate/persistence |
| `grpc_stream_interface.h` | The `GrpcStreamInterface` / `GrpcStreamCallbacks` contracts between mux and stream |
