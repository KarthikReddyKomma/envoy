# Wasm Access Logger

Access-log sink that forwards each record into a Wasm plugin's `onLog` ABI callback, letting user-provided Wasm code (C++, Rust, AssemblyScript, etc.) read the stream's request/response context and emit logs, metrics, or side effects. Unlike the other sinks in this tree this logger does no formatting, buffering, or transport: it simply finds the active `Wasm` VM for the configured plugin on the current worker thread and delegates to `Wasm::log(plugin, log_context, stream_info)`, which runs the guest's `onLog` handler for the current stream.

Proto: `envoy.extensions.access_loggers.wasm.v3.WasmAccessLog` (see `api/envoy/extensions/access_loggers/wasm/v3/wasm.proto`). Factory name: `envoy.access_loggers.wasm` (legacy alias `envoy.wasm_access_log`) - config.cc:41, 46.

## Files
- `wasm_access_log_impl.h` - `WasmAccessLog` (header-only `AccessLog::Instance`). Holds the `PluginConfigPtr` and an optional pre-built `AccessLog::FilterPtr`. `log(...)` evaluates the filter, then calls `wasm->log(plugin, log_context, stream_info)` if the VM is live.
- `config.h` / `config.cc` - `WasmAccessLogFactory` (AccessLogInstanceFactory) that builds a `Common::Wasm::PluginConfig` from the proto and registers the Wasm custom-stat namespace.

The Wasm VM plumbing, lifecycle (VM create/fail-open/fail-closed), thread-local plugin handle, and `onLog` dispatch all live under `source/extensions/common/wasm/`.

## Interface
- `AccessLog::Instance::log(const Formatter::Context&, const StreamInfo::StreamInfo&) override` (wasm_access_log_impl.h:22). No `Common::ImplBase` - the class implements `Instance` directly because the filter evaluation is inlined.

## Flow
1. Config load (config.cc:17): downcast the proto, build a `Common::Wasm::PluginConfig` with the server factory context, scope, and init manager. `TrafficDirection::UNSPECIFIED` is passed because access logs do not have an inherent direction. The logger is wrapped as a `shared_ptr<WasmAccessLog>` and returned.
2. VM lifecycle: `PluginConfig` owns the Wasm plugin artifact and coordinates VM creation across workers via a thread-local plugin handle. `plugin_config_->wasm()` returns the active VM for the current thread (or `nullptr` if the VM is unavailable, e.g. still loading or failed open).
3. Per record (wasm_access_log_impl.h:22):
   - If a filter was attached, call `filter_->evaluate(log_context, stream_info)`; return early if false.
   - Fetch `plugin_config_->wasm()`; if null, silently drop (no-op).
   - Call `wasm->log(plugin, log_context, stream_info)` which crosses into the guest's `onLog` ABI with serialized access to request/response headers, trailers, stream info, and filter state through the proxy-wasm host functions.

## Key decision points
- `wasm_access_log_impl.h:30` - null-check on `wasm()` means a failed-open VM silently skips logging rather than crashing the worker.
- `wasm_access_log_impl.h:24` - filter is evaluated **before** the VM call so Wasm code is never invoked for suppressed records.
- `config.cc:28` - `fail_open=false` (last arg) is the default plugin-creation behavior; VM errors surface through the init manager.
- `config.cc:31` - registers `Extensions::Common::Wasm::CustomStatNamespace` so guest-defined stats appear under the Wasm-owned namespace rather than colliding with first-party stat names.
- `config.cc:46` - `LEGACY_REGISTER_FACTORY` keeps the `envoy.wasm_access_log` alias.

## Configuration
```yaml
access_log:
- name: envoy.access_loggers.wasm
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.wasm.v3.WasmAccessLog
    config:
      name: my_plugin
      root_id: my_root_id
      vm_config:
        runtime: envoy.wasm.runtime.v8
        code:
          local: { filename: /etc/envoy/access_log_plugin.wasm }
      configuration:
        "@type": type.googleapis.com/google.protobuf.StringValue
        value: '{"level":"info"}'
```

## Stats / errors
- No stats are emitted by the extension itself. Guest stats are registered in the `CustomStatNamespace` (config.cc:31-32); any `ENVOY_LOG` output comes from the VM runtime.
- VM load / validation failures surface through the init manager during warmup and will block the listener's ready state unless the plugin is configured to fail open.
- If the VM becomes unavailable at runtime (crash, recreate-in-progress), `log()` silently drops records - there is no retry or buffering.
