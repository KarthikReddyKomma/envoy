# XFCC Value Formatter

Adds the built-in `%XFCC_VALUE(key)%` substitution command. Parses the
`x-forwarded-client-cert` (XFCC) header and extracts the value of one
specific key (`by`, `hash`, `cert`, `chain`, `subject`, `uri`, or
`dns`), respecting the XFCC quoting and escaping rules.

## Files
- `xfcc_value.h/cc` - `XfccValueFormatterCommandParser`,
  `XfccValueFormatterProvider`, and the
  `XfccValueCommandParserFactory` registration class.

## Interface
- Factory base: `Envoy::Formatter::BuiltInCommandParserFactory`.
- Command parser base: `Envoy::Formatter::CommandParser`.
- Formatter provider base: `Envoy::Formatter::FormatterProvider`.
- Factory name: `envoy.built_in_formatters.xfcc_value`.

## Logic
- Supported keys live in a `CONSTRUCT_ON_FIRST_USE` `absl::flat_hash_set`
  in lower case. `parse()` rejects an empty subcommand with
  `EnvoyException("requires a subcommand")` and rejects unsupported keys
  at parse time.
- `XfccValueFormatterProvider::format`:
  1. Pulls the XFCC value via
     `headers.getForwardedClientCertValue()`.
  2. `parseValueFromXfccByKey` scans the header left-to-right, splitting
     on `,` that aren't inside double-quotes (quote state is tracked
     alongside a `countBackslashes` check so escaped quotes are
     respected). The first matching XFCC element wins, modelling
     "oldest proxy" semantics.
  3. `parseElementForKey` splits each element on `;`, again respecting
     quotes, and `parseKeyValuePair` matches the requested key
     case-insensitively.
  4. Values are optionally unquoted and then unescaped (`\"` -> `"`,
     `\\` -> `\`), with a fast path when no backslashes are present.
- Parse failures result in an `absl::InvalidArgumentError` being logged
  at `debug` level and `absl::nullopt` being returned.

## Key decision points
- `xfcc_value.cc:27` - `countBackslashes` walks backwards from a quote
  character to decide whether it's escaped, so `\\"` terminates a
  quoted value but `\"` does not.
- `xfcc_value.cc:104` - leftmost element wins, per the XFCC spec, since
  Envoy assumes the oldest proxy entry contains the original client
  certificate.
- `xfcc_value.cc:136` - an `ASSERT(!in_quotes)` in the element scanner
  would trip if the outer parser missed an unmatched quote; the outer
  function rejects unmatched quotes first so this branch is truly
  unreachable.
- `xfcc_value.cc:222` - `max_length` is accepted but ignored (the
  parameter name is anonymous): the XFCC value has a bounded set of
  supported keys so truncation is not applied.

## Configuration
- None. Built-in parser registered at compile time.

## Stats / errors
- None. Parse errors are logged at debug level; callers see
  `absl::nullopt`.
