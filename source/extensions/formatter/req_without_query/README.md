# Req Without Query Formatter

Adds the `%REQ_WITHOUT_QUERY(main:alt)%` substitution command, which
looks up a request header (with optional alternative fallback) and emits
its value with the URL query string stripped. Equivalent to `%REQ%` with
query-string scrubbing.

Proto: `envoy.extensions.formatter.req_without_query.v3.ReqWithoutQuery`
(empty). Factory name: `envoy.formatter.req_without_query`.

## Files
- `req_without_query.h/cc` - `ReqWithoutQuery` (`FormatterProvider`) and
  `ReqWithoutQueryCommandParser`.
- `config.h/cc` - `ReqWithoutQueryFactory` registration class.

## Interface
- Factory base: `Envoy::Formatter::CommandParserFactory`.
- Command parser base: `Envoy::Formatter::CommandParser`.
- Formatter provider base: `Envoy::Formatter::FormatterProvider`.

## Logic
- `parse` handles the command `REQ_WITHOUT_QUERY`, calls
  `SubstitutionFormatUtils::parseSubcommandHeaders` to extract the
  main header name and optional alternative, and constructs a
  `ReqWithoutQuery(main, alt, max_length)`. Errors from the utility
  propagate out via `THROW_IF_NOT_OK_REF`.
- `format()` fetches the main header from the request headers; if
  missing and an alternative header was configured, it falls back to
  that. The resulting value is run through
  `Http::Utility::stripQueryString` (drops everything after the first
  `?`) and then truncated to `max_length`.
- `formatValue()` mirrors the string path but returns
  `ValueUtil::nullValue()` if neither header is present.

## Key decision points
- `req_without_query.cc:54` - if the main header is missing the code
  checks the alternative only when it's non-empty; this prevents an
  empty string configuration from accidentally matching every header.
- `req_without_query.cc:62` - TODO comment references
  https://github.com/envoyproxy/envoy/issues/13454 about logging all
  header values; for now only the first match is used.
- Query-string stripping happens after header selection so both main
  and alternative results are normalised identically.

## Configuration
- Proto is empty; main / alternative header names come from the
  subcommand (`main:alt`).

## Stats / errors
- None. Missing headers produce empty output or `NullValue`.
