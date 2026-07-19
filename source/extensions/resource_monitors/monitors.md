# Resource Monitors — Per-Monitor Deep Dive

The exact pressure computation, config, and platform handling for each concrete monitor. All
monitors share the `FactoryBase` registration pattern from `OVERVIEW.md` §3.

---

## fixed_heap

| | |
|--|--|
| Config | `envoy.extensions.resource_monitors.fixed_heap.v3.FixedHeapConfig` |
| Resource name | `envoy.resource_monitors.fixed_heap` |
| Class | `FixedHeapMonitor` (`fixed_heap/fixed_heap_monitor.h`) |
| Kind | Regular |

Reports heap pressure as `used / max_heap`. It reads TCMalloc counters through a
`MemoryStatsReader` seam (virtual, so tests can mock it) wrapping `Memory::Stats`:

```cpp
uint64_t reservedHeapBytes()    { return Memory::Stats::totalCurrentlyReserved(); }
uint64_t unmappedHeapBytes()    { return Memory::Stats::totalPageHeapUnmapped(); }
uint64_t freeMappedHeapBytes()  { return Memory::Stats::totalPageHeapFree(); }
uint64_t allocatedHeapBytes()   { return Memory::Stats::totalCurrentlyAllocated(); }
```

The "used" figure has two definitions, gated by a runtime flag:

```cpp
size_t used = 0;
if (Runtime::runtimeFeatureEnabled("envoy.reloadable_features.fixed_heap_use_allocated")) {
  used = stats_->allocatedHeapBytes();
} else {
  const size_t physical    = stats_->reservedHeapBytes();
  const size_t unmapped    = stats_->unmappedHeapBytes();
  const size_t free_mapped = stats_->freeMappedHeapBytes();
  used = physical - unmapped - free_mapped;   // physically-backed, in-use heap
}
usage.resource_pressure_ = used / double(max_heap);
```

- **Legacy/default:** `pressure = (reserved − unmapped − free_mapped) / max_heap`.
- **With `fixed_heap_use_allocated`:** `pressure = currentlyAllocated / max_heap`.

`max_heap` is an `absl::variant<uint64_t, Runtime::UInt64>` — either the static
`max_heap_size_bytes` or a runtime-keyed value, resolved per poll. The factory enforces exactly
one is set and non-zero; a zero max at update time triggers `onFailure`.

---

## cpu_utilization

| | |
|--|--|
| Config | `envoy.extensions.resource_monitors.cpu_utilization.v3.CpuUtilizationConfig` |
| Resource name | `envoy.resource_monitors.cpu_utilization` |
| Class | `CpuUtilizationMonitor` (`cpu_utilization/cpu_utilization_monitor.h`) |
| Kind | Regular |

Reports the CPU busy fraction, smoothed. The stats source is a `CpuStatsReader`
(`virtual absl::StatusOr<double> getUtilization()`), selected by `config.mode()`:

| Mode | Reader | Source |
|------|--------|--------|
| `HOST` (default) | `LinuxCpuStatsReader` | `/proc/stat` |
| `CONTAINER` | `LinuxContainerCpuStatsReader::create(...)` → `CgroupV1CpuStatsReader` or `CgroupV2CpuStatsReader` (auto-detected) | cgroup files |

cgroup paths (centralized in `cpu_paths.h`):
- **v1:** `/sys/fs/cgroup/cpu/cpu.shares` + `/sys/fs/cgroup/cpuacct/cpuacct.usage`
- **v2:** `/sys/fs/cgroup/cpu.stat` (`usage_usec`) + `/sys/fs/cgroup/cpu.max` + `/sys/fs/cgroup/cpuset.cpus.effective`

### Delta-over-poll

Each reader keeps `previous_cpu_times_` and returns `0.0` on the first call (to establish a
baseline). The host reader parses `work = user + nice + system` and `total = work + idle`, then:

```cpp
const double work_over_period  = current.work_time  - previous_.work_time;
const int64_t total_over_period = current.total_time - previous_.total_time;
if (work_over_period < 0 || total_over_period <= 0) return InvalidArgumentError(...);
const double utilization = work_over_period / total_over_period;
previous_ = current;
```

cgroup v1 normalizes nanosecond usage by allocated millicores (`work = usage_ns * 1000 /
cpu.shares`) with monotonic wall-clock as `total`; cgroup v2 additionally divides by
`effective_cores` (from `cpu.max` quota/period clamped to the `cpuset.cpus.effective` count) and
clamps to `[0,1]`.

### EWMA smoothing

The monitor never reports the raw value — it smooths with an exponentially-weighted moving
average (history-heavy):

```cpp
constexpr double DAMPENING_ALPHA = 0.05;
utilization_ = current_utilization * DAMPENING_ALPHA + (1 - DAMPENING_ALPHA) * utilization_;
usage.resource_pressure_ = utilization_;
```

