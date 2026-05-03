# File

Writes substitution-formatter records to a local file (path taken from `path`). The on-disk format is chosen between plain text, JSON or typed JSON by the config oneof. This is the most common access log in production deployments.

Proto: `envoy.extensions.access_loggers.file.v3.FileAccessLog`.

## Files
- `config.h/cc` — `FileAccessLogFactory` registered as `envoy.access_loggers.file` (with legacy alias `envoy.file_access_log`). Reads the config, picks the right formatter for the active `access_log_format` oneof case, and constructs a `FileAccessLog`.

The actual logger lives in the sibling `common/` folder (`file_access_log_impl.{h,cc}`, class `AccessLoggers::File::FileAccessLog`), which derives from `Common::ImplBase`.

## Sink / logger role
Implements `AccessLog::Instance::log()` via `Common::ImplBase`. Per-record writes are forwarded to `AccessLog::AccessLogFile::write()`, which already handles buffering and `SIGHUP` reopen.

## Flow
1. Factory builds a `Formatter::FormatterPtr` from whichever oneof case is set:
   - `format` (string): wraps into a `SubstitutionFormatString` inline literal; empty string -> `defaultSubstitutionFormatter()`.
   - `json_format`: `createJsonFormatter(..., typed=false, ...)`.
   - `typed_json_format`: repackaged as `SubstitutionFormatString.json_format` (typed path).
   - `log_format` (the current recommended field): `SubstitutionFormatStringUtils::fromProtoConfig(...)`.
   - unset: default HTTP formatter.
2. Builds a `Filesystem::FilePathAndType{DestinationType::File, path}` and constructs `FileAccessLog`.
3. `FileAccessLog` constructor calls `AccessLogManager::createAccessLog(file_info)`; a bad status is thrown via `THROW_IF_NOT_OK_REF`.
4. On `log()`, `Common::ImplBase` gates with the filter, then `emitLog()` formats and calls `log_file_->write()`.

## Key decision points
- `config.cc:33` — empty `format` falls back to the default formatter (kept for backwards compatibility with the deprecated field).
- `config.cc:47` — `json_format` uses the non-typed JSON formatter; `typed_json_format` routes through the typed JSON path.
- `file_access_log_impl.cc:14` — status check for `createAccessLog()` failure (bad path, open error, ...).

## Configuration
- `path` — filesystem path (required, validated by the proto).
- `log_format` / `format` / `json_format` / `typed_json_format` oneof — output format. Prefer `log_format` (a `SubstitutionFormatString`) for new configs.

## Stats / errors
No counters owned by this logger. Errors:
- Proto validation failure via `downcastAndValidate`.
- File open / access failure thrown from the `FileAccessLog` constructor.

Buffered async writes, file-handle reopen, and disk backpressure handling live inside `AccessLog::AccessLogManager`.
