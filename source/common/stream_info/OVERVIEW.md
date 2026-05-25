# StreamInfo — Overview

`stream_info/` is small in line count but **load-bearing** for Envoy: every other module reads it for telemetry,
formatting, tracing, and access logging. This document gives the mental model — for class-level details see
[`stream_info_impl.md`](stream_info_impl.md) / [`filter_state.md`](filter_state.md) / [`utility.md`](utility.md).

---

## The four roles

### 1. `StreamInfoImpl` — the per-stream metadata bag

A request- or connection-scoped struct of:

- Timing (`start_time_`, `final_time_`, `DownstreamTiming`, `UpstreamTiming`).
- Identity (`StreamIdProviderImpl`, `connectionID()`, `attemptCount()`).
- L4 facts (downstream remote/local/direct-remote addresses, downstream `Ssl::ConnectionInfo`, downstream
  connection termination details).
- L7 facts (HTTP protocol, request headers pointer, response code + details, response code-details / flags).
- Routing facts (`Router::Route`, `VirtualHost`, `UpstreamClusterInfo`).
- A single `UpstreamInfoImpl` slot (the *current* attempt — may be swapped on retries).
- Metadata (`envoy::config::core::v3::Metadata` for `dynamicMetadata` + opaque-typed metadata).
- `FilterStateImpl` at the **FilterChain** lifespan, chained to upstream `FilterState` (Connection lifespan) via
  `parent_`.
- Bytes meters (`BytesMeter` for downstream and upstream — counts headers vs body, sent vs received,
  retransmitted vs total).
- Custom `ResponseFlag` set and `customFlags()` machinery for extensions to register their own.

`StreamInfoImpl` is **not thread-safe**. It's owned by the codec / network filter manager and is only touched from
the dispatcher thread that owns the connection.

### 2. `UpstreamInfoImpl` — the per-attempt sub-bag

Whenever Envoy makes an upstream attempt (connect pool pick, TCP proxy upstream, retry), a fresh
`UpstreamInfoImpl` is constructed and `StreamInfoImpl::setUpstreamInfo(...)` swapped in. It holds:

- The picked `Upstream::HostDescription`.
- Upstream local + remote addresses (resolved at connect time).
- Upstream `Ssl::ConnectionInfo` (after TLS handshake).
- An `UpstreamTiming` (connect start/end, first up/downstream byte ts, handshake complete).
- Upstream connection ID + transport failure reason + cluster name override.
- A second `FilterStateImpl` instance dedicated to **upstream filter state** (lifespan = `FilterChain` but tied
  to the per-attempt upstream filter chain).

On a retry, the *old* `UpstreamInfoImpl` is dropped — only the **last** attempt's data is preserved on the parent
`StreamInfoImpl`. (Per-attempt history is stored on `FilterState` if any filter wants it.)

### 3. `FilterStateImpl` — the typed, lifespan-scoped store

The "side channel" filters use to pass typed objects to each other or to access loggers.

- **Key** is `absl::string_view` (caller-owned literal or stored string).
- **Value** is a `std::shared_ptr<FilterState::Object>`. Any class deriving from `Object` can be stored.
- **Lifespan** controls how long the value lives:

  ```
  Filter (per-filter scratch, never propagated)
    → FilterChain (default; lives as long as the HTTP request)
      → Request (across recreate-stream / internal redirect)
        → DownstreamRequest
          → DownstreamConnection
            → TopSpan (longest available — usually the conn)
  ```

  `setData()` at a deeper lifespan than the current store **walks up** the `parent_` chain (creating the parent
  store on demand) and writes there, so all subsequent children see the same value.

- **Mutability**: `READ_ONLY` vs `MUTABLE`. Re-`setData` of the same key fails (or returns an error status) unless
  the existing entry is `MUTABLE` and `StateType::Mutable` is passed again.

- **Shared with upstream**: `objectsSharedWithUpstreamConnection()` flags objects that should be replicated to
  the per-upstream-attempt `FilterState` so upstream filters can read them.

The four small concrete `FilterState::Object`s (`BoolAccessorImpl`, `Uint32AccessorImpl`, `Uint64AccessorImpl`,
`UpstreamAddress`) cover the trivial cases (single primitive value or `Network::Address` ptr) so callers don't
need to define a one-off class every time.

### 4. Utilities (`utility.h`)

- **`ResponseFlagUtils`** — the registry of all response flags. Owns both the canonical built-in flags
  (`NoHealthyUpstream`, `UpstreamConnectionFailure`, `RateLimited`, …) and a runtime registry for extensions that
  call `ResponseFlagUtils::CustomFlag`. Provides:
  - `toShortString(stream_info)` — used by `%RESPONSE_FLAGS%` formatter.
  - `appendString` / `setResponseFlag` helpers.
  - The numeric IDs that match the wire-level enum used in access-log proto.

