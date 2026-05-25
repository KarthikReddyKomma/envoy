# `utility.{h,cc}` — Response flags, timing, proxy-status, address helpers

`utility.h` is a single header with **four** utility classes, all stateless except for the (static) custom
response flag registry:

- **`ResponseFlagUtils`** — registry & formatter for response flags.
- **`CustomResponseFlag` + `REGISTER_CUSTOM_RESPONSE_FLAG`** — the static-init handshake for extensions to add
  their own flags.
- **`TimingUtility`** — thin wrapper over `StreamInfo` that exposes "how long did X take" as
  `chrono::nanoseconds` optionals.
- **`Utility`** (the literal name) — string-formatting helpers for downstream addresses (used by formatters and
  log helpers).
- **`ProxyStatusUtils`** — RFC 9209 `Proxy-Status` header builder.

---

## Response flags

### The built-in registry (`CORE_RESPONSE_FLAGS`)

```cpp
constexpr static std::array CORE_RESPONSE_FLAGS{
  FlagStrings{"LH",  "FailedLocalHealthCheck",     CoreResponseFlag::FailedLocalHealthCheck},
  FlagStrings{"UH",  "NoHealthyUpstream",          CoreResponseFlag::NoHealthyUpstream},
  FlagStrings{"UT",  "UpstreamRequestTimeout",     CoreResponseFlag::UpstreamRequestTimeout},
  FlagStrings{"LR",  "LocalReset",                 CoreResponseFlag::LocalReset},
  FlagStrings{"UR",  "UpstreamRemoteReset",        CoreResponseFlag::UpstreamRemoteReset},
  FlagStrings{"UF",  "UpstreamConnectionFailure",  CoreResponseFlag::UpstreamConnectionFailure},
  FlagStrings{"UC",  "UpstreamConnectionTermination", CoreResponseFlag::UpstreamConnectionTermination},
  FlagStrings{"UO",  "UpstreamOverflow",           CoreResponseFlag::UpstreamOverflow},
  FlagStrings{"NR",  "NoRouteFound",               CoreResponseFlag::NoRouteFound},
  FlagStrings{"DI",  "DelayInjected",              CoreResponseFlag::DelayInjected},
  FlagStrings{"FI",  "FaultInjected",              CoreResponseFlag::FaultInjected},
  FlagStrings{"RL",  "RateLimited",                CoreResponseFlag::RateLimited},
  FlagStrings{"UAEX","UnauthorizedExternalService",CoreResponseFlag::UnauthorizedExternalService},
  FlagStrings{"RLSE","RateLimitServiceError",      CoreResponseFlag::RateLimitServiceError},
  FlagStrings{"DC",  "DownstreamConnectionTermination", CoreResponseFlag::DownstreamConnectionTermination},
  FlagStrings{"URX", "UpstreamRetryLimitExceeded", CoreResponseFlag::UpstreamRetryLimitExceeded},
  FlagStrings{"SI",  "StreamIdleTimeout",          CoreResponseFlag::StreamIdleTimeout},
  FlagStrings{"IH",  "InvalidEnvoyRequestHeaders", CoreResponseFlag::InvalidEnvoyRequestHeaders},
  FlagStrings{"DPE", "DownstreamProtocolError",    CoreResponseFlag::DownstreamProtocolError},
  FlagStrings{"UMSDR","UpstreamMaxStreamDurationReached", CoreResponseFlag::UpstreamMaxStreamDurationReached},
  FlagStrings{"RFCF","ResponseFromCacheFilter",    CoreResponseFlag::ResponseFromCacheFilter},
  FlagStrings{"NFCF","NoFilterConfigFound",        CoreResponseFlag::NoFilterConfigFound},
  FlagStrings{"DT",  "DurationTimeout",            CoreResponseFlag::DurationTimeout},
  FlagStrings{"UPE", "UpstreamProtocolError",      CoreResponseFlag::UpstreamProtocolError},
  FlagStrings{"NC",  "NoClusterFound",             CoreResponseFlag::NoClusterFound},
  FlagStrings{"OM",  "OverloadManagerTerminated",  CoreResponseFlag::OverloadManager},
  FlagStrings{"DF",  "DnsResolutionFailed",        CoreResponseFlag::DnsResolutionFailed},
  FlagStrings{"DO",  "DropOverload",               CoreResponseFlag::DropOverLoad},
  FlagStrings{"DR",  "DownstreamRemoteReset",      CoreResponseFlag::DownstreamRemoteReset},
  FlagStrings{"UDO", "UnconditionalDropOverload",  CoreResponseFlag::UnconditionalDropOverload},
};
```

