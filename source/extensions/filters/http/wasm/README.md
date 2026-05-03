# Wasm (`envoy.filters.http.wasm`)

Thin Envoy wrapper that installs a Proxy-Wasm plugin as an HTTP filter. The heavy lifting — VM lifecycle, thread-local plugin handles, ABI callbacks — lives in `Extensions::Common::Wasm::PluginConfig`. This filter's job is just to construct that shared config, create a per-stream context via `createContext()`, and register the resulting object both as a stream filter and as an access-log handler so Wasm callbacks fire for every lifecycle phase.

Proto: `envoy.extensions.filters.http.wasm.v3.Wasm`.

## Files
- `config.h/cc` — `WasmFilterConfig` / `UpstreamWasmFilterConfig` factories registered for both downstream and upstream HTTP filter chains. Contains the templated `createFilterFactoryFromProtoTyped`.
- `wasm_filter.h/cc` — `FilterConfig` which derives directly from `Extensions::Common::Wasm::PluginConfig`; three ctors target `FactoryContext`, `UpstreamFactoryContext`, and `ServerFactoryContext`.

## Lifecycle
- Registered twice at `config.cc:14-15` — as `NamedHttpFilterConfigFactory` and as `UpstreamHttpFilterConfigFactory`. Both entries resolve to the same templated `createFilterFactoryFromProtoTyped` (`config.h:59-74`).
- The three public `createFilterFactoryFromProto*` overloads (`config.h:29-56`) downcast-and-validate the proto then delegate to the template with the matching `FactoryContext` type. The `ServerFactoryContext` overload returns a raw `FilterFactoryCb` (no `StatusOr`) because it is used in contexts where the filter must succeed.
- Inside the template (`config.h:60-74`):
  - `customStatNamespaces().registerStatNamespace("wasmcustom")` ensures Wasm-exported metrics are accepted by the stat registry (`config.h:63-64`).
  - Builds a `FilterConfig` shared pointer (one per listener/cluster, threaded through `PluginConfig` which keeps a TLS slot for VM handles).
  - Returned cb calls `filter_config->createContext()` on each new stream. If the VM failed to produce a context (`nullptr`), the filter fails open by simply not registering anything (`config.h:67-70`).
  - Otherwise `addStreamFilter(filter)` and `addAccessLogHandler(filter)` attach the same `Context` object (implements `StreamFilter` + `AccessLog::Instance`) so Proxy-Wasm ABI hooks get both stream events and `log()` on destroy.
- `FilterConfig` ctors (`wasm_filter.cc:8-24`): forward `config.config()` (the `PluginConfig` proto), the server factory context, stats scope, init manager, traffic direction (taken from `listener_info.direction()` downstream, hard-coded to `OUTBOUND` for upstream/server variants), and the listener metadata pointer (nullptr for upstream/server).
- Per-stream decode/encode callbacks are implemented by `Common::Wasm::Context` (defined in `source/extensions/common/wasm/`), not in this filter — this folder contains no custom header/data/trailer overrides.

## Decision / logic
- Fail-open on VM failure: `config.h:67-70` — when the plugin could not yield a context, the filter is silently omitted from the chain so the stream still proceeds.
- Traffic direction selection: `wasm_filter.cc:12` uses the listener direction; `wasm_filter.cc:18, 24` hard-code `OUTBOUND` for upstream and server-scoped configs.
- Listener-metadata plumbing: `wasm_filter.cc:12` passes `&listenerInfo().metadata()` for downstream; the other two ctors pass `nullptr` because no listener scope exists.
- `fail_open` semantics for ABI errors, VM restart policies, plugin matching, and all per-stream header/body handling live in `PluginConfig` / `Common::Wasm::Context`.

## Configuration
- `config` — `envoy.extensions.wasm.v3.PluginConfig`. Contains the VM config (runtime, code source, configuration, capability restrictions), `name`, `root_id`, `failure_policy`, and plugin configuration.
- Upstream variant is controlled by the registration factory, not proto fields; both variants share the same message type.
- No per-route config surface defined in this folder.

## Stats
No filter-level counters are defined here. Stats come from two places:
- Common Wasm scaffolding (`PluginConfig` / VM handles) — e.g. `envoy.wasm.*` fail-open / plugin-create counters.
- Wasm-exported stats in the `wasmcustom` namespace registered at `config.h:63-64`, emitted by plugin code via the Proxy-Wasm metric ABI.
