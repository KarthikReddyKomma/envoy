# CEL Formatter

Adds `%CEL(expression)%` and `%TYPED_CEL(expression)%` substitution
commands for access logs and headers. Expressions are parsed and compiled
at config time using CEL, then evaluated per-request against stream info,
headers, and local info.

Proto: `envoy.extensions.formatter.cel.v3.Cel` (empty).

## Files
- `cel.h/cc` - `CELFormatter` (`FormatterProvider`) and
  `CELFormatterCommandParser` that recognises `CEL` / `TYPED_CEL`.
- `config.h/cc` - `CELFormatterFactory` (registered
  `CommandParserFactory`) and `BuiltInCELFormatterFactory` (registered
  `BuiltInCommandParserFactory`).

## Interface
- Base: `Envoy::Formatter::FormatterProvider`,
  `Envoy::Formatter::CommandParser`.
- Factory bases: `Envoy::Formatter::CommandParserFactory` and
  `Envoy::Formatter::BuiltInCommandParserFactory`.
- Extension names: `envoy.formatter.cel`,
  `envoy.built_in_formatters.cel`.

## Logic
- `CELFormatterCommandParser::parse` accepts `CEL` or `TYPED_CEL`,
  calls `google::api::expr::parser::Parse(subcommand)`, and constructs a
  `CELFormatter` with the parsed `cel::expr::Expr`, the server's
  `LocalInfo`, a shared CEL builder, and the `typed` flag.
- `CELFormatter` compiles the expression once in its initializer list
  via `Expr::CompiledExpression::Create`; failure throws at config
  time rather than per-request.
- `format()` evaluates the compiled expression against request headers,
  response headers, response trailers, and stream info, returning the
  CEL value via `Expr::print`. When a `max_length` was supplied the
  resulting string is truncated.
- `formatValue()` either returns the CEL value serialized as
  `Protobuf::Value` (when `TYPED_CEL`) or wraps the `format()` string
  in `ValueUtil::stringValue`.

## Key decision points
- `config.cc:15` - the typed `createCommandParserFromProto` entry logs a
  warning because CEL is now a built-in formatter; configuring it
  explicitly is unnecessary.
- `cel.cc:25` - compilation failures convert CEL's `absl::Status` into
  an `EnvoyException`, so bad expressions block bootstrap instead of
  silently producing empty log fields.
- `cel.cc:66` - when the CEL value is a string and `max_length` is set,
  the typed output is truncated before reflection to match the string
  formatter's semantics.
- The entire CEL functionality is behind `USE_CEL_PARSER`; builds
  without it throw `EnvoyException` at parser creation time.

## Configuration
- No proto fields; the command parser reads the expression from the
  substitution subcommand.

## Stats / errors
- None. Evaluation failures return `absl::nullopt` / `NullValue` (the
  log gets the empty placeholder).
