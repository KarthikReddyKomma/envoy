# File Content Formatter

Adds the `%FILE_CONTENT(path)%` /
`%FILE_CONTENT(path:watch_directory)%` substitution command. Reads the
contents of a file on the filesystem (optionally watching a directory for
changes) and substitutes that data into access logs or headers.

Proto: `envoy.extensions.formatter.file_content.v3.FileContent` (empty).
Factory name: `envoy.formatter.file_content`.

## Files
- `config.h/cc` - `FileContentFormatterFactory` registration plus the
  anonymous-namespace `FileContentFormatterProvider` (wraps a
  `DataSourceProvider<std::string>`) and `FileContentCommandParser`.

## Interface
- Factory base: `Envoy::Formatter::CommandParserFactory`.
- Command parser base: `Envoy::Formatter::CommandParser`.
- Formatter provider base: `Envoy::Formatter::FormatterProvider`.

## Logic
- `FileContentCommandParser::parse` recognises the command
  `FILE_CONTENT`, splits the subcommand on `:` to separate the file path
  from the optional watch directory, and builds a
  `envoy::config::core::v3::DataSource` from the parts.
- A `DataSourceProvider<std::string>` is created with
  `allow_empty = true` and `modify_watch = true`, plus a transform
  lambda that calls
  `Envoy::Formatter::SubstitutionFormatUtils::truncate` so the cached
  value already respects `max_length`.
- `FileContentFormatterProvider::format` pulls the current data from the
  provider; if the underlying read is unavailable it returns
  `absl::nullopt`. `formatValue` mirrors this into `Protobuf::Value`.
- `ASSERT_IS_MAIN_OR_TEST_THREAD()` enforces that parsing happens on the
  main thread since the provider creates thread locals.

## Key decision points
- `config.cc:74` - the parser enforces exactly 1 or 2 colon-separated
  parts; anything else throws `EnvoyException` at config time so typos
  don't silently produce empty output.
- `config.cc:84` - `allow_empty = true` means an empty file yields an
  empty (but present) string; `modify_watch = true` wires the provider
  to the filesystem watcher so updates are picked up.
- Truncation happens inside the provider transform lambda, not at
  format time, so the cached value is already bounded and format
  cost stays O(1).

## Configuration
- No proto fields; the path (and optional watch dir) come from the
  substitution subcommand.

## Stats / errors
- None. Parse-time failures throw; runtime read failures return
  `absl::nullopt`.
