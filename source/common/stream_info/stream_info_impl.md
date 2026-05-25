# `stream_info_impl.h` — `StreamInfoImpl` & `UpstreamInfoImpl`

This is the single header that defines:

- **`StreamInfoImpl`** — implements `Envoy::StreamInfo::StreamInfo` and is the per-stream metadata bag attached
  to every L4 connection and every L7 stream.
- **`UpstreamInfoImpl`** — implements `Envoy::StreamInfo::UpstreamInfo` and is the per-upstream-attempt
  sub-bag that hangs off `StreamInfoImpl::upstream_info_`.

It also defines two helper structs visible only through these classes:

- `BytesMeterImpl` (alias for `StreamInfo::BytesMeter`) — counters for header / wire bytes in each direction.
- The constructors / `setFromForRecreateStream(...)` helper used by HCM's recreate-stream flow.

Roughly **600 lines** of mostly inline field accessors. Mutation methods do **not** lock — `StreamInfoImpl` is
single-threaded by contract.

---

## Public surface (the bits people actually call)

### Construction

```cpp
// Connection-level (TCP proxy, UDP, network filter manager)
StreamInfoImpl(TimeSource& ts,
               Network::ConnectionInfoProviderSharedPtr down_info,
               FilterState::LifeSpan life_span = FilterState::LifeSpan::Connection);

// HTTP stream (HCM)
StreamInfoImpl(Http::Protocol protocol,
               TimeSource& ts,
               Network::ConnectionInfoProviderSharedPtr down_info,
               FilterState::LifeSpan life_span = FilterState::LifeSpan::FilterChain,
               FilterStateSharedPtr parent_filter_state = nullptr);
```

The HTTP overload's `parent_filter_state` is what wires the request-scoped `FilterStateImpl` to the
connection-scoped one (so writes at `Lifespan::DownstreamConnection` bubble up).

### Sectioned field accessors

