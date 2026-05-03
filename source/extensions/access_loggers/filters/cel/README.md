# CEL Access Log Filter

Access-log *filter* extension (not a logger). Decides whether a given log record is emitted by evaluating a CEL (Common Expression Language) expression against the request/response context and returning a boolean.

Proto: `envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter`.

## Files
- `config.h/cc` — `CELAccessLogExtensionFilterFactory` registered under `envoy.access_loggers.extension_filters.cel` as an `AccessLog::ExtensionFilterFactory`. Parses the expression at config load (using the CEL text parser behind `USE_CEL_PARSER`) and constructs the filter.
- `cel.h/cc` — `CELAccessLogExtensionFilter` (`AccessLog::Filter`). Compiles the parsed expression via `Filters::Common::Expr::CompiledExpression::Create` and evaluates it per-record.

## Sink / logger role
Implements `AccessLog::Filter::evaluate()` — not `AccessLog::Instance`. The filter is attached to any access logger via the common `ExtensionFilter` hook and gates calls to the logger's `log()` method.

## Flow
1. Factory calls `google::api::expr::parser::Parse(expression)` at config load; a syntax error becomes an `EnvoyException`.
2. Optional `cel_config` (runtime / activation settings) is threaded through `getBuilder()` so the compiled expression shares Envoy's CEL builder (and any host functions registered there).
3. Constructor calls `CompiledExpression::Create(builder, input_expr)`; an invalid expression throws `EnvoyException` with the compile diagnostic.
4. Per request, `evaluate()` allocates a fresh `Protobuf::Arena`, calls `expr_.evaluate(arena, &local_info, stream_info, request_headers, response_headers, response_trailers)`.
5. Returns `true` iff the result is a boolean `true`. Any evaluation error, missing value, or non-bool result returns `false` — i.e. the log record is suppressed on failure.

## Key decision points
- `config.cc:29` — expression parsing (throws on parse error).
- `config.cc:45` — `EnvoyException` raised when built without `USE_CEL_PARSER`.
- `cel.cc:18` — compile-time failure throws `EnvoyException`.
- `cel.cc:31` — errors or non-bool results conservatively suppress the record (`return false`).

## Configuration
- `expression` — CEL text expression (required). Examples: `response.code >= 500`, `request.headers['x-debug'] == 'true'`.
- `cel_config` — optional `envoy.config.core.v3.CelExpressionConfig` selecting CEL runtime options (e.g. host function library).

## Stats / errors
No counters. Surface errors are config-time `EnvoyException`s (parse / compile / CEL not available); runtime errors are swallowed.
