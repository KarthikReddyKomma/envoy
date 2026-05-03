# Wasm stats sink

Delegates the entire stats flush path to a user-supplied WebAssembly
plugin. The sink holds a `Common::Wasm::PluginConfig` and, on each
flush, hands the raw `Stats::MetricSnapshot` to the plugin's
`onStatsUpdate` ABI hook. The Wasm guest decides how to serialise,
filter, or forward the metrics. Registered as `envoy.stat_sinks.wasm`.

Proto: `envoy.extensions.stat_sinks.wasm.v3.Wasm`.

## Files
- `config.h` — `WasmSinkFactory` declaration, `WasmName` constant
  (`config.h:18`).
- `config.cc` — proto validation, `PluginConfig` construction, custom
  stat namespace registration, factory registration.
- `wasm_stat_sink_impl.h` — `WasmStatSink`, the tiny `Stats::Sink` that
  forwards flushes to the Wasm VM.
- `BUILD` — extension registration, depends on
  `//source/extensions/common/wasm:wasm_lib`.

## Interface
- `Stats::Sink::flush(MetricSnapshot&)` —
  `wasm_stat_sink_impl.h:23`. Resolves the plugin's current `Wasm*`
  (may be null while a VM is reloading) and invokes
  `wasm->onStatsUpdate(plugin_config_->plugin(), snapshot)`.
- `Stats::Sink::onHistogramComplete(Histogram&, uint64_t)` — intentional
  no-op (`wasm_stat_sink_impl.h:29`). Per-histogram sample callbacks are
  not forwarded; the plugin only sees aggregated snapshot data at flush
  time.
- Factory:
  - `WasmSinkFactory::createStatsSink()` (`config.cc:17`).
  - `createEmptyConfigProto()` returns an empty `Wasm` proto.
  - `name()` returns `envoy.stat_sinks.wasm`.

## Flow
1. Factory downcasts + validates the proto
   (`config.cc:20-22`).
2. Constructs a `Common::Wasm::PluginConfig` with
   `TrafficDirection::UNSPECIFIED`, the server scope, the init manager,
   and `singleton=true` (`config.cc:24-26`) — this triggers VM load and
   start.
3. Registers `Common::Wasm::CustomStatNamespace` with the API's
   `customStatNamespaces()` (`config.cc:28-29`). This allows stats
   created by the Wasm plugin itself to be tagged and passed through
   without colliding with Envoy's stat namespace.
4. Wraps the `PluginConfig` in a `WasmStatSink` and returns it
   (`config.cc:31`).
5. At runtime, the server stats flush timer calls `flush(snapshot)`.
   `plugin_config_->wasm()` returns the live `Wasm*` (or null during a
   reload / after a crash-induced detach); when non-null,
   `onStatsUpdate()` is invoked. The Wasm ABI exposes the snapshot to
   the guest via proxy-wasm host calls; serialisation is the guest's
   responsibility.

## Key decision points
- The sink is intentionally a thin trampoline; there is no serialisation
  or buffering in native code
  (`wasm_stat_sink_impl.h:23-27`). All policy lives in the Wasm guest.
- `plugin_config_->wasm()` is consulted every flush
  (`wasm_stat_sink_impl.h:24`) rather than cached, so VM reloads are
  picked up automatically.
- Histograms are dropped at `onHistogramComplete` because the per-sample
  path does not fit the "push whole snapshot to the guest" model; the
  guest sees histograms only via `snapshot.histograms()` in `flush()`.
- `CustomStatNamespace` registration happens once at factory time
  (`config.cc:28`). It is a process-wide registration, so it is safe
  even if multiple Wasm sinks are configured.
- No `absl::StatusOr` error handling around `PluginConfig` construction
  — load errors are surfaced by the Wasm framework asynchronously via
  `initManager`.

## Configuration

`envoy.extensions.stat_sinks.wasm.v3.Wasm`:

- `config` — standard Wasm plugin config (`PluginConfig` proto):
  VM config (runtime, code source), plugin name, root_id,
  configuration string, fail-open/fail-close, etc.

The custom stat namespace used by plugins that emit their own metrics is
`Common::Wasm::CustomStatNamespace`; plugins create stats in this
namespace via proxy-wasm host calls and Envoy surfaces them with the
appropriate prefix tag.

## Stats / errors
- No sink-local Envoy counters.
- Wasm VM errors (load failure, trap during `onStatsUpdate`) are handled
  by the Wasm framework: the guard `wasm != nullptr`
  (`wasm_stat_sink_impl.h:24`) skips flushes when the VM is unavailable,
  so the server never crashes on a failed guest.
- All plugin-facing error reporting happens inside the guest and/or
  through the Wasm framework's standard failure stats, not here.
