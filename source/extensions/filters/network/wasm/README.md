# Wasm Network Filter (`envoy.filters.network.wasm`)

Thin bridge that instantiates a proxy-wasm plugin as a network filter. The actual L4 filter (implementing `Network::ReadFilter`/`WriteFilter` via the proxy-wasm `Context`) is produced by the shared Wasm plugin infrastructure under `source/extensions/common/wasm/`; this folder only provides the config plumbing. Lifecycle callbacks (`onNewConnection`, `onData`, `onWrite`, connection close) are delivered to the Wasm VM which dispatches them to the configured plugin.

Proto: `envoy.extensions.filters.network.wasm.v3.Wasm`.

## Files
- `config.h/cc` — `WasmFilterConfig` factory registered as `envoy.filters.network.wasm` (`config.h:20`). Registers the custom stat namespace used by Wasm plugins, builds a `FilterConfig`, and installs the filter (`config.cc:16`).
- `wasm_filter.h/cc` — `FilterConfig` (`wasm_filter.h:19`), a thin subclass of `Extensions::Common::Wasm::PluginConfig`. It forwards the proto `config()` message along with the server factory context, scope, init manager, listener direction, and listener metadata to the common plugin-config base (`wasm_filter.cc:8`). All filter semantics live in that base and in the proxy-wasm runtime.

## Lifecycle
The `Network::Filter` returned by `FilterConfig::createContext()` is a proxy-wasm `Context` object from `source/extensions/common/wasm/`. The standard L4 hooks map directly onto proxy-wasm ABI calls:
- `onNewConnection()` → proxy-wasm `proxy_on_new_connection`.
- `onData(data, end_stream)` → `proxy_on_downstream_data` (bytes stay in the Envoy buffer; the VM reports how many to pass/block).
- `onWrite(data, end_stream)` → `proxy_on_upstream_data`.
- Connection close events → `proxy_on_downstream_connection_close` / `proxy_on_upstream_connection_close`.

If the plugin cannot be created (e.g., Wasm VM failure with `fail_open`), `createContext()` returns `nullptr` and the filter factory silently skips `addFilter` — "fail open" (`config.cc:22`).

## Decision / logic
- Factory VM fetch: `PluginConfig` (constructed at `wasm_filter.cc:10`) resolves or fetches the VM described by the `VmConfig`, loading bytecode either inline or asynchronously via remote data source — hence `FactoryContext::initManager()` is passed through so the listener warms up before traffic hits it.
- Direction-aware context: the listener direction (`context.listenerInfo().direction()`) is passed to `PluginConfig` so the Wasm plugin can identify inbound vs. outbound traffic (`wasm_filter.cc:11`).
- Per-connection context: `filter_config->createContext()` (`config.cc:23`) is called per connection; each connection gets its own root/stream context pair.

## Configuration
- `config` — an `envoy.extensions.wasm.v3.PluginConfig` that carries the `VmConfig` (runtime: v8/wasmtime/null, module bytecode or HTTP/data-source fetch), `name`, `root_id`, `configuration`, `fail_open`, and capability restrictions. Forwarded verbatim to `PluginConfig` (`wasm_filter.cc:10`). Plugin-specific behavior, buffer handling, and any stats beyond the common namespace are defined by the module itself.

## Stats
No counters are emitted directly from this C++ code. The factory registers the Wasm custom stat namespace on the API so plugins can create stats at runtime:
```
context.serverFactoryContext().api().customStatNamespaces()
    .registerStatNamespace(Extensions::Common::Wasm::CustomStatNamespace); // config.cc:19
```
Any user-facing counters/gauges/histograms are created by the Wasm module via proxy-wasm host calls and appear under that namespace.

## Factory
`WasmFilterConfig::createFilterFactoryFromProtoTyped` (`config.cc:16`) registers the custom stat namespace, builds `FilterConfig`, and returns a lambda that, per connection, pulls a new context from the plugin config and adds it as a bidirectional `Network::Filter` (`config.cc:22`). Registered via `REGISTER_FACTORY` at `config.cc:33`.
