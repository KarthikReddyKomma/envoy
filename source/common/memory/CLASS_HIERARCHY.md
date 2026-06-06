# Memory — class hierarchy (UML)

UML-style Mermaid for the memory utilities. Unlike most folders these are concrete classes with static methods
(no pure-virtual interfaces). See [`OVERVIEW.md`](OVERVIEW.md) for behavior.

---

## Components

```mermaid
classDiagram
    class Stats {
        <<static>>
        +totalCurrentlyAllocated()$ uint64_t
        +totalCurrentlyReserved()$ uint64_t
        +totalThreadCacheBytes()$ uint64_t
        +totalPageHeapUnmapped()$ uint64_t
        +totalPageHeapFree()$ uint64_t
        +totalPhysicalBytes()$ uint64_t
        +dumpStats()$ optional~string~
        +dumpStatsToLog()$
    }

    class Utils {
        <<static>>
        +releaseFreeMemory(max_unfreed=0)$
        +tryShrinkHeap()$
    }

    class AllocatorManager {
        -bytes_to_release_ : uint64_t
        -memory_release_interval_msec_ : ms
        -background_release_rate_bytes_per_second_ : size_t
        -api_ : Api::Api&
        -tcmalloc_thread_ : Thread::ThreadPtr
        +AllocatorManager(api, config)
        +~AllocatorManager()
        -configureBackgroundMemoryRelease()
        -configureTcmallocOptions(config)
    }

    class HeapShrinker {
        -active_ : bool
        -shrink_counter_ : Counter*
        -timer_ : TimerPtr
        -timer_interval_ : ms
        -max_unfreed_memory_bytes_ : uint64_t
        +HeapShrinker(dispatcher, overload_manager, scope)
        -shrinkHeap()
    }

    class AlignedAllocator~T, Alignment~ {
        +value_type : T
        +allocate(n) T*
        +deallocate(p, n)
        +round_up_to_alignment(bytes)$ size_t
        +rebind
    }

    AllocatorManager ..> Stats : (indirectly via tcmalloc)
    HeapShrinker ..> Utils : releaseFreeMemory()
    Utils ..> Stats : tryShrinkHeap reads metrics
    AllocatorManager ..> Allocator : tcmalloc ProcessBackgroundActions
    Utils ..> Allocator : ReleaseMemoryToSystem / purge
    Stats ..> Allocator : GetNumericProperty / mallctl

    note for AllocatorManager "owns the bg release thread\n(Google tcmalloc only)"
    note for HeapShrinker "overload-action driven\nperiodic release"
```

---

## How the pieces collaborate

```mermaid
flowchart LR
    subgraph Continuous
      AM["AllocatorManager"] -->|steady rate| ALLOC["allocator"]
    end
    subgraph OnOverload
      OM["OverloadManager"] --> HS["HeapShrinker"]
      HS --> U1["Utils::releaseFreeMemory"]
      U1 --> ALLOC
    end
    subgraph OnConfigChurn
      XDS["xDS / admin (main thread)"] --> U2["Utils::tryShrinkHeap"]
      U2 -->|gated by threshold| ALLOC
    end
    ALLOC --> S["Stats"] --> ADMIN["/memory + stats"]
```

---

## Free functions (namespace `Memory`)

| Function | Meaning |
|---|---|
| `maxUnfreedMemoryBytes()` | Read the global unfreed-memory threshold (default 100 MB). |
| `setMaxUnfreedMemoryBytes(v)` | Set it (atomic). |

---

## Build-time selection (not a class, but the key axis)

```mermaid
flowchart TD
    Build["build config"] --> A{"which allocator?"}
    A -->|TCMALLOC| G["Google tcmalloc<br/>(full feature set)"]
    A -->|GPERFTOOLS_TCMALLOC| GP["gperftools<br/>(no bg release / limits)"]
    A -->|JEMALLOC| J["jemalloc<br/>(epoch refresh, arena purge)"]
    A -->|none| N["all funcs return 0 / no-op"]
```

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `HeapShrinker` → `Utils` | uses | Releases on overload tick. |
| `Utils` → `Stats` | uses | `tryShrinkHeap` reads physical vs allocated. |
| `AllocatorManager` → allocator | owns thread | Background release loop. |
| `Stats`/`Utils`/`AllocatorManager` → allocator | compile-time `#if` | tcmalloc / gperftools / jemalloc / none. |
| `AlignedAllocator` → (STL) | allocator_traits | Standalone over-aligned allocator. |
