# Resource Monitors

> Documentation for Envoy's resource monitor extensions.
> Source lives in `source/extensions/resource_monitors/`:
> `common/`, `fixed_heap/`, `cpu_utilization/`, `cgroup_memory/`,
> `downstream_connections/`, `injected_resource/`.

These are the **concrete resources** the overload manager watches. The overload manager
(`source/server/overload_manager_impl.*`, documented in
[`../../server/overload/`](../../server/overload/README.md)) provides the *framework* — the
refresh loop, triggers, actions, and thread-local publication. This folder provides the actual
sensors: heap usage, CPU utilization, cgroup memory, the global downstream-connection count, and
a test/synthetic resource.

## Two kinds of monitor

| Kind | Interface | Model | Used for |
|------|-----------|-------|----------|
| **Regular** (polled) | `Server::ResourceMonitor` | Pull: overload manager periodically calls `updateResourceUsage(callbacks)`; monitor reports pressure via `onSuccess`/`onFailure` | Things you can sample on a timer (heap, CPU, memory, a file) |
| **Proactive** (accounting) | `Server::ProactiveResourceMonitor` | Push: data-path code calls `tryAllocateResource`/`tryDeallocateResource` synchronously and atomically | Hot-path limits that can't wait for a poll (global connection cap) |

Five of the six monitors are regular; only `downstream_connections` is proactive.

## The six monitors at a glance

| Monitor | Resource name | Kind | What it measures |
|---------|---------------|------|------------------|
| `fixed_heap` | `envoy.resource_monitors.fixed_heap` | Regular | TCMalloc heap usage / `max_heap_size_bytes` |
| `cpu_utilization` | `envoy.resource_monitors.cpu_utilization` | Regular | CPU busy fraction (host `/proc/stat` or cgroup), EWMA-smoothed |
| `cgroup_memory` | `envoy.resource_monitors.cgroup_memory` | Regular | cgroup memory usage / limit (v1 or v2) |
| `downstream_connections` | `envoy.resource_monitors.global_downstream_max_connections` | **Proactive** | Active downstream connections vs a global cap |
| `injected_resource` | `envoy.resource_monitors.injected_resource` | Regular | A pressure value read from a file (tests) |
| `common` | — | — | Shared factory base classes |

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | The two interface models in detail, the `FactoryBase`/`ProactiveFactoryBase` registration pattern, and how the overload manager consumes each kind. |
| `monitors.md` | Per-monitor deep dive: the exact pressure formula, stats-reader seams, config fields, and platform/cgroup handling for each of the five concrete monitors. |
| `CLASS_HIERARCHY.md` | UML diagrams for the interfaces, factory bases, and each monitor. |

## One-paragraph mental model

Each monitor is a small extension registered under `envoy.resource_monitors` via a `FactoryBase`
(or `ProactiveFactoryBase`) that downcasts and validates its typed config proto. Regular monitors
implement `updateResourceUsage()` to compute a pressure fraction (usually 0.0–1.0) and report it
through callbacks; the overload manager scales that against configured thresholds to drive
actions. The one proactive monitor instead exposes atomic allocate/deallocate calls that the
connection-accept path uses to enforce a global cap in real time. All the interesting per-monitor
logic is *how* pressure is computed — heap math, CPU deltas with EWMA smoothing, cgroup file
reads — which is the subject of `monitors.md`.
