# Wasm Bootstrap Extension

Bootstrap extension that starts a singleton or per-worker WasmService at
server start-up. It's the entry point for running standalone Wasm plugins
(not attached to an HTTP/TCP filter chain) inside Envoy.

Proto: `envoy.extensions.wasm.v3.WasmService`.
Factory name: `envoy.bootstrap.wasm`.

## Files
- `config.h/cc` - `WasmFactory` (`BootstrapExtensionFactory` registration)
  and `WasmServiceExtension` (the bootstrap extension itself).

## Interface
- Factory base: `Server::Configuration::BootstrapExtensionFactory`.
- Extension base: `Server::BootstrapExtension`.

## Logic
- `WasmFactory::createBootstrapExtension` validates the proto, registers
  `Extensions::Common::Wasm::CustomStatNamespace` as a custom stat
  namespace, then constructs a `WasmServiceExtension`.
- `WasmServiceExtension::onServerInitialized` calls `createWasm`, which
  builds an `Extensions::Common::Wasm::PluginConfig` using the server
  scope, init manager, traffic direction `UNSPECIFIED`, and the
  `singleton` flag from the proto config.
- `wasmService()` exposes the constructed `PluginConfig` for code that
  needs to schedule async calls into the Wasm VM.

## Key decision points
- `config.cc:31` - custom stat namespace registration must happen at
  bootstrap time so Prometheus output can strip the Wasm prefix.
- `config.cc:16` - `createWasm` is deferred to `onServerInitialized`
  rather than done in the factory so all server resources (scope,
  init manager, cluster manager) are available when the VM starts.
- Validation uses `staticValidationVisitor()` because the `WasmService`
  proto is part of static bootstrap config, not dynamic xDS.

## Configuration
- `config` - `envoy.extensions.wasm.v3.PluginConfig` describing the VM,
  code, and runtime to use.
- `singleton` - if true, the plugin runs on the main thread as a single
  instance; otherwise per-worker.

## Stats / errors
- Wasm runtime stats are emitted under
  `Extensions::Common::Wasm::CustomStatNamespace`. Load-time failures
  surface as `EnvoyException` from proto validation or
  `PluginConfig` construction.
