# Proto API Scrubber (`envoy.filters.http.proto_api_scrubber`)

A bi-directional HTTP filter that inspects gRPC request and response payloads and either scrubs individual fields or rejects the whole RPC based on CEL-driven match trees. It uses the `grpc_field_extraction` `MessageConverter` to reassemble framed gRPC messages, then runs `proto_processing_lib::proto_scrubber::ProtoScrubber` using a per-method `FieldChecker` that consults the configured match tree. Method-level matchers can block entire RPCs; field-level matchers decide whether each field is preserved or removed. Non-gRPC traffic passes through untouched.

Proto: `envoy.extensions.filters.http.proto_api_scrubber.v3.ProtoApiScrubberConfig`.

## Files
- `config.h/cc` — `FilterFactoryCreator` (`ExceptionFreeFactoryBase`) that builds a shared `ProtoApiScrubberFilterConfig` via `ProtoApiScrubberFilterConfig::create(...)` and installs a `ProtoApiScrubberFilter` per stream (`config.cc:19-32`). `REGISTER_FACTORY` at `config.cc:34`.
- `filter.h/cc` — `ProtoApiScrubberFilter` extending `Http::PassThroughFilter`; owns request/response `MessageConverter`s, `FieldChecker`s, and lazily-built `ProtoScrubber`s (`filter.h:80-126`).
- `filter_config.h/cc` — thread-safe `ProtoApiScrubberFilterConfig` that loads the descriptor set, builds CEL match trees per method/message/field, exposes type lookups, and holds the `ProtoApiScrubberStats`.
- `scrubbing_util_lib/` — `FieldChecker` adapter that bridges `FieldCheckerInterface` with Envoy's match-tree data model.

## Lifecycle
Installed as a full stream filter (`config.cc:29`: `callbacks.addStreamFilter(...)`). Filter callbacks overridden:
- `decodeHeaders` (`filter.cc:153-192`) — validates headers, runs method-level block, prepares request converter.
- `decodeData` (`filter.cc:194-315`) — accumulates framed messages, lazily creates `request_scrubber_`, scrubs, writes back.
- `encodeHeaders` (`filter.cc:317-337`) — detects gRPC response and prepares response converter.
- `encodeData` (`filter.cc:339-443`) — mirrors `decodeData` on the response side.

## Decision / logic
`decodeHeaders`:
- `filter.cc:156`: increments `total_requests_`.
- `filter.cc:158-164`: if headers don't indicate gRPC, pass through without setting `is_valid_grpc_request_`.
- `filter.cc:166-167`: marks `is_valid_grpc_request_ = true`, increments `total_requests_checked_`.
- `filter.cc:168-175`: `validateMethodName(:path)` — must match `/pkg.Svc/Method`, no wildcards. Failure increments `invalid_method_name_` and calls `rejectRequest` with gRPC status from `absl::Status`.
- `filter.cc:180-182`: `checkMethodLevelRestrictions(headers)` — if the match tree returns `isMatch()`, increments `method_blocked_`, tags the active span `proto_api_scrubber.outcome=blocked`, and rejects with `NotFound` / `"METHOD_BLOCKED"` (`filter.cc:134-146`). `isInsufficientData()` fails open (`filter.cc:127-132`).
- `filter.cc:185-189`: builds `request_msg_converter_` sized by `decoder_callbacks_->bufferLimit()`.

`decodeData`:
- `filter.cc:198-202`: skip if non-gRPC.
- `filter.cc:205-221`: `accumulateMessages` on the converter; any error increments `request_buffer_conversion_error_` and rejects with mapped gRPC status (`Internal`, `FailedPrecondition`, or `ResourceExhausted`).
- `filter.cc:223-227`: incomplete message -> `StopIterationAndBuffer` (unless `end_stream`).
- `filter.cc:234-255`: lazy build `request_scrubber_` via `createRequestProtoScrubber()`; failure increments `request_scrubbing_failed_`.
- `filter.cc:258-275`: skip the final empty `StreamMessage`.
- `filter.cc:277-291`: time and invoke `request_scrubber_->Scrub(...)`; record `request_scrubbing_latency_`. Scrub error is logged + stat but does **not** fail the request (warn + passthrough).
- `filter.cc:293-309`: convert back to buffer via `convertMessageToBuffer`; failure rejects.
- `filter.cc:311`: moves scrubbed bytes into the data buffer.

`encodeHeaders` (`filter.cc:321-328`): bail out if not a gRPC response; otherwise create `response_msg_converter_`.

`encodeData` (mirrors `decodeData` with `response_*` stats):
- `filter.cc:343-345`: no converter -> pass-through.
- `filter.cc:348-356`: buffering errors increment `response_buffer_conversion_error_` and call `rejectResponse`.
- `filter.cc:358-361`: incomplete response body returns `StopIterationNoBuffer`.
- `filter.cc:368-386`: lazy `response_scrubber_` via `createResponseProtoScrubber()`.
- `filter.cc:402-418`: scrub + latency histogram `response_scrubbing_latency_`; failures only warn.
- `filter.cc:420-436`: buffer conversion failure rejects via `rejectResponse`.

`createRequestProtoScrubber` / `createResponseProtoScrubber` (`filter.cc:445-480`) build the per-request `FieldChecker`s (bound to the current method name and request/response headers/trailers) and hand them to `ProtoScrubber` along with the correct `ScrubberContext`.

`rejectRequest` / `rejectResponse` (`filter.cc:482-500`) produce `sendLocalReply` with the mapped HTTP status plus `rc_detail` of the form `proto_api_scrubber_<errorType>{<detail>}`.

## Configuration
- `ProtoApiScrubberConfig` (top-level) — descriptor source, filtering mode (only `OVERRIDE` supported; others rejected during `validateFilteringMode`), and per-method/message/field `RestrictionConfig`s containing `xds.type.matcher.v3.Matcher` trees.
- Per-method entries produce method-level match trees consulted in `checkMethodLevelRestrictions`.
- Field-level entries produce match trees consulted inside `FieldChecker` to decide whether to preserve each field during scrubbing.

No route-specific overrides: the factory only implements `createFilterFactoryFromProtoTyped` (`config.h:25-28`).

## Stats
Defined at `filter_config.h:54-99` under the filter's stats prefix:
- Counters: `request_scrubbing_failed`, `response_scrubbing_failed`, `method_blocked`, `request_buffer_conversion_error`, `response_buffer_conversion_error`, `invalid_method_name`, `total_requests`, `total_requests_checked`.
- Histograms (ms): `request_scrubbing_latency`, `response_scrubbing_latency`.

The filter also tags the active span: `proto_api_scrubber.outcome=blocked` on method block (`filter.cc:139`), `proto_api_scrubber.request_error` / `response_error` on scrub failures (`filter.cc:286`, `filter.cc:411`).

## Factory
`FilterFactoryCreator` (`config.h:18`):
- `ExceptionFreeFactoryBase<ProtoApiScrubberConfig>` registered under `kFilterName = "envoy.filters.http.proto_api_scrubber"` (`filter.h:29`).
- `createFilterFactoryFromProtoTyped` returns `absl::StatusOr<FilterFactoryCb>`, propagating descriptor-load / validation errors from `ProtoApiScrubberFilterConfig::create` via `RETURN_IF_ERROR` (`config.cc:24-26`).
- `REGISTER_FACTORY(FilterFactoryCreator, NamedHttpFilterConfigFactory)` at `config.cc:34`.