The header is grouped into eight logical sections. Each is one row in the
[hierarchy diagram](CLASS_HIERARCHY.md#stream-info--implementations) — calling out the fields here so you can
ctrl-f the header faster.

| Section                           | Fields & methods                                                                 |
|-----------------------------------|----------------------------------------------------------------------------------|
| **Timing**                        | `startTime()`, `startTimeMonotonic()`, `currentDuration()`, `lastDownstreamTxByteSent()`, `requestComplete()`, `downstreamTiming()`, `setUpstreamInfo` chain → `upstreamTiming()` |
| **Identity**                      | `getStreamIdProvider()`, `setStreamIdProvider()`, `getRequestIDProvider()`, `setRequestIDProvider()`, `connectionID()`, `setConnectionID(uint64_t)`, `attemptCount()`, `setAttemptCount(uint32_t)` |
| **L4 downstream**                 | `downstreamAddressProvider()` (returns `ConnectionInfoProvider&`), `setDownstreamSslConnection(InfoConstSharedPtr)`, `downstreamSslConnection()`, `downstreamTransportFailureReason()`, `connectionTerminationDetails()` |
| **L7 request / response**         | `protocol()`, `setProtocol(Protocol)`, `getRequestHeaders()`, `setRequestHeaders(RequestHeaderMap&)`, `responseCode()`, `setResponseCode(uint32_t)`, `responseCodeDetails()`, `setResponseCodeDetails(string)`, `responseFlags()`, `hasResponseFlag(Flag)`, `setResponseFlag(Flag)`, `customResponseFlags()` |
| **Routing**                       | `route()`, `setRoute(RouteConstSharedPtr)`, `routeName()`, `virtualClusterName()`, `setVirtualClusterName(string)`, `upstreamClusterInfo()`, `setUpstreamClusterInfo(ClusterInfoConstSharedPtr)` |
| **Upstream attempt**              | `upstreamInfo()`, `setUpstreamInfo(UpstreamInfoSharedPtr)`, `upstreamInfo() const`                                                                              |
| **Metadata / filter state**       | `dynamicMetadata()`, `setDynamicMetadata(ns, value)`, `setDynamicTypedMetadata(ns, value)`, `filterState()`, `upstreamInfo()->upstreamFilterState()` |
| **Bytes & tracing**               | `getUpstreamBytesMeter()`, `setUpstreamBytesMeter()`, `getDownstreamBytesMeter()`, `bytesReceived()`, `bytesSent()`, `addBytesSent(n)`, `addBytesReceived(n)`, `traceReason()`, `setTraceReason(Reason)`, `isShadow()`, `setIsShadow(bool)`, `parentStreamInfo()`, `setParentStreamInfo(StreamInfo&)` |

---

## Lifecycle methods

### `onRequestComplete()`

Records the request-end monotonic timestamp into `final_time_`. After this is called:

- `requestComplete()` returns the precise duration `final_time_ - start_time_monotonic_`.
- `currentDuration()` continues to return *current* duration if called again (used by long-tail observers).

HCM calls this exactly once per HTTP stream, at the moment the upstream/downstream response ends (whichever
defines the *end* per protocol). TCP/UDP proxies call it when their connection closes.

### `setFromForRecreateStream(StreamInfoImpl& other)`

When `Http::ConnectionManagerImpl` performs an internal redirect or `recreateStream`, it allocates a *fresh*
`StreamInfoImpl` but wants to preserve "the user-visible request continues". Carries forward:

- `start_time_` (so duration is end-to-end, not per redirected hop).
- `downstream_bytes_meter_`.
- `attempt_count_`.
- `request_id_extension_` and the matching `stream_id_provider_` (so log correlation IDs survive).
- The request-scoped `FilterStateImpl` from `other.filterState()->parent()` at `Lifespan::Request`.
- Any custom response flags already set.

Notably it does **not** carry forward `responseFlags()`, `responseCode()`, the upstream info, or the route — the
redirect chooses those fresh.

### `dumpState(ostream&, indent)`

Implements `Envoy::FmtFormatter` / `DUMP_STATE` for crash dumps. Walks every populated field and prints it in a
debugger-friendly indent-based format. Used by the segfault handler to attach "what was this stream doing" to a
crash report — extremely valuable when triaging a core file.

---

## `UpstreamInfoImpl` in detail

```cpp
class UpstreamInfoImpl final : public UpstreamInfo {
public:
  // Setters used by routers / cluster managers / pools:
  void setUpstreamHost(Upstream::HostDescriptionConstSharedPtr host) override;
  void setUpstreamLocalAddress(Network::Address::InstanceConstSharedPtr) override;
  void setUpstreamRemoteAddress(Network::Address::InstanceConstSharedPtr) override;
  void setUpstreamConnectionId(uint64_t) override;
  void setUpstreamInterfaceName(absl::string_view) override;
  void setUpstreamSslConnection(Ssl::ConnectionInfoConstSharedPtr) override;
  void setUpstreamTransportFailureReason(absl::string_view) override;
  void setUpstreamNumStreams(uint64_t) override;
  void setUpstreamProtocol(Http::Protocol) override;
  void setUpstreamFilterState(FilterStateSharedPtr) override;

  // Reads (one accessor per field; UpstreamTiming returned by ref so callers can mutate it).
  UpstreamTiming& upstreamTiming() override;

  void dumpState(std::ostream& os, int indent) const;

private:
  Upstream::HostDescriptionConstSharedPtr upstream_host_;
  Network::Address::InstanceConstSharedPtr upstream_local_address_;
  Network::Address::InstanceConstSharedPtr upstream_remote_address_;
  absl::optional<uint64_t> upstream_connection_id_;
  std::string upstream_interface_name_;
  Ssl::ConnectionInfoConstSharedPtr upstream_ssl_info_;
  UpstreamTiming upstream_timing_;
  std::string upstream_transport_failure_reason_;
  uint64_t upstream_num_streams_{};
  absl::optional<Http::Protocol> upstream_protocol_;
  FilterStateSharedPtr upstream_filter_state_;
};
```

### Why `upstream_filter_state_` is separate

The downstream `FilterStateImpl` (on `StreamInfoImpl`) has lifespan `FilterChain` (or `Connection`). The
**upstream** filter chain has its own filter set that runs per attempt and may have its own scratch — having a
distinct `FilterStateImpl` here keeps the upstream filters from accidentally writing to the downstream store
(and vice versa). The upstream `FilterState` is created by the cluster manager / pool when it constructs the
upstream filter chain for the attempt.

### Retries

```mermaid
sequenceDiagram
    autonumber
    participant Router
    participant SI as StreamInfoImpl
    participant U1 as UpstreamInfoImpl (attempt #1)
    participant U2 as UpstreamInfoImpl (attempt #2)

    Router->>SI: setUpstreamInfo(U1)
    Router->>U1: setUpstreamHost(host_a); setUpstreamConnectionId(...)
    Note over U1: timing recorded, transport reason ""
    U1-->>Router: upstream connect failed
    Router->>SI: setUpstreamInfo(U2)   %% U1 is dropped (no other refs)
    Router->>U2: setUpstreamHost(host_b); ...
    Note over SI: only the LATEST attempt is on SI; previous attempts' state is lost
    Note over SI: filters that need history should plant in filter_state_
```

If a feature needs per-attempt history (e.g., adaptive load balancing), it should snapshot the data into
`StreamInfoImpl::filterState()` under a key the filter owns, then walk attempts.

---

## Bytes meters

`BytesMeter` is a plain counter struct (no virtual). `StreamInfoImpl` holds **two** shared_ptr meters:

- `downstream_bytes_meter_` — counts traffic between Envoy and the downstream client.
- `upstream_bytes_meter_` — counts traffic between Envoy and the upstream server (the *current* attempt).

Each meter tracks:

| Field                  | Meaning                                                  |
|------------------------|----------------------------------------------------------|
| `header_bytes_sent`    | bytes of HTTP headers / TLS records framed as "header"   |
| `header_bytes_received`| ditto, inbound                                           |
| `wire_bytes_sent`      | total bytes that hit the socket (incl. retransmit)       |
| `wire_bytes_received`  | total bytes read from the socket                         |
| `body_bytes_sent`      | application-level body bytes                             |
| `body_bytes_received`  | ditto, inbound                                           |

Why a shared_ptr? Because on **internal redirect / recreate stream**, the *downstream* meter is forwarded to
the next `StreamInfoImpl` (via `setFromForRecreateStream`) so `%BYTES_RECEIVED%` reports the cumulative bytes
the client actually sent. The upstream meter is *not* forwarded — each attempt has its own.

---

## Custom response flags

`response_flags_` is `absl::flat_hash_set<ResponseFlag>` where `ResponseFlag` is a thin wrapper around `uint16_t`
(numeric ID). Built-in flags use IDs ≤ `LastFlag` (defined in
`envoy/stream_info/stream_info.h`). Extensions allocate IDs > `LastFlag` via
`ResponseFlagUtils::CustomFlag(name, /*id*/Envoy::StreamInfo::ResponseFlagUtils::nextId())`.

`customResponseFlags()` returns only the IDs that fall in the custom range, used by `%RESPONSE_FLAGS%` to
disambiguate when formatting.

---

## Field-by-field debug tip

If `dumpState` looks empty in a core file, the most common culprit is **`StreamInfoImpl` was destroyed before the
crash** (use stack-unwind of the access logger or the codec stream to confirm). The
`Network::ConnectionInfoProvider` it points at is also reference-counted — if its provider is gone,
`downstream_address_provider_->dumpState()` will be the only line missing.