The short string (`"UH"`, etc.) is what `%RESPONSE_FLAGS%` emits by default. The long PascalCase string
(`NoHealthyUpstream`, etc.) is what `%RESPONSE_FLAGS_LONG%` emits.

### `toShortString(stream_info)` algorithm

```mermaid
flowchart TD
    Start[toShortString SI] --> A{any flag set?}
    A -- no --> EmptyDash[return "-"]
    A -- yes --> B[buf = ""]
    B --> C[for each FlagStrings in CORE_RESPONSE_FLAGS]
    C --> D{SI.hasResponseFlag flag?}
    D -- yes --> E[append short_string and ","]
    D -- no --> C
    C --> F[for each custom registered flag]
    F --> G{SI.hasResponseFlag custom_flag?}
    G -- yes --> H[append custom short_string and ","]
    G -- no --> F
    F --> I[strip trailing ","]
    I --> Done[return buf]
```

The order is the iteration order of `CORE_RESPONSE_FLAGS` (then custom, in registration order), so it's stable
across runs and the access log line is greppable.

### Custom flag registration

```cpp
// In some_extension.cc
REGISTER_CUSTOM_RESPONSE_FLAG(MYF, MyFancyFlag);

// Anywhere in the same TU:
stream_info.setResponseFlag(CUSTOM_RESPONSE_FLAG(MYF));
```

What that expands to:

1. `static CustomResponseFlag registered_MYF{"MYF", "MyFancyFlag"};` — at constructor time it calls
   `ResponseFlagUtils::registerCustomFlag("MYF", "MyFancyFlag")` which assigns the next free 16-bit ID and
   stores `{flag, "MyFancyFlag"}` into the singleton `mutableResponseFlagsMap()`.
2. `CUSTOM_RESPONSE_FLAG(MYF)` evaluates to `registered_MYF.flag()` — the `ResponseFlag` wrapping that ID.

Restrictions documented in the header:

- The macro **must be in a source file** (one TU; not a header).
- Don't use it to initialize another static (registration order between TUs is undefined; the lookup map may
  not exist yet).
- To share across TUs, expose a function:
  ```cpp
  // header.h
  ResponseFlag getMyRegisteredFlag();
  // source.cc
  REGISTER_CUSTOM_RESPONSE_FLAG(MYF, MyFancyFlag);
  ResponseFlag getMyRegisteredFlag() { return CUSTOM_RESPONSE_FLAG(MYF); }
  ```

### `toResponseFlag(string_view)`

Reverse lookup — used by the access-log filter to parse `%RESPONSE_FLAGS%` patterns like
`response_flags(UH,UT)`. Looks at both `CORE_RESPONSE_FLAGS` and the custom map.

---

## `TimingUtility`

Convenience wrapper that gives **one place** for "how long from request-start until X happened" computations:

```cpp
TimingUtility(stream_info)
    .firstUpstreamTxByteSent()         // Δ from start_time_monotonic_
    .firstUpstreamRxByteReceived()
    .upstreamHandshakeComplete()
    .firstDownstreamTxByteSent()
    .lastDownstreamAckReceived()
    .downstreamHandshakeComplete();
```

Each method:

1. Looks at the right `UpstreamTiming` / `DownstreamTiming` member (returns `absl::nullopt` if not set).
2. Returns the **delta from `StreamInfo::startTimeMonotonic()`** as `chrono::nanoseconds`.

Formatters call this via `%REQUEST_DURATION%`, `%RESPONSE_DURATION%`, `%RESPONSE_TX_DURATION%`,
`%DOWNSTREAM_HANDSHAKE_DURATION%`, etc.

### Why nanoseconds optionals?

- Optional because not every stream has every event (e.g., handshake completes only on TLS).
- Nanoseconds because some events happen <1µs apart and the access log formatter does its own
  `duration_cast<microseconds>` / `<milliseconds>`.

---

## `Utility` (downstream address helpers)

Tiny grab-bag of formatting helpers — exists so the formatter and the L4 access logger don't both reimplement
"give me an IP without port" / "give me just the port".

