# CEL Expression Evaluator (shared filter infrastructure)

Shared C++ library that compiles and evaluates Common Expression Language (CEL) expressions against an Envoy stream. It is the runtime behind any filter or subsystem that advertises "attribute" expressions: RBAC matchers, `ext_proc` attribute extraction, CEL access log filters / formatters, the CEL input matcher, rate-limit descriptor producers, the OpenTelemetry CEL sampler, and the Wasm ABI attribute lookup. The library wraps Google's `cel-cpp` runtime, owns the process-wide builder cache, exposes a `StreamActivation` that binds request/response/connection/xDS data to CEL identifiers, and provides a `CelState` filter-state object so scripts can persist typed values on a stream.

## Files
- `evaluator.h/cc` - Builder factory + cache, `StreamActivation`, `CompiledExpression`, and the `CelException` type.
- `context.h/cc` - CEL wrappers (`RequestWrapper`, `ResponseWrapper`, `ConnectionWrapper`, `UpstreamWrapper`, `PeerWrapper`, `FilterStateWrapper`, `XDSWrapper`) plus the `*LookupValues` singletons keyed on constant attribute strings (`path`, `headers`, `code`, `mtls`, `cluster_name`, ...).
- `cel_state.h/cc` - `CelStatePrototype` and `CelState` filter-state object used to expose user-owned bytes/string/protobuf/flatbuffer blobs back to CEL via `exprValue()`.

## Public interface
- `BuilderConstPtr createBuilder(OptRef<CelExpressionConfig>, Protobuf::Arena* = nullptr)` - builds a `CelExpressionBuilder` with Envoy's security-oriented defaults (`enable_comprehension=false`, `regex_max_program_size=100`, list concat off) and registers built-in/regex/string extensions (`evaluator.cc:107`). Passing an arena enables constant folding for the legacy RBAC path.
- `BuilderInstanceSharedConstPtr getBuilder(CommonFactoryContext&, OptRef<CelExpressionConfig>)` - looks up or creates a cached builder via the `BuilderCache` singleton (`evaluator.cc:194`).
- `CompiledExpression::Create(...)` overloads accepting `cel::expr::Expr`, `xds::type::v3::CelExpression`, and the deprecated `google.api.expr.v1alpha1.Expr` - returns `absl::StatusOr` instead of throwing (`evaluator.cc:211`, `:225`, `:246`).
- `CompiledExpression::evaluate(Protobuf::Arena&, local_info, stream_info, request_headers, response_headers, response_trailers)` - builds an activation on the fly, runs the expression, returns `absl::optional<CelValue>` (`evaluator.cc:261`).
- `CompiledExpression::matches(stream_info, request_headers)` - convenience wrapper returning `bool` (`evaluator.cc:281`).
- `ActivationPtr createActivation(...)` - standalone activation factory; callers can reuse it across multiple expressions sharing the same stream (`evaluator.cc:98`).
- `std::string print(CelValue)` - stringifies results, honouring `envoy.reloadable_features.cel_message_serialize_text_format` (`evaluator.cc:292`).
- `CelState::exprValue(arena, last)` - materializes a stored blob as `CelValue` (string/bytes/flatbuffers/protobuf) for CEL access (`cel_state.cc:15`).

## Implementation logic
- `StreamActivation::FindValue` dispatches on a 10-entry `ActivationLookupTable` (`evaluator.cc:35`) and creates the appropriate wrapper on the caller's arena. `Response` lookups flip `needs_response_path_data_` so filters can detect whether trailers/headers are required (`evaluator.cc:62`).
- `BuilderCache` hashes the proto config with `MessageUtil::hash` and keeps `weak_ptr`s so builders are freed when the last xDS consumer drops the reference (`evaluator.cc:166`). Must run on the main thread; both `createBuilder` and `getOrCreateBuilder` assert via `ASSERT_IS_MAIN_OR_TEST_THREAD()` (`evaluator.cc:109`, `:168`).
- `CompiledExpression` owns `builder_` and `source_expr_` so its compiled `expr_` never dangles (`evaluator.h:172`). `Create` copies the incoming `cel::expr::Expr` before compilation because the cel-cpp runtime stores pointers back into the source tree.
- Wrappers use `HeadersWrapper<T>` to expose `request.headers["..."]` with lazy key iteration; header name validation rejects invalid keys (`context.h:183`).
- `CelState::exprValue` short-circuits to `CreateBytes` when `last` is true (terminal leaf), otherwise deserializes protobuf or parses flatbuffers reflection on the arena (`cel_state.cc:22`, `:33`).

## Consumers
- HTTP filters: `rbac` (`source/extensions/filters/common/rbac/matchers.h`), `ext_proc` (`source/extensions/filters/http/ext_proc/matching_utils.cc`).
- Network filters: the network `rbac` filter via the same `filters/common/rbac` matchers.
- Access loggers: `access_loggers/filters/cel`, formatters in `formatter/cel`.
- Matching framework: `matching/http/cel_input`, `matching/input_matchers/cel_matcher`.
- Wasm ABI attribute lookup: `common/wasm/context.cc`.
- Rate-limit descriptors: `rate_limit_descriptors/expr`.
- Tracing: `tracers/opentelemetry/samplers/cel`.
- ext_proc request modifiers: `http/ext_proc/processing_request_modifiers/mapped_attribute_builder`.

## Stats / errors / failure modes
- No stats are emitted here. Compilation problems surface as `CelException` (`evaluator.h:185`) from `createBuilder` when built-in/regex/string registration fails, or as `absl::Status` from `CompiledExpression::Create`.
- Evaluation errors are swallowed: `evaluate()` returns `absl::nullopt` on cel-cpp errors; `matches()` returns `false` if the expression fails or does not yield a bool (`evaluator.cc:268`, `:288`). Callers that need error visibility must use the `(Activation&, Arena*)` overload which returns `absl::StatusOr<CelValue>`.
- Builder-cache misuse (calling off the main thread) trips the thread assertion and aborts the process in debug builds.
