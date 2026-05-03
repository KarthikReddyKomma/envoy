# Dynamic Modules (`envoy.load_balancing_policies.dynamic_modules`)

A load-balancing policy whose host-selection logic is implemented by an
externally-loaded dynamic module (Envoy's ABI-stable plugin system). The C++
code here is the thin Envoy-side shim: it loads the module by name, hands it a
protobuf config, and forwards every `chooseHost` call to the module via the
`envoy_dynamic_module_on_lb_choose_host` ABI. Use this when your LB algorithm
is shipped as a compiled artifact outside the Envoy binary.

Proto: `envoy.extensions.load_balancing_policies.dynamic_modules.v3.DynamicModulesLoadBalancerConfig`.

## Files
- `config.h/cc` — `Factory` registered as
  `envoy.load_balancing_policies.dynamic_modules`. `loadConfig` loads the
  module via `DynamicModules::newDynamicModuleByName`, packages up the bytes
  of the nested `lb_policy_config` Any, calls `DynamicModuleLbConfig::create`
  and returns a `TypedDynamicModuleLbConfig`. `create` wraps an internal
  `LbFactory` in a simple `ThreadAwareLb`.
- `lb_config.h/cc` — `DynamicModuleLbConfig` (module-side LB config wrapper),
  the ABI function pointers (`on_lb_new_`, `on_lb_destroy_`,
  `on_host_membership_update_`, `on_choose_host_`), and the metrics namespace
  registration.
- `load_balancer.h/cc` — `DynamicModuleLoadBalancer`, the per-worker LB. It
  creates the in-module LB instance in its constructor, registers for host
  membership updates, and implements `chooseHost` by delegating to the module.
- `abi_impl.cc` — implementations of the ABI callbacks exposed back to the
  module: per-host data storage (`setHostData` / `getHostData`), priority-set
  queries, and filter/stream-info accessors.

## Load balancer class
`DynamicModuleLoadBalancer` implements `Upstream::LoadBalancer` directly. It is
not zone-aware and does not extend `EdfLoadBalancerBase`; the module is
responsible for whatever algorithm it wants.

## Algorithm
Entirely delegated to the dynamic module. On every request
`DynamicModuleLoadBalancer::chooseHost` calls `config_->on_choose_host_`,
passing the `LoadBalancerContext*` opaquely and receiving back a
`(priority, host_index)` pair. The shim looks up
`priority_set_.hostSetsPerPriority()[priority]->healthyHosts()[host_index]` and
returns it. Out-of-range returns from the module are logged and become a null
host (`load_balancer.cc:54-69`).

Host membership changes are delivered into the module through
`on_host_membership_update_`. Envoy parks the `hosts_added`/`hosts_removed`
vectors on the `DynamicModuleLoadBalancer` so that the module can query them
through the ABI during the callback (`load_balancer.cc:21-29`).

## Key decision points
- Module load and LB config creation: `config.cc:64-109`.
- Thread-aware create path: `config.cc:52-62`.
- In-module LB instantiation and member-update registration:
  `load_balancer.cc:8-30`.
- Per-request delegation: `load_balancer.cc:39-70`.
- Per-host opaque data storage:
  `load_balancer.cc:89-123` (used by modules that need to stash state keyed by
  host).

## Configuration
Proto fields consumed in `Factory::loadConfig` (`config.cc:64-109`):
- `dynamic_module_config.name` — module name passed to
  `DynamicModules::newDynamicModuleByName`.
- `dynamic_module_config.do_not_close` — keep the `.so` resident even after
  the config is destroyed.
- `dynamic_module_config.load_globally` — RTLD_GLOBAL.
- `dynamic_module_config.metrics_namespace` — namespace for module-exposed
  stats (defaults to `DefaultMetricsNamespace`).
- `lb_policy_name` + `lb_policy_config` — identifier and typed-Any config for
  the in-module LB implementation.

## Stats
The module creates stats under the configured `metrics_namespace`. When the
runtime feature `envoy.reloadable_features.dynamic_modules_strip_custom_stat_prefix`
is on, the namespace is registered as a custom stat namespace so the prefix is
stripped from Prometheus output (`config.cc:100-106`). No stats are added by
the shim itself.