| Function                                              | Returns                                                       |
|-------------------------------------------------------|---------------------------------------------------------------|
| `formatDownstreamAddressNoPort(addr)`                 | `"10.1.2.3"` or `"::1"`                                       |
| `formatDownstreamAddressNoPort(addr, mask=16)`        | `"10.1.0.0/16"` (CIDR-masked for log anonymization)           |
| `formatDownstreamAddressJustPort(addr)`               | `"8443"` (as string, suitable for header value)               |
| `extractDownstreamAddressJustPort(addr)`              | `optional<uint32_t>{8443}`                                    |
| `formatDownstreamAddressJustEndpointId(addr)`         | For `EnvoyInternalAddress`, returns the endpoint id string    |

Masking is used by some compliance-conscious operators who want `%DOWNSTREAM_REMOTE_ADDRESS%` to log a CIDR
instead of the exact client IP.

---

## `ProxyStatusUtils` — RFC 9209 builder

`Proxy-Status` is a response header that lets intermediaries declare "I failed for *this* reason"; specification
in [draft-ietf-httpbis-proxy-status](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-proxy-status).
Envoy emits it from the HCM when configured via `HttpConnectionManager.proxy_status_config`.

### API

```cpp
// 1) Resolve which proxy_status error applies from the StreamInfo response flags:
absl::optional<ProxyStatusError> err = ProxyStatusUtils::fromStreamInfo(stream_info);

// 2) Compose the header string:
std::string h = ProxyStatusUtils::makeProxyStatusHeader(
    stream_info, *err, /*proxy_name*/"envoy-fpr1",
    proxy_status_config);
// → "envoy-fpr1; error=connection_refused; details=\"upstream_connect_failure\"; e_ts=1718"

// 3) (Optional) Map back to a recommended HTTP status code:
absl::optional<Http::Code> code = ProxyStatusUtils::recommendedHttpStatusCode(*err);
```

### Mapping `ResponseFlag` → `ProxyStatusError`

`fromStreamInfo()` walks `responseFlags()` in priority order and returns the *first* applicable
`ProxyStatusError`. The priority chain (excerpt) is roughly:

| Response flag                          | `ProxyStatusError`             |
|----------------------------------------|---------------------------------|
| `DnsResolutionFailed`                  | `dns_error`                    |
| `NoHealthyUpstream`                    | `destination_unavailable`      |
| `NoClusterFound`                       | `destination_not_found`        |
| `UpstreamConnectionFailure`            | `connection_refused`           |
| `UpstreamConnectionTermination`        | `connection_terminated`        |
| `UpstreamRequestTimeout`               | `http_response_timeout`        |
| `UpstreamProtocolError`                | `http_protocol_error`          |
| `DownstreamProtocolError`              | `http_request_error`           |
| `InvalidEnvoyRequestHeaders`           | `http_request_denied`          |
| `RateLimited` / `OverloadManager`      | `proxy_internal_response`      |
| (no match)                             | `nullopt`                       |

### Proxy name resolution

```cpp
std::string name = ProxyStatusUtils::makeProxyName(node_id, server_name, &proxy_status_config);
```

- If `proxy_status_config.use_node_id` → `node_id`.
- Else if `proxy_status_config.literal_proxy_name` is set → that literal.
- Else → `server_name` (the Envoy server-name header value).

### Optional parameters

`makeProxyStatusHeader` will append `details="..."` (taking `stream_info.responseCodeDetails()` and escaping
quotes), `e_ts=<seconds-since-stream-start>` for the elapsed time, and any other configured params, all per
the proxy-status config flags.

### `recommendedHttpStatusCode`

Per the RFC: each error has a recommended status code (e.g., `connection_refused → 502`,
`http_response_timeout → 504`). HCM consults this when it has to synthesize a local response and the operator
opted-in to RFC-conformant codes.

---

## Cross-references

- The **flag enum values** themselves live in `envoy/stream_info/stream_info.h` (`CoreResponseFlag`).
- The **formatter strings** (`%RESPONSE_FLAGS%`, `%PROXY_STATUS%`) are wired in
  `source/common/formatter/stream_info_formatter.cc`.
- The **HCM glue** that actually adds the `Proxy-Status` header to outgoing responses is in
  `source/common/http/conn_manager_utility.cc::mutateResponseHeaders()`.
- The **access log proto mapping** between numeric `ResponseFlag` IDs and the
  `envoy.data.accesslog.v3.AccessLogCommon` enum is in
  `source/common/access_log/grpc_access_log_utils.cc`.
