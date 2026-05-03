# Stdout / Stderr Access Logger

Thin access-log sinks that write formatted records to the process's standard streams. Both factories delegate to the shared `createStreamAccessLogInstance<...>` helper (`source/extensions/access_loggers/common/stream_access_log_common_impl.h`) which in turn builds a `FileAccessLog` targeted at a `Filesystem::DestinationType::Stdout` or `::Stderr` handle. The underlying writer is the same `Filesystem::AsyncFileManager` used by the `file` access logger - so the stream loggers inherit identical formatter support (`log_format`, text/JSON, custom commands) and the same async-flush semantics with no separate buffering layer here.

Protos:
- `envoy.extensions.access_loggers.stream.v3.StdoutAccessLog`
- `envoy.extensions.access_loggers.stream.v3.StderrAccessLog`

See `api/envoy/extensions/access_loggers/stream/v3/stream.proto`. Factory names: `envoy.access_loggers.stdout` and `envoy.access_loggers.stderr` (config.cc:38, 61). Legacy aliases `envoy.stdout_access_log` / `envoy.stderr_access_log` are kept for back-compat (config.cc:43, 66).

## Files
- `config.h` / `config.cc` - `StdoutAccessLogFactory` and `StderrAccessLogFactory`, both `AccessLog::AccessLogInstanceFactory`s.

The concrete `AccessLog::Instance` class (`FileAccessLog`) and the `createStreamAccessLogInstance` helper live outside this directory:
- `source/extensions/access_loggers/common/file_access_log_impl.{h,cc}` - `FileAccessLog`.
- `source/extensions/access_loggers/common/stream_access_log_common_impl.h` - the templated factory helper both stream factories share.

## Interface
- `AccessLog::Instance::log(...)` -> `FileAccessLog::emitLog(...)` - writes the formatted record bytes to the configured `DestinationType` (Stdout/Stderr) through `Filesystem::AsyncFileManager`.

## Flow
1. Config load (config.cc:23 / config.cc:46): the factory downcasts into its proto (`StdoutAccessLog` or `StderrAccessLog`), then calls `createStreamAccessLogInstance<ProtoType, DestinationType>(config, filter, context, command_parsers)`.
2. `createStreamAccessLogInstance` builds the substitution formatter from `log_format`, picks the stream destination type, and constructs a `FileAccessLog` with a `Filesystem::FilePathAndType{DestinationType::Stdout|Stderr, ""}`.
3. Records go through `AccessLog::Instance::log` -> formatter -> `FileAccessLog::emitLog` -> async write to the destination file descriptor. There is no in-process batching; the `AsyncFileManager` handles queueing and flushing to the FD.

## Key decision points
- `config.cc:27-30` / `config.cc:50-53` - the only difference between the two factories is the `Filesystem::DestinationType` template argument.
- `config.cc:43` and `config.cc:66` - `LEGACY_REGISTER_FACTORY` preserves the older names (`envoy.stdout_access_log`, `envoy.stderr_access_log`) so existing configs keep working.
- Heavy lifting (formatter parsing, file manager plumbing, async write path) lives in `source/extensions/access_loggers/common/` - this directory contributes only the factory registrations.

## Configuration
```yaml
access_log:
- name: envoy.access_loggers.stdout
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
    log_format:
      text_format_source:
        inline_string: "%START_TIME% %REQ(:METHOD)% %REQ(:PATH)% %RESPONSE_CODE%\n"
- name: envoy.access_loggers.stderr
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StderrAccessLog
    log_format:
      json_format:
        time: "%START_TIME%"
        status: "%RESPONSE_CODE%"
```

## Stats / errors
- No extension-specific stats. Any I/O errors surface via the shared `AsyncFileManager` counters (see `source/common/filesystem/`).
- Config-time failures (bad format strings, invalid proto) throw in `createStreamAccessLogInstance` during listener config load.