A failed read propagates via `onFailure`.

---

## cgroup_memory

| | |
|--|--|
| Config | `envoy.extensions.resource_monitors.cgroup_memory.v3.CgroupMemoryConfig` |
| Resource name | `envoy.resource_monitors.cgroup_memory` |
| Class | `CgroupMemoryMonitor` (`cgroup_memory/cgroup_memory_monitor.h`) |
| Kind | Regular |

Reports cgroup memory pressure as `usage / limit`. The `CgroupMemoryStatsReader` base auto-selects
v2 (preferred) or v1:

| Version | Usage file | Limit file |
|---------|-----------|-----------|
| v2 | `/sys/fs/cgroup/memory.current` | `/sys/fs/cgroup/memory.max` |
| v1 | `/sys/fs/cgroup/memory/memory.usage_in_bytes` | `/sys/fs/cgroup/memory/memory.limit_in_bytes` |

"Unlimited" sentinels (`"max"` on v2, `uint64::max`/`-1` on v1) both map to `UNLIMITED_MEMORY`.
The effective limit is `min(config.max_memory_bytes, cgroup limit)` (or just the cgroup limit when
the config field is 0):

```cpp
const uint64_t limit = max_memory_bytes_ > 0 ? std::min(max_memory_bytes_, raw_limit) : raw_limit;
if (limit == CgroupMemoryStatsReader::UNLIMITED_MEMORY) {
  usage_stats.resource_pressure_ = 0.0;     // no limit -> no pressure
} else {
  usage_stats.resource_pressure_ = double(usage) / limit;
}
```

Read/parse failures throw `EnvoyException`, forwarded via `onFailure`.

---

## downstream_connections (proactive)

| | |
|--|--|
| Config | `envoy.extensions.resource_monitors.downstream_connections.v3.DownstreamConnectionsConfig` |
| Resource name | `envoy.resource_monitors.global_downstream_max_connections` |
| Class | `ActiveDownstreamConnectionsResourceMonitor` (`downstream_connections/downstream_connections_monitor.h`) |
| Kind | **Proactive** |

The only proactive monitor. State is a single `std::atomic<int64_t> current_` compared against
`const int64_t max_` (= `config.max_active_downstream_connections()`). Allocation is a lock-free
CAS loop that refuses to exceed the cap:

```cpp
bool tryAllocateResource(int64_t increment) {
  auto current = current_.load(std::memory_order_relaxed);
  while (current + increment <= max_) {
    if (current_.compare_exchange_weak(current, current + increment,
            std::memory_order_release, std::memory_order_relaxed)) {
      return true;
    }
  }
  return false;   // would breach the global cap
}
```

`tryDeallocateResource` mirrors it (refusing to drop below 0). The connection-accept path calls
`tryAllocateResource(1)` when a downstream connection is created and `tryDeallocateResource(1)` on
close; a `false` return rejects the connection. (A `ThreadSynchronizer` with `try_allocate_pre_cas`
sync points exists only so tests can deterministically interleave the CAS loop.)

---

## injected_resource (test/synthetic)

| | |
|--|--|
| Config | `envoy.extensions.resource_monitors.injected_resource.v3.InjectedResourceConfig` |
| Resource name | `envoy.resource_monitors.injected_resource` |
| Class | `InjectedResourceMonitor` (`injected_resource/injected_resource_monitor.h`) |
| Kind | Regular |

Reads its pressure value directly from a file so integration tests can force Envoy into/out of
overload deterministically. It installs a filesystem watcher for `MovedTo` events (the file is
updated via atomic symlink/rename swap):

```cpp
watcher_->addWatch(filename_, Filesystem::Watcher::Events::MovedTo, [this](uint32_t) {
  onFileChanged();          // just sets a dirty flag
  return absl::OkStatus();
});
```

Reads are lazy: `onFileChanged()` sets `file_changed_`; the next `updateResourceUsage()` re-reads
and parses a float in `[0,1]`, reporting `onSuccess(pressure)` or `onFailure` for out-of-range /
unparseable content.

---

## Summary

| Monitor | Pressure formula | Stats seam | Kind |
|---------|-----------------|------------|------|
| fixed_heap | `used / max_heap` (used = allocated, or reserved−unmapped−free_mapped) | `MemoryStatsReader` → `Memory::Stats` | Regular |
| cpu_utilization | EWMA of `Δwork / Δtotal` (host or cgroup) | `CpuStatsReader` (`/proc/stat` or cgroup) | Regular |
| cgroup_memory | `usage / min(config, cgroup limit)` | `CgroupMemoryStatsReader` (v1/v2) | Regular |
| downstream_connections | `current / max` (atomic) | atomic counter | **Proactive** |
| injected_resource | value read from file | file watcher | Regular |
