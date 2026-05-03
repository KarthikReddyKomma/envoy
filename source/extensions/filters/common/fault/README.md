# Fault Config Library (shared filter infrastructure)

Small config-only library that resolves the "what abort / delay / rate limit to apply" question at request time. It lets filters accept either a static fault configured at bootstrap or a per-request override read from trusted inline headers such as `x-envoy-fault-abort-request`, `x-envoy-fault-delay-request`, and `x-envoy-fault-throughput-response`. The library owns no runtime state; every method is called from the data-path filter each time a fault decision is needed.

## Files
- `fault_config.h` - Public types: `HeaderNames`, `HeaderPercentageProvider`, `FaultAbortConfig`, `FaultDelayConfig`, `FaultRateLimitConfig`, with nested `*Provider` strategies for fixed vs. header-driven values.
- `fault_config.cc` - Header parsing logic for abort codes, delay durations, throughput values, and per-request percentage clamping.

## Public interface
- `HeaderNames` (`fault_config.h:19`) - `ConstSingleton` of the six `x-envoy-fault-*` lower-case header keys; `HeaderNames::get().AbortRequest`, `.AbortGrpcRequest`, `.AbortRequestPercentage`, `.DelayRequest`, `.DelayRequestPercentage`, `.ThroughputResponse`, `.ThroughputResponsePercentage`.
- `HeaderPercentageProvider(header_name, default_percentage)` (`fault_config.h:38`) - `percentage(request_headers)` returns either the configured default or `min(header_value, default)` when the request carries a numeric override (`fault_config.cc:14`). The clamp prevents downstream clients from dialing faults above the operator's cap.
- `FaultAbortConfig(proto)` (`fault_config.h:54`) - picks one of three strategies based on the `ErrorTypeCase`:
  - `FixedAbortProvider` for `http_status` and `grpc_status`.
  - `HeaderAbortProvider` for `header_abort`. Reads `x-envoy-fault-abort-request` (HTTP) / `x-envoy-fault-abort-grpc-request` (gRPC) per call.
  - `ERROR_TYPE_NOT_SET` panics.
  Exposes `httpStatusCode`, `grpcStatusCode`, `percentage`, each taking optional `request_headers`.
- `FaultDelayConfig(proto)` (`fault_config.h:152`) - `FixedDelayProvider` or `HeaderDelayProvider` (reads `x-envoy-fault-delay-request`). Exposes `duration(request_headers)` returning `absl::optional<std::chrono::milliseconds>`.
- `FaultRateLimitConfig(proto)` (`fault_config.h:234`) - `FixedRateLimitProvider` or `HeaderRateLimitProvider` (reads `x-envoy-fault-throughput-response`). Exposes `rateKbps(request_headers)`.

## Implementation logic
- Constructors switch on the oneof case (`fault_config.cc:38`, `:98`, `:134`) and allocate the matching `*Provider` via `unique_ptr`. An unset oneof `PANIC`s immediately - callers must validate the proto first.
- Header-based providers use `absl::SimpleAtoi` on the first header value only (HTTP allows duplicates but these are untrusted inline headers, so only index 0 is parsed) (`fault_config.cc:28`, `:70`, `:91`, `:127`, `:160`).
- `HeaderAbortProvider::httpStatusCode` clamps parsed codes to `200 <= code < 600`; anything outside that range returns `absl::nullopt` (`fault_config.cc:74`). `grpcStatusCode` does no range check; the value is just cast to `Grpc::Status::GrpcStatus` (`:81`).
- `HeaderDelayProvider::duration` returns `absl::nullopt` if the header is missing or unparseable (`fault_config.cc:117`).
- `HeaderRateLimitProvider::rateKbps` additionally drops a parsed value of `0`, because a rate of 0 KiB/s has no well-defined behaviour (`fault_config.cc:164`).
- `HeaderPercentageProvider::percentage` keeps the denominator from the config and uses `std::min` with the config numerator as the ceiling (`fault_config.cc:33`).

## Consumers
- `source/extensions/filters/http/fault` - HTTP fault filter uses all three configs plus the per-route runtime percentage gate.
- `source/extensions/filters/network/mongo_proxy` - uses `FaultDelayConfig` to inject delays into MongoDB replies.

## Stats / errors / failure modes
- No stats, no IO, no I/O timers. All failure modes surface as `absl::nullopt` from the getter, which the caller interprets as "do nothing for this request."
- Missing or malformed headers always fall back to either the static config (for percentages) or silent no-op (for absolute values).
- Construction will `PANIC` if the proto's oneof is unset - callers must run `ErrorTypeCase` / `FaultDelaySecifierCase` / `LimitTypeCase` discrimination through `MessageUtil::validate` first (`fault_config.cc:54`, `:112`, `:144`).
- HTTP header parsing only considers the first value; duplicate headers are silently ignored (`fault_config.cc:26`).
