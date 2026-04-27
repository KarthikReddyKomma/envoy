# Memory and Performance Documentation

## Overview

This section provides comprehensive documentation on Envoy's memory management and performance characteristics. Understanding these systems is critical for building high-performance, memory-efficient Envoy deployments and extensions.

### What You'll Learn

- **Threading Model** - Multi-threaded architecture, thread-local storage, event loops
- **Buffer Management** - Zero-copy data structures, slice management, watermarks
- **Memory Management** - TCMalloc, heap profiling, memory accounting
- **Performance Optimization** - Lock-free patterns, profiling tools, best practices

---

## Documentation Structure

### 01. Threading Model

**File:** [01-threading-model.md](01-threading-model.md)

Learn about Envoy's multi-threaded architecture:

- Main thread vs worker threads
- Thread Local Storage (TLS) for lock-free data access
- Event loop (dispatcher) architecture
- Thread-safe design patterns
- Cross-thread communication via post()

**Key Concepts:**
```
Main Thread → Configuration, Admin, xDS
Worker Threads → Connection handling, request processing
TLS → Replicated data per-thread (zero-lock reads)
Dispatcher → Event loop per thread (libevent)
```

**When to read this:**
- Building new filters or extensions
- Debugging threading issues
- Understanding Envoy's concurrency model
- Optimizing for multi-core systems

---

### 02. Buffer Management

**File:** [02-buffer-management.md](02-buffer-management.md)

Deep dive into Envoy's buffer system:

- Buffer::Instance interface
- Slice-based memory management (16 KB default)
- Zero-copy operations (move, fragments)
- Reservation API for efficient I/O
- Watermarks for flow control
- Memory accounting per-stream

**Key Concepts:**
```
Buffer → Collection of slices
Slice → 16 KB memory block (drained, data, reservable)
Move → Zero-copy transfer between buffers
Reservation → Direct I/O into buffer (avoid copy)
Watermarks → Back-pressure mechanism
```

**When to read this:**
- Implementing filters that process data
- Debugging buffer-related issues
- Optimizing data path performance
- Understanding memory usage patterns

---

### 03. Memory Management

**File:** [03-memory-management.md](03-memory-management.md)

Complete guide to memory allocation and tracking:

- TCMalloc (Thread-Caching Malloc) integration
- Memory statistics and monitoring
- Heap profiling with pprof
- Heap shrinker for memory release
- Smart pointer usage patterns
- Leak detection with sanitizers

**Key Concepts:**
```
TCMalloc → High-performance allocator
Thread Cache → Per-thread allocation cache
Page Heap → Large object allocator
Memory Stats → Runtime tracking
Heap Profiler → Allocation analysis
```

**When to read this:**
- Investigating memory leaks
- Profiling memory usage
- Configuring memory limits
- Optimizing allocation patterns
- Understanding memory overhead

---

### 04. Performance Optimization

**File:** [04-performance-optimization.md](04-performance-optimization.md)

Best practices for high-performance code:

- Lock-free design patterns
- Zero-copy techniques
- Cache-friendly code
- Efficient allocation strategies
- Profiling tools (pprof, perf, benchmarks)
- Compiler optimizations

**Key Concepts:**
```
Lock-Free → TLS, atomics, RCU
Zero-Copy → Buffer move, fragments, string_view
Cache-Friendly → Sequential access, data locality
Profiling → CPU profiling, heap profiling, benchmarks
```

**When to read this:**
- Optimizing filter performance
- Debugging performance issues
- Conducting performance analysis
- Setting up continuous profiling

---

## Quick Start

### For Filter Developers

1. **Understand threading** → Read [01-threading-model.md](01-threading-model.md)
   - Learn about TLS for lock-free data access
   - Understand event loop execution order

2. **Learn buffer operations** → Read [02-buffer-management.md](02-buffer-management.md)
   - Use `move()` instead of `add()` when possible
   - Understand watermarks for flow control

3. **Profile your code** → Read [04-performance-optimization.md](04-performance-optimization.md)
   - Set up benchmarks
   - Use pprof for profiling
   - Follow optimization best practices

### For Operators

1. **Configure memory limits** → Read [03-memory-management.md](03-memory-management.md)
   - Set buffer limits
   - Configure heap shrinker
   - Monitor memory metrics

2. **Performance tuning** → Read [01-threading-model.md](01-threading-model.md) and [04-performance-optimization.md](04-performance-optimization.md)
   - Set worker thread count
   - Configure connection limits
   - Enable profiling endpoints

3. **Monitoring** → Read all sections
   - Key metrics to monitor
   - Admin endpoints for diagnostics
   - Performance dashboards

---

## Common Use Cases

### Debugging Memory Leaks

1. Enable heap profiling:
   ```bash
   export HEAPPROFILE=/tmp/envoy.heap
   envoy -c envoy.yaml
   ```

2. Analyze with pprof:
   ```bash
   pprof --http=:8080 envoy /tmp/envoy.heap.0001.heap
   ```

3. See [03-memory-management.md](03-memory-management.md) for details.

### Optimizing Filter Performance

1. Profile CPU usage:
   ```bash
   curl -X POST localhost:9901/cpuprofiler?enable=y
   # Generate load
   curl localhost:9901/cpuprofiler?enable=n > profile.prof
   pprof --http=:8080 envoy profile.prof
   ```

2. Identify bottlenecks in flame graph

3. Apply optimizations from [04-performance-optimization.md](04-performance-optimization.md)

4. Benchmark improvements

### Investigating High Memory Usage

1. Check memory stats:
   ```bash
   curl localhost:9901/memory
   ```

2. Dump heap profile:
   ```bash
   curl -X POST localhost:9901/heap_dump
   ```

