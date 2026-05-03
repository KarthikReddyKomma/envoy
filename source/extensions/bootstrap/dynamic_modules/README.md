# Dynamic Modules Bootstrap Extension

Hosts an out-of-tree bootstrap extension implemented as a C ABI dynamic
module (`.so`). The bootstrap factory loads the module, resolves the ABI
entry points, and creates a per-module `DynamicModuleBootstrapExtension`
that is driven through Envoy's server lifecycle (`onServerInitialized`,
`onWorkerThreadInitialized`, drain, shutdown).

Proto:
`envoy.extensions.bootstrap.dynamic_modules.v3.DynamicModuleBootstrapExtension`.
Factory name: `envoy.bootstrap.dynamic_modules`.

## Files
- `factory.h/cc` - `DynamicModuleBootstrapExtensionFactory`
  (`BootstrapExtensionFactory`) that loads the module and builds the
  `DynamicModuleBootstrapExtensionConfig`.
- `extension.h/cc` - `DynamicModuleBootstrapExtension`: owns the in-module
  extension pointer; forwards Envoy lifecycle events into ABI callbacks.
- `extension_config.h` - `DynamicModuleBootstrapExtensionConfig` plus
  scheduler/timer helper classes, metrics handle types
  (`ModuleCounterHandle`, `ModuleGaugeHandle`, `ModuleHistogramHandle` and
  their Vec variants), and the `newDynamicModuleBootstrapExtensionConfig`
  factory.
- `abi_impl.cc` - implementations of the `envoy_dynamic_module_callback_*`
  C ABI functions exposed to the loaded module.

## Interface
- Base: `Server::BootstrapExtension`.
- Factory base: `Server::Configuration::BootstrapExtensionFactory`.

## Logic
- `factory.cc:25` loads the `.so` via
  `Extensions::DynamicModules::newDynamicModuleByName` using
  `do_not_close` and `load_globally` from the proto config.
- The extension config is the seam between Envoy and the module: it
  owns the `DynamicModulePtr`, resolves every ABI function pointer
  (`on_bootstrap_extension_*`), provides async HTTP callouts via
  `sendHttpCallout`, manages file watchers
  (`envoy_dynamic_module_callback_bootstrap_extension_file_watcher_add_watch`),
  and exposes metric factories.
- Optional cluster / listener lifecycle subscriptions are enabled
  post-startup (`enableClusterLifecycle`, `enableListenerLifecycle`) since
  the `ClusterManager` and `ListenerManager` are not available during
  `BootstrapExtensionFactory::createBootstrapExtension`.
- `DynamicModuleBootstrapExtension::registerLifecycleCallbacks`
  (`extension.cc:43`) hooks the drain manager and
  `ServerLifecycleNotifier::Stage::ShutdownExit`. Shutdown forwards a
  completion callback to the module via a heap-allocated
  `Event::PostCb` passed through the C ABI as an opaque void\*.
- `DynamicModuleBootstrapExtensionConfigScheduler` and
  `DynamicModuleBootstrapExtensionTimer` wrap main-thread posting so
  module code on other threads can schedule event hooks.

## Key decision points
- `factory.cc:60` - a runtime guard
  (`envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix`)
  controls whether the module's metrics namespace is registered as a
  custom stat namespace. Legacy namespace stripping is the default
  when the guard is on.
- `factory.cc:42` - `metrics_namespace` falls back to
  `DefaultMetricsNamespace = "dynamicmodulescustom"` when unset.
- `extension.cc:44` - drain callbacks use
  `Network::DrainDirection::All` so both inbound and outbound drains
  notify the module.
- `extension_config.h` uses `std::enable_shared_from_this` so HTTP
  callouts and timers can safely hold weak refs back to the config.

## Configuration
- `dynamic_module_config.name` / `do_not_close` / `load_globally` -
  module load flags.
- `dynamic_module_config.metrics_namespace` - stat namespace override
  (default `dynamicmodulescustom`).
- `extension_name` - identifies the in-module bootstrap extension to
  instantiate.
- `extension_config` - opaque `google.protobuf.Any` passed to
  `on_bootstrap_extension_new`.

## Stats / errors
- Module-defined counters, gauges, histograms and their labeled
  (`Vec`) variants; scoped under the configured metrics namespace.
- Load-time failures (bad module, ABI mismatch, bad `Any` config)
  throw `EnvoyException` from `factory.cc:27/35/52`.
