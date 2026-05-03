# Dynamic Modules

Access logger that delegates every record to a user-supplied dynamic shared object loaded through the Envoy Dynamic Modules ABI. The module decides how to serialize, batch, and ship records; this folder is the glue that loads the `.so`, resolves the callbacks, and forwards per-worker log events.

Proto: `envoy.extensions.access_loggers.dynamic_modules.v3.DynamicModuleAccessLog`.

## Files
- `config.h/cc` — `DynamicModuleAccessLogFactory` registered under `envoy.access_loggers.dynamic_modules`. `createAccessLogInstance()` loads the module by name, extracts `logger_config` (via `MessageUtil::knownAnyToBytes`), resolves symbols through `newDynamicModuleAccessLogConfig()`, and optionally registers a custom stat namespace behind the `dynamic_modules_strip_custom_stat_prefix` runtime guard.
- `access_log.h/cc` — `DynamicModuleAccessLog` (subclass of `Common::ImplBase`) plus `ThreadLocalLogger`, which wraps one module-created logger handle per worker in a thread-local slot.
- `access_log_config.h/cc` — `DynamicModuleAccessLogConfig` shared across all instances bound to the same `.so`: holds resolved ABI callbacks (`on_logger_new_`, `on_logger_log_`, `on_logger_flush_`, `on_logger_destroy_`, `on_config_destroy_`), the opaque in-module config pointer, and `ModuleCounterHandle` / `ModuleGaugeHandle` / `ModuleHistogramHandle` tables that map 1-based ABI IDs back to Envoy stats.
- `abi_impl.cc` — C bridge the module calls back into (stream-info getters, metric ops) — see this file for the Envoy-side implementation of the ABI.

## Sink / logger role
Implements `AccessLog::Instance::log()` via `Common::ImplBase`, so filter evaluation happens in the base class and records that pass are handed to the module's `on_logger_log_` callback.

## Flow
1. Factory loads the module (`newDynamicModuleByName`, honouring `do_not_close` and `load_globally`).
2. `newDynamicModuleAccessLogConfig()` dlsym-resolves the required ABI symbols; failure surfaces as an `EnvoyException`.
3. `DynamicModuleAccessLog` constructor allocates a TLS slot, and the TLS initializer on each worker calls `on_logger_new_(in_module_config_, tl_logger_ptr)` — the void pointer is stashed so ABI callbacks can round-trip to the worker context.
4. On each log, `emitLog()` parks `context` and `stream_info` on the thread-local logger, converts `AccessLogType` to `envoy_dynamic_module_type_access_log_type`, and invokes `on_logger_log_(this_ptr, module_logger, abi_type)`.
5. During shutdown, `~ThreadLocalLogger` calls `on_logger_flush_` (if provided) then `on_logger_destroy_`. When the last config reference drops, `~DynamicModuleAccessLogConfig` invokes `on_config_destroy_`.

## Key decision points
- `access_log.cc:31` — worker index derived from dispatcher name (`worker_N`); main/test thread assigned `concurrency` as a free index.
- `access_log.cc:55` — silently drops the record if the worker has no module logger (null `logger_`).
- `access_log.cc:18` — flush callback is best-effort (only called if the module exposed it).
- `config.cc:60` — legacy prometheus compatibility behind `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix`.
- `access_log_config.h:103` — IDs in the ABI are 1-based; 0 is reserved to signal "invalid" and `getCounterById(0)` returns empty.

## Configuration
- `dynamic_module_config.name`, `dynamic_module_config.do_not_close`, `dynamic_module_config.load_globally`, `dynamic_module_config.metrics_namespace` (defaults to `dynamicmodulescustom`).
- `logger_name` — opaque string handed to the module.
- `logger_config` — `google.protobuf.Any` converted to bytes by `knownAnyToBytes` (supports `StringValue`, `BytesValue`, `Struct`).

## Stats / errors
Stats are entirely module-defined; they live under `stats_scope_` created with `metrics_namespace` as the prefix, and the module adds counters/gauges/histograms via the ABI (`addCounter()` / `addGauge()` / `addHistogram()`).
Errors raised as `EnvoyException`:
- module load failure (`config.cc:28`),
- `knownAnyToBytes` failure for `logger_config` (`config.cc:37`),
- symbol resolution or in-module config creation failure in `newDynamicModuleAccessLogConfig()` (`config.cc:53`).