3. Analyze allocations per component

4. See [03-memory-management.md](03-memory-management.md) for configuration options

### Understanding Threading Issues

1. Check thread-safety:
   - Review [01-threading-model.md](01-threading-model.md)
   - Verify thread-local data access patterns
   - Check for shared mutable state

2. Debug with thread sanitizer:
   ```bash
   bazel build --config=tsan //source/exe:envoy-static
   ```

3. Review cross-thread communication patterns

---

## Performance Metrics

### Key Metrics to Monitor

**Memory:**
```bash
# Total allocated
server.memory.allocated

# Heap size
server.memory.heap_size

# Per-connection buffer usage
http.*.downstream_cx_buffer_bytes_allocated
```

**Performance:**
```bash
# Request rate
http.*.downstream_rq_total

# Latency (P50, P99, P999)
http.*.downstream_rq_time

# Worker thread utilization
worker_*.watchdog_miss
```

**Buffers:**
```bash
# Buffer usage
buffer.*.total_size
buffer.*.watermark_*
```

### Admin Endpoints

```bash
# Memory statistics
GET /memory

# Heap profile
POST /heap_dump

# CPU profile (start)
POST /cpuprofiler?enable=y

# CPU profile (stop and download)
GET /cpuprofiler?enable=n

# Stats
GET /stats
GET /stats?format=json
GET /stats?filter=memory

# Thread dump
GET /threads
```

---

## Configuration Examples

### Minimal Memory Configuration

```yaml
# Low memory footprint
memory_allocator_manager:
  bytes_to_release: 10485760  # 10 MB
  memory_release_interval_msec: 1000

static_resources:
  listeners:
  - per_connection_buffer_limit_bytes: 32768  # 32 KB
```

### High Performance Configuration

```yaml
# Maximum performance
static_resources:
  listeners:
  - enable_reuse_port: true
    per_connection_buffer_limit_bytes: 1048576  # 1 MB

# Worker thread count (match physical cores)
node:
  metadata:
    envoy.config.bootstrap.concurrency: 16

memory_allocator_manager:
  bytes_to_release: 209715200  # 200 MB
  tcmalloc_options:
    max_total_thread_cache_bytes: 52428800  # 50 MB
```

### Production Monitoring

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901

stats_sinks:
- name: envoy.stat_sinks.statsd
  typed_config:
    "@type": type.googleapis.com/envoy.config.metrics.v3.StatsdSink
    address:
      socket_address:
        address: 127.0.0.1
        port_value: 8125
```

---

## Best Practices Summary

### Memory Management

✅ **DO:**
- Set buffer limits to prevent unbounded growth
- Configure heap shrinker for long-running processes
- Monitor memory metrics continuously
- Profile memory usage under load
- Use smart pointers (unique_ptr, shared_ptr)

❌ **DON'T:**
- Allow unbounded buffer growth
- Ignore memory warnings
- Leak connections or streams
- Use raw pointers for ownership

### Performance

✅ **DO:**
- Use thread-local storage (TLS) for shared data
- Prefer `buffer.move()` over `buffer.add()`
- Use reservations for socket I/O
- Profile before optimizing
- Benchmark changes

❌ **DON'T:**
- Use locks on hot path
- Copy buffers unnecessarily
- Allocate in tight loops
- Optimize without profiling
- Trust intuition over measurements

### Threading

✅ **DO:**
- Keep connections on their original thread
- Use TLS for lock-free reads
- Use post() for cross-thread communication
- Validate thread safety with ASSERT(isThreadSafe())

❌ **DON'T:**
- Access connections from wrong thread
- Share mutable state without synchronization
- Use fine-grained locking
- Migrate connections between threads

---

## Additional Resources

### Source Code

- **Event Loop:** `source/common/event/`
- **Thread Local:** `source/common/thread_local/`
- **Buffer:** `source/common/buffer/`
- **Memory:** `source/common/memory/`

### External Documentation

- [TCMalloc Documentation](https://github.com/google/tcmalloc/tree/master/docs)
- [pprof User Guide](https://github.com/google/pprof)
- [libevent Documentation](https://libevent.org/)
- [Envoy Official Docs](https://www.envoyproxy.io/docs/envoy/latest/)

### Related Documentation

- **Architecture:** `docs/ENVOY_ARCHITECTURE_REFERENCE.md`
- **Event Loop:** `source/common/event/EVENT_LOOP_ARCHITECTURE.md`
- **Filters:** `docs/filter-chain/`
- **Testing:** `docs/testing/`

---

## Contributing

Found an issue or have suggestions for improving this documentation?

1. Check existing documentation in `source/common/*/`
2. Review code examples in `test/`
3. Submit improvements via pull request

---

## Glossary

**Buffer** - Data structure for managing byte streams

**Dispatcher** - Event loop that processes I/O events, timers, and callbacks

**Slice** - 16 KB memory block within a buffer

**TLS** - Thread Local Storage, per-thread data replication

**TCMalloc** - Thread-Caching Malloc, high-performance allocator

**Watermark** - Threshold for triggering back-pressure

**Zero-Copy** - Data transfer without copying (using move or reference)

**RCU** - Read-Copy-Update, pattern for infrequent updates to read-heavy data

**Lock-Free** - Data structure or algorithm that doesn't use locks

**Hot Path** - Frequently executed code (critical for performance)

**Cold Path** - Infrequently executed code (not performance critical)

---

## Support

For questions or issues:

- **General questions:** [envoy-users mailing list](https://groups.google.com/forum/#!forum/envoy-users)
- **Bug reports:** [GitHub Issues](https://github.com/envoyproxy/envoy/issues)
- **Slack:** [Envoy Slack](https://envoyslack.cncf.io/)