- **`TimingUtility`** — wraps `DownstreamTiming` and `UpstreamTiming` and gives one place to compute things like
  *first upstream tx duration*, *first downstream rx-to-tx duration*, etc. Used by formatters and access logs so
  the math doesn't have to be redone everywhere.

- **`ProxyStatusUtils`** — RFC 9209 `Proxy-Status` header serializer. Maps Envoy's `ResponseFlag` set into the
  `proxy_status` enum (`http_request_error`, `connection_refused`, …), composes the header value with optional
  `details=`, `e_ts=`, etc. parameters per configuration.

---

## Construction, lifetime, ownership

```mermaid
sequenceDiagram
    autonumber
    participant L as Listener / NetworkFilterManager
    participant CB as Codec / HCM
    participant SI as StreamInfoImpl
    participant FS as FilterStateImpl

    L->>L: accept connection; create conn-level StreamInfoImpl(Connection lifespan FilterState)
    CB->>SI: new StreamInfoImpl(protocol, ts, conn_info, FilterChain, parent_filter_state=conn FS)
    SI->>FS: own a FilterChain-lifespan FilterStateImpl; parent_ = conn FS
    Note over SI: filters write / read SI fields during request
    CB->>SI: onRequestComplete() → final_time_ = monotime now
    CB-->>L: stream gone; SI destroyed (FilterChain FS shared_ptr decremented)
    L-->>L: connection closes; conn-level SI destroyed too
```

`StreamInfoImpl` is held by **unique_ptr** in the codec stream; everything that wants to read it long-term holds a
**raw pointer** that is valid for the same callback context. Access logs read it *before* destruction. The
`FilterStateImpl` is **shared_ptr** so objects in it can outlive a particular child if the parent still holds the
reference (this is what makes `Lifespan::Request` work across recreate-stream).

---

## Recreate-stream / internal redirect

When the router decides to internally redirect, it calls
`Http::ConnectionManagerImpl::doEndStream` → recreate flow. The new stream constructs a *new* `StreamInfoImpl` but
passes:

- The same `start_time_` (so the user-visible duration is end-to-end).
- The original `FilterState` at `Lifespan::Request` (so filters can plant a flag that survives the redirect).
- The same `RequestIDExtension` and `requestID()` (so log correlation still works).

`StreamInfoImpl::setFromForRecreateStream(other)` is the helper that carries forward the bits that should persist.

---

## Response flags

```mermaid
flowchart LR
    Plug[filter / router / etc.] -->|setResponseFlag(F)| SI[StreamInfoImpl]
    SI -->|getResponseFlags()| Bits[absl::flat_hash_set<ResponseFlag>]
    Bits --> Fmt[%RESPONSE_FLAGS% formatter]
    Bits --> RFU[ResponseFlagUtils::toShortString]
    RFU --> Log[access log line]
    Bits --> Px[ProxyStatusUtils → Proxy-Status header]
    Ext[extension calls CustomFlag()] --> Reg[ResponseFlagUtils registry]
    Reg -.lookups by name/id.-> RFU
```

Built-in flags live in `envoy/stream_info/stream_info.h`. **Custom** flags must:

1. Pick a 32-bit ID that does not collide with the built-in `ResponseFlag` enum.
2. Register via `ResponseFlagUtils::CustomFlag(name, id)` at static-init time (typically in the extension's
   `StaticRegistration`).
3. Use `StreamInfo::setResponseFlag(flag)` from the extension.

`responseFlags()` returns the **bitset-style hash set**; the formatters and `Proxy-Status` writer consume that
to render the human-facing string.

---

## What this folder explicitly does **not** do

- **No I/O** — addresses are stored, not resolved here; SSL info is stored, not done here.
- **No formatting** — `formatter/` is its own world; we just provide the typed slots and the
  `ResponseFlagUtils::toShortString` helper that the formatter calls.
- **No persistence / xDS** — `StreamInfo` is per-stream RAM only.
- **No thread safety** — single-thread, single-stream, deal with it.

If you need any of those, look in `formatter/`, `access_log/`, or `network/`.

---

See also:

- [`stream_info_impl.md`](stream_info_impl.md) — the giant header in detail.
- [`filter_state.md`](filter_state.md) — the keyed/lifespan store.
- [`utility.md`](utility.md) — flag + timing + proxy-status helpers.
- [`CLASS_HIERARCHY.md`](CLASS_HIERARCHY.md) — visual UML.
