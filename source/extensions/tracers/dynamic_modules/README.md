# Dynamic modules tracer (`envoy.tracers.dynamic_modules`)

Tracer that delegates all span semantics to an out-of-process-compiled
shared object loaded via Envoy's dynamic modules ABI. Envoy only owns the
`Tracing::Driver` / `Tracing::Span` interface wiring and the stats plumbing;
everything about how spans are represented, propagated, and exported is
implemented inside the module. Useful for writing a tracer in any language
that can expose the C ABI defined in `source/extensions/dynamic_modules/abi`.

Proto: `envoy.extensions.tracers.dynamic_modules.v3.DynamicModuleTracer`.

## Files
- `config.h/cc` - `DynamicModuleTracerFactory` registered with
  `REGISTER_FACTORY(..., TracerFactory)`. Loads the shared object via
  `Extensions::DynamicModules::newDynamicModuleByName`, parses the optional
  `tracer_config` Any, builds the per-module `DynamicModuleTracerConfig`,
  and returns a `DynamicModuleDriver`.
- `tracer_config.h/cc` - `DynamicModuleTracerConfig` resolves all ABI
  function symbols from the module (all `envoy_dynamic_module_on_tracer_*`
  entry points) and owns the per-module stat handles (counters, gauges,
  histograms and their *_vec variants). `DynamicModuleDriver::startSpan` and
  `DynamicModuleSpan` forward every `Tracing::Span` call into the module
  through those resolved function pointers.
- `abi_impl.cc` - C-linkage ABI callbacks the module calls back into Envoy
  (e.g. read/write trace context headers, register and update metrics,
  decode the baggage API). Symbols exported here are the counterparts of
  the `on_tracer_*` functions implemented by the module.

## Tracer role
- `Tracing::Driver::startSpan(...)` - `DynamicModuleDriver::startSpan` calls
  `on_start_span_` with the current `Tracing::TraceContext*`.
- `Span::injectContext` - `DynamicModuleSpan::injectContext` updates
  `trace_context_` and calls `on_span_inject_context_`; the module writes
  the propagation headers it owns.
- `Span::finishSpan` - `on_span_finish_`, then `on_span_destroy_` on
  destruction.
- Additional Span methods (`setTag`, `log`, `spawnChild`, `getTraceId`,
  `getSpanId`, baggage get/set, sampled, local-decision) are all forwarded
  through the resolved ABI function pointers.

## Flow
- Span creation: driver hands the module the active `TraceContext*`; the
  module parses whatever propagation format it wants (B3, W3C, custom) via
  `abi_impl.cc` header accessors.
- Context propagation: `injectContext` is a one-shot forward call; the
  module mutates the outgoing headers directly.
- Batching / export: entirely the module's responsibility. Envoy has no
  buffer, timer, or exporter here.

## Key decision points
- `config.cc:22` - load-mode decisions (`do_not_close`, `load_globally`)
  flow from the proto to the loader.
- `tracer_config.h:39` - `DefaultMetricsNamespace = "dynamicmodulescustom"`
  prefix used when the proto omits `metrics_namespace`.
- `config.cc:53` - `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix`
  governs whether the custom stat prefix is stripped on registration.
- `tracer_config.h:269` - `newDynamicModuleTracerConfig` returns a
  `StatusOr`; symbol-resolution failures become `EnvoyException` in
  `config.cc`.

## Configuration
- `dynamic_module_config` - which shared object to load, plus load flags
  (`do_not_close`, `load_globally`) and the stat `metrics_namespace`.
- `tracer_name` - opaque name handed to the module as
  `envoy_dynamic_module_on_tracer_config_new`.
- `tracer_config` - opaque `Any` whose bytes are forwarded verbatim to the
  module.

## Stats / errors
- Counters, gauges, histograms, and *_vec variants registered at runtime by
  the module via `abi_impl.cc`. All live under
  `<metrics_namespace>.*` (`dynamicmodulescustom.*` by default).
- Symbol resolution, module load failures, and tracer-config creation
  failures are thrown as `EnvoyException` from
  `DynamicModuleTracerFactory::createTracerDriver` (`config.cc:25`,
  `config.cc:33`, `config.cc:48`).
