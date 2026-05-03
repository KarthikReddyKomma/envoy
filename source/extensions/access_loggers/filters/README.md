# Access Log Filters

This directory is the home of access-log filter plug-ins: extension implementations of `AccessLog::Filter` that decide whether a given log record should be emitted by any access-log sink (file, gRPC, OTel, fluentd, etc.). Filters are pulled via the `envoy.access_loggers.extension_filters` factory category and wired from the `filter.extension_filter` oneof in the access-log config. Built-in filters (status code, duration, header, runtime, response-flag, not-health-check, and AND/OR/NOT composites) live in `source/common/access_log/access_log_impl.cc`; this tree only hosts the extension-filter plugins that require non-trivial dependencies.

## Children

- `cel/` - CEL (Common Expression Language) boolean expression filter. Proto: `envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter`.
- `process_ratelimit/` - Process-wide deterministic sampling filter ("log 1 of every N"). Proto: `envoy.extensions.access_loggers.filters.process_ratelimit.v3.ProcessRateLimitConfig`.

## Interface

Each filter implements `AccessLog::Filter::evaluate(const Formatter::HttpFormatterContext&, const StreamInfo::StreamInfo&) const` and returns `true` to keep the record or `false` to drop it.

## Registration

Filters are registered as factories of type `AccessLog::ExtensionFilterFactory` so that `AccessLogFactory::fromProto(...)` in `source/common/access_log/access_log_impl.cc` can instantiate them when it encounters an `extension_filter` config. See each child's `config.cc` for the `REGISTER_FACTORY` call.

## Flow

1. Access-log subsystem builds the `FilterPtr` once per logger during listener/HCM config load.
2. For every stream, the logger calls `filter_->evaluate(...)` before formatting.
3. If `evaluate` returns `false`, the record is skipped - no formatting, no I/O.

## Where the generic filters live

Composite filters (`AndFilter`, `OrFilter`, `NotFilter`) and the simple predicates (`StatusCodeFilter`, `DurationFilter`, `RuntimeFilter`, `HeaderFilter`, `ResponseFlagFilter`, `NotHealthCheckFilter`, `TraceableFilter`, `MetadataFilter`, `LogTypeFilter`) are implemented directly in `source/common/access_log/access_log_impl.{h,cc}` rather than as extensions.

## See also

- `ACCESS_LOG_FILTERS.md` - user-facing configuration examples and patterns.
- `source/common/access_log/access_log_impl.cc` - built-in filter implementations and the `AccessLogFactory` entry point.
