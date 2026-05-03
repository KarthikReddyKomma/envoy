# Dynamic Modules Cluster (`envoy.clusters.dynamic_modules`)

An ABI-driven cluster plug-in: the host set and the load-balancer decision are owned by a user-supplied dynamic library (Rust / Go / C) linked via the Envoy dynamic-modules ABI. Envoy itself provides the plumbing: it loads the `.so`, resolves a fixed set of C symbols (`envoy_dynamic_module_on_cluster_*`), and calls them in response to cluster / server / LB lifecycle events. The module adds/removes hosts through callback functions and is consulted on every `chooseHost()`.

Proto: `envoy.extensions.clusters.dynamic_modules.v3.ClusterConfig` (see `api/envoy/extensions/clusters/dynamic_modules/v3/cluster.proto`).

## Files
- `cluster.h` / `cluster.cc` - `DynamicModuleClusterConfig`, `DynamicModuleClusterHandle`, `DynamicModuleCluster`, `DynamicModuleClusterScheduler`, `DynamicModuleAsyncHostSelectionHandle`, `DynamicModuleLoadBalancer`, `DynamicModuleThreadAwareLoadBalancer`, `DynamicModuleClusterFactory`.
- `abi_impl.cc` - Implementations of the C ABI callback entry points exported to the module (`envoy_dynamic_module_callback_*`): host add/remove, metric registration, HTTP callouts, async host selection completion, scheduler create/delete.

## Cluster type
- `DynamicModuleCluster` extends `Upstream::ClusterImplBase` and `std::enable_shared_from_this<DynamicModuleCluster>` (`cluster.h:295`). It manages hosts directly in `priority_set_` instead of using a `PriorityStateManager`, because hosts arrive from the module and are tracked by raw pointer for O(1) round-trips through the ABI.
- `initializePhase()` returns `Primary` (`cluster.h:300`).

## Host plumbing
- `DynamicModuleClusterConfig::create()` (`cluster.cc:52-110`) `dlsym`s the required entry points via `DynamicModule::getFunctionPointer<...>` (listed at `cluster.h:41-55`). Required: `on_cluster_config_new/destroy`, `on_cluster_new/init/destroy`, `on_cluster_lb_new/destroy/choose_host`. Optional: async cancel, `on_cluster_scheduled`, lifecycle hooks (`server_initialized`, `drain_started`, `shutdown`), HTTP callout completion, LB membership-update hook.
- After symbol resolution it calls `on_cluster_config_new(cfg, name_buf, config_buf)` (`cluster.cc:104`) to build the module-side config object (`in_module_config_`).
- `DynamicModuleCluster` ctor (`cluster.cc:153-167`) calls `on_cluster_new(in_module_config_, this)` to allocate the in-module cluster instance (`in_module_cluster_`), seeds priority 0 with an empty host set, then `registerLifecycleCallbacks()` installs optional server_initialized / drain / shutdown notifications (`cluster.cc:184-222`).
- `startPreInit()` delegates to the module (`cluster.cc:224-228`): the module must eventually call `envoy_dynamic_module_callback_cluster_pre_init_complete`, which routes back to `preInitComplete()` -> `onPreInitComplete()`.
- `addHosts(...)` (`cluster.cc:253+`) is invoked by the module via an ABI callback. It builds `HostImpl` objects (with locality, metadata, weight, per-host data), registers them in a raw-pointer -> `HostSharedPtr` map under `host_map_lock_`, then grows the targeted priority's host set directly.
- `removeHosts(...)` and `updateHostHealth(...)` provide symmetric operations.
- `findHost(raw_host_ptr)` / `findHostByAddress(...)` are the reverse lookups used inside the ABI to translate module-side pointers back to `HostSharedPtr` (`cluster.h:307-308`).

## Lifetime and thread-safety
- `DynamicModuleClusterHandle` (`cluster.h:275-287`) owns the `shared_ptr<DynamicModuleCluster>` and guarantees that the destructor runs on the main thread. If the handle is dropped on a worker, its destructor posts the teardown onto the main dispatcher (`cluster.cc:125-147`). This is necessary because `shutdown_handle_` / `drain_handle_` / `server_initialized_handle_` unregister from main-thread-owned notifiers.
- `DynamicModuleClusterScheduler` (`cluster.h:399-424`) lets the module post events from any thread; `commit(event_id)` captures a `weak_ptr` and posts through `dispatcher_`, so a stale scheduler no-ops safely if the cluster is gone.

## Load balancing hooks
- `DynamicModuleThreadAwareLoadBalancer::factory()` creates one `DynamicModuleLoadBalancer` per worker using the worker-local `priority_set` (`cluster.cc:28-44`).
- `DynamicModuleLoadBalancer` ctor (`cluster.h:456`) subscribes to `priority_set_.addMemberUpdateCb(...)` so the module's `on_cluster_lb_on_host_membership_update` hook fires on the worker thread when hosts change. During the callback, `hosts_added_` / `hosts_removed_` are exposed to the ABI via `hostsAdded()` / `hostsRemoved()`.
- `chooseHost(context)` forwards into `on_cluster_lb_choose_host(...)`. The module can return three outcomes:
  - *Sync success* - a raw host pointer that is resolved through `DynamicModuleCluster::findHost`.
  - *Sync failure* - null host response.
  - *AsyncPending* - an `envoy_dynamic_module_type_cluster_lb_async_handle_module_ptr`; the LB builds a `DynamicModuleAsyncHostSelectionHandle` (`cluster.h:432-446`) and captures `active_async_dispatcher_` + `active_async_cancelled_` so that the module can complete (or the router can cancel) from any thread.
- `withActiveInstance(lb, f)` (`cluster.h:502`) is a process-wide registry lookup used by the async-completion ABI to safely call back into an LB that may have been destroyed.
- Per-host custom data is stored per worker in `per_host_data_` keyed by `(priority, index)` (`cluster.h:518`), accessed through `setHostData`/`getHostData`.

## Key decision points
- Required vs optional symbol resolution - `cluster.cc:65-98`.
- Teardown-on-main-thread guard for lifecycle handles - `cluster.cc:125-147`.
- Validation of host weights (1..128) and addresses in `addHosts` - `cluster.cc:266-275`.
- Locality construction from per-host region/zone/sub_zone - `cluster.cc:277-286`.
- HTTP callouts: `sendHttpCallout` allocates a `HttpCalloutCallback` tracked in `http_callouts_` keyed by monotonically increasing `next_callout_id_`, cancelled en masse in the dtor (`cluster.cc:170-182`).

## Configuration
- `dynamic_module_config` (which `.so` to load, version checks).
- `cluster_name` / `cluster_config` opaque bytes forwarded verbatim to `on_cluster_config_new`.
- The top-level `Cluster` still controls `connect_timeout`, `circuit_breakers`, transport sockets, etc.

## Stats
- Standard `ClusterImplBase` stats.
- Custom metrics the module registers via `DynamicModuleClusterConfig::addCounter/addGauge/addHistogram` (and their `Vec` variants). They live in the `dynamicmodulescustom.` sub-scope created in `DynamicModuleClusterConfig::DynamicModuleClusterConfig` (`cluster.cc:112-113`). Handles are 1-based IDs in the ABI, converted internally with `ID_TO_INDEX` (`cluster.h:181`).
