# Debugging Tools and Techniques

This document covers the tools and techniques available for debugging Envoy, including the admin interface, debuggers, sanitizers, and profiling tools.

## Table of Contents

1. [Admin Interface](#admin-interface)
2. [GDB Debugging](#gdb-debugging)
3. [Core Dump Analysis](#core-dump-analysis)
4. [Memory Debugging](#memory-debugging)
5. [Thread Sanitizer](#thread-sanitizer)
6. [Performance Profiling](#performance-profiling)
7. [Network Debugging](#network-debugging)
8. [Configuration Debugging](#configuration-debugging)

---

## Admin Interface

The admin interface provides runtime visibility and control over Envoy.

### Enabling the Admin Interface

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901
```

### Essential Admin Endpoints

#### 1. Stats and Metrics

```bash
# Get all stats
curl http://localhost:9901/stats

# Filter stats
curl http://localhost:9901/stats?filter=http
curl http://localhost:9901/stats?filter=cluster.*upstream

# Get stats in Prometheus format
curl http://localhost:9901/stats/prometheus

# Reset stats
curl -X POST http://localhost:9901/stats/recentlookups/disable
curl -X POST http://localhost:9901/stats/recentlookups/enable
```

**Key Stats for Debugging**:
```bash
# Connection stats
curl http://localhost:9901/stats | grep -E "downstream_cx_|upstream_cx_"

# HTTP stats
curl http://localhost:9901/stats | grep -E "http.*rq_|http.*rs_"

# Timeout stats
curl http://localhost:9901/stats | grep timeout

# Error stats
curl http://localhost:9901/stats | grep -E "error|failure|reject"

# Cluster health
curl http://localhost:9901/stats | grep cluster.*health
```

#### 2. Configuration Dump

```bash
# Dump all configuration
curl http://localhost:9901/config_dump

# Pretty print with jq
curl -s http://localhost:9901/config_dump | jq '.'

# Extract specific config
curl -s http://localhost:9901/config_dump | jq '.configs[] | select(.["@type"] | contains("Listener"))'
curl -s http://localhost:9901/config_dump | jq '.configs[] | select(.["@type"] | contains("Cluster"))'
curl -s http://localhost:9901/config_dump | jq '.configs[] | select(.["@type"] | contains("Route"))'
```

**Useful Config Queries**:
```bash
# List all listeners
curl -s http://localhost:9901/config_dump | jq '.configs[0].dynamic_listeners[].name'

# List all clusters
curl -s http://localhost:9901/config_dump | jq '.configs[1].dynamic_active_clusters[].cluster.name'

# Show specific cluster config
curl -s http://localhost:9901/config_dump | \
  jq '.configs[1].dynamic_active_clusters[] | select(.cluster.name == "my_cluster")'

# Show route configuration
curl -s http://localhost:9901/config_dump | \
  jq '.configs[3].dynamic_route_configs[].route_config'
```

#### 3. Clusters and Endpoints

```bash
# List all clusters
curl http://localhost:9901/clusters

# Format output
curl -s http://localhost:9901/clusters?format=json | jq '.'

# Check specific cluster health
curl http://localhost:9901/clusters | grep -A 20 "my_cluster"

# View cluster circuit breaker stats
curl http://localhost:9901/clusters | grep -E "cx_|rq_"
```

**Cluster Status Interpretation**:
```
healthy: Host is healthy and receiving traffic
unhealthy: Host failed health check
degraded: Host is degraded but still receiving traffic
draining: Host is being drained
timeout: Health check timed out
```

#### 4. Runtime Configuration

```bash
# Get runtime values
curl http://localhost:9901/runtime

# Modify runtime (if enabled)
curl -X POST "http://localhost:9901/runtime_modify?key=feature.enabled&value=1"
```

#### 5. Logging Control

```bash
# Get current log levels
curl http://localhost:9901/logging

# Set log level for all loggers
curl -X POST "http://localhost:9901/logging?level=debug"

# Set specific logger levels
curl -X POST "http://localhost:9901/logging?router=trace&connection=debug"

# Reset to default
curl -X POST "http://localhost:9901/logging?level=info"
```

#### 6. Server Info

```bash
# Get server information
curl http://localhost:9901/server_info

# Health check endpoint
curl http://localhost:9901/ready
curl http://localhost:9901/healthcheck/ok
curl http://localhost:9901/healthcheck/fail

# Get command line options
curl http://localhost:9901/server_info | jq '.command_line_options'
```

#### 7. Listeners

```bash
# List active listeners
curl http://localhost:9901/listeners

# Listener stats
curl http://localhost:9901/stats | grep listener
```

#### 8. Memory and CPU

```bash
# Memory usage
curl http://localhost:9901/memory

# CPU profiling (if enabled)
curl http://localhost:9901/cpuprofiler?enable=y
# ... run workload ...
curl http://localhost:9901/cpuprofiler?enable=n

# Heap profiling
curl http://localhost:9901/heapprofiler?enable=y
# ... run workload ...
curl http://localhost:9901/heapprofiler?enable=n
```

#### 9. Connection Draining

```bash
# Drain listeners (graceful shutdown)
curl -X POST http://localhost:9901/drain_listeners

# Quit server
curl -X POST http://localhost:9901/quitquitquit

# Immediate shutdown
curl -X POST http://localhost:9901/shutdown
```

#### 10. Tracing and Debugging

```bash
# Dump current runtime config
curl http://localhost:9901/runtime

# Connection information
curl http://localhost:9901/listeners

# Get certificates
curl http://localhost:9901/certs
```

---

## GDB Debugging

### Building with Debug Symbols

```bash
# Build with debug info
bazel build -c dbg //source/exe:envoy-static

# Or with optimized debug
bazel build -c opt --copt=-g //source/exe:envoy-static
```

### Starting Envoy in GDB

```bash
# Start with GDB
gdb --args ./bazel-bin/source/exe/envoy-static -c config.yaml

# Common GDB commands
(gdb) run                          # Start execution
(gdb) break main                   # Set breakpoint at main
(gdb) break ClassName::method      # Break at method
(gdb) continue                     # Continue execution
(gdb) next                         # Step over
(gdb) step                         # Step into
(gdb) finish                       # Step out
(gdb) print variable               # Print variable
(gdb) backtrace                    # Show stack trace
(gdb) info threads                 # List threads
(gdb) thread 3                     # Switch to thread 3
```

### Setting Breakpoints

```bash
# Break on function
(gdb) break Envoy::Router::Filter::decodeHeaders

# Break on file:line
(gdb) break source/common/http/conn_manager_impl.cc:123

# Conditional breakpoint
(gdb) break ClassName::method if variable == value

# Break on exception
(gdb) catch throw
(gdb) catch catch

# List breakpoints
(gdb) info breakpoints

# Delete breakpoint
(gdb) delete 1
```

### Inspecting State

```bash
# Print variable
(gdb) print stream_info
(gdb) print stream_info.responseCode()

# Print structure
(gdb) print *headers
(gdb) print headers->Path()->value().getStringView()

# Print array
(gdb) print array[0]@10  # Print first 10 elements

# Call methods
(gdb) call object.method()
(gdb) call stream_info.setResponseCode(500)

# Examine memory
(gdb) x/10x address     # 10 hex words
(gdb) x/s address       # String
```

### Debugging Multi-threaded Code

```bash
# List all threads
(gdb) info threads

# Switch thread
(gdb) thread 5

# Break in specific thread
(gdb) break file.cc:line thread 5

# Apply command to all threads
(gdb) thread apply all backtrace

# Set scheduler locking
(gdb) set scheduler-locking on   # Only current thread runs
(gdb) set scheduler-locking off  # All threads run
```

### GDB Python Scripts

Create `.gdbinit` in project root:

```python
python
import sys
sys.path.insert(0, '/path/to/envoy/tools/gdb')
end

# Custom commands
define print_headers
  python
  # Print HTTP headers
  headers = gdb.parse_and_eval("headers")
  # ... implementation
  end
end
```

### Useful GDB Tricks

```bash
# Save breakpoints
(gdb) save breakpoints breakpoints.txt

# Load breakpoints
(gdb) source breakpoints.txt

# Log output to file
(gdb) set logging on
(gdb) set logging file debug.log

# Pretty printing
(gdb) set print pretty on
(gdb) set print object on

# Attach to running process
gdb -p <pid>
```

---

## Core Dump Analysis

### Enabling Core Dumps

```bash
# Check current limit
ulimit -c

# Enable unlimited core dumps
ulimit -c unlimited

# Set core dump pattern
echo "/tmp/cores/core.%e.%p.%t" | sudo tee /proc/sys/kernel/core_pattern
```

### Analyzing Core Dumps

```bash
# Load core dump in GDB
gdb ./envoy-static /tmp/cores/core.envoy.12345.1234567890

# Basic analysis
(gdb) backtrace              # Show crash stack trace
(gdb) backtrace full         # Show with local variables
(gdb) thread apply all bt    # Stack trace for all threads

# Examine crash context
(gdb) info registers
(gdb) info locals
(gdb) info args

# Navigate stack
(gdb) frame 3                # Jump to frame 3
(gdb) up                     # Move up stack
(gdb) down                   # Move down stack

# Examine state at crash
(gdb) print this
(gdb) print *this
```

### Common Crash Patterns

#### 1. Null Pointer Dereference

```bash
(gdb) backtrace
#0  0x... in ClassName::method (this=0x123, ptr=0x0)
#1  0x... in caller()

(gdb) frame 0
(gdb) print ptr
$1 = (Type *) 0x0

# Find where ptr should have been set
(gdb) print this->member_
```

#### 2. Stack Overflow

```bash
# Very deep stack trace
(gdb) backtrace
# ... hundreds of frames ...

# Check for recursion
(gdb) frame 0
(gdb) frame 100
# Compare - likely recursive
```

#### 3. Heap Corruption

```bash
# Crash in malloc/free
(gdb) backtrace
#0  0x... in malloc()
#1  0x... in std::vector::push_back()

# This often indicates earlier corruption
# Need to run with ASAN to catch
```

---

## Memory Debugging

### Address Sanitizer (ASAN)

#### Building with ASAN

```bash
# Build with ASAN
bazel build -c dbg --config=asan //source/exe:envoy-static

# Run with ASAN
./bazel-bin/source/exe/envoy-static -c config.yaml
```

#### ASAN Environment Variables

```bash
# Detect leaks
export ASAN_OPTIONS=detect_leaks=1

# Abort on error
export ASAN_OPTIONS=abort_on_error=1

# Log to file
export ASAN_OPTIONS=log_path=/tmp/asan.log

# Symbolize stack traces
export ASAN_SYMBOLIZER_PATH=/usr/bin/llvm-symbolizer

# Combined options
export ASAN_OPTIONS=detect_leaks=1:abort_on_error=1:log_path=/tmp/asan.log
```

#### ASAN Error Types

1. **Heap buffer overflow**:
   ```
   ERROR: AddressSanitizer: heap-buffer-overflow
   READ of size 4 at 0x... thread T0
   ```

2. **Use after free**:
   ```
   ERROR: AddressSanitizer: heap-use-after-free
   WRITE of size 8 at 0x... thread T0
   ```

3. **Stack buffer overflow**:
   ```
   ERROR: AddressSanitizer: stack-buffer-overflow
   READ of size 1 at 0x... thread T0
   ```

4. **Global buffer overflow**:
   ```
   ERROR: AddressSanitizer: global-buffer-overflow
   ```

5. **Memory leak**:
   ```
   Direct leak of 1024 bytes in 1 object(s)
   ```

### Memory Leak Detection

#### Using ASAN Leak Detector

```bash
# Enable leak detection
export ASAN_OPTIONS=detect_leaks=1

# Run Envoy
./envoy-static -c config.yaml

# Ctrl+C to exit - leak report will be generated
```

#### Using Valgrind

```bash
# Install Valgrind
sudo apt-get install valgrind

# Run with Valgrind
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --verbose \
         --log-file=valgrind.log \
         ./envoy-static -c config.yaml

# Analyze results
grep "definitely lost" valgrind.log
grep "indirectly lost" valgrind.log
```

#### Using Heap Profiler

```bash
# Build with tcmalloc heap profiler
bazel build -c opt --define tcmalloc=gperftools //source/exe:envoy-static

# Run with profiling
HEAPPROFILE=/tmp/envoy.hprof ./envoy-static -c config.yaml

# Analyze with pprof
pprof --text ./envoy-static /tmp/envoy.hprof.0001.heap
pprof --web ./envoy-static /tmp/envoy.hprof.0001.heap
```

### Analyzing Memory Usage

```bash
# Runtime memory stats via admin interface
curl http://localhost:9901/memory

# Detailed stats
curl -s http://localhost:9901/memory | jq '.'

# Monitor memory over time
watch -n 5 'curl -s http://localhost:9901/memory | jq ".total_allocated_bytes"'
```

---

## Thread Sanitizer

### Building with TSAN

```bash
# Build with Thread Sanitizer
bazel build -c dbg --config=tsan //source/exe:envoy-static

# Run
./bazel-bin/source/exe/envoy-static -c config.yaml
```

### TSAN Environment Variables

```bash
# Set options
export TSAN_OPTIONS=halt_on_error=1:second_deadlock_stack=1

# Log to file
export TSAN_OPTIONS=log_path=/tmp/tsan.log

# Suppressions file
export TSAN_OPTIONS=suppressions=/path/to/tsan.supp
```

### TSAN Error Types

1. **Data Race**:
   ```
   WARNING: ThreadSanitizer: data race
   Write of size 4 at 0x... by thread T2
   Previous write of size 4 at 0x... by thread T1
   ```

2. **Deadlock**:
   ```
   WARNING: ThreadSanitizer: lock-order-inversion (potential deadlock)
   ```

3. **Thread Leak**:
   ```
   WARNING: ThreadSanitizer: thread leak
   ```

### Debugging Data Races

```bash
# Get full stack traces
export TSAN_OPTIONS=history_size=7

# Second deadlock stack
export TSAN_OPTIONS=second_deadlock_stack=1

# Run and capture output
./envoy-static -c config.yaml 2>&1 | tee tsan.log

# Analyze
grep -A 20 "WARNING: ThreadSanitizer" tsan.log
```

---

## Performance Profiling

### CPU Profiling with perf

```bash
# Record CPU profile
sudo perf record -F 99 -p <envoy-pid> -g -- sleep 30

# Generate report
sudo perf report

# Generate flame graph
sudo perf script | ./FlameGraph/stackcollapse-perf.pl | \
  ./FlameGraph/flamegraph.pl > envoy-profile.svg
```

### CPU Profiling with gperftools

```bash
# Build with profiler
bazel build -c opt --define=tcmalloc=gperftools //source/exe:envoy-static

# Enable profiling via admin
curl -X POST http://localhost:9901/cpuprofiler?enable=y

# Run workload...

# Disable and save
curl -X POST http://localhost:9901/cpuprofiler?enable=n

# Analyze profile
pprof --text ./envoy-static /tmp/envoy.prof
pprof --web ./envoy-static /tmp/envoy.prof
```

### Heap Profiling

```bash
# Enable heap profiling via admin
curl -X POST http://localhost:9901/heapprofiler?enable=y

# Run workload...

# Disable
curl -X POST http://localhost:9901/heapprofiler?enable=n

# Analyze
pprof --text ./envoy-static /tmp/envoy.hprof
pprof --pdf ./envoy-static /tmp/envoy.hprof > heap.pdf
```

### Admin Stats for Performance

```bash
# Request latency
curl http://localhost:9901/stats | grep duration

# Connection pool stats
curl http://localhost:9901/stats | grep cx_connect_ms

# Upstream timing
curl http://localhost:9901/stats | grep upstream_rq_time

# Circuit breaker stats
curl http://localhost:9901/stats | grep cx_pool
```

---

## Network Debugging

### tcpdump

```bash
# Capture traffic to Envoy
sudo tcpdump -i any -w envoy.pcap port 8000

# Capture upstream traffic
sudo tcpdump -i any -w upstream.pcap dst port 8080

# Read capture
tcpdump -r envoy.pcap -A

# Filter HTTP
tcpdump -r envoy.pcap -A 'tcp port 8000 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'
```

### Wireshark

```bash
# Capture and open in Wireshark
sudo tcpdump -i any -w capture.pcap port 8000
wireshark capture.pcap

# Wireshark filters:
# - http
# - tcp.stream eq 0
# - ip.addr == 192.168.1.100
```

### strace

```bash
# Trace system calls
strace -p <envoy-pid> -f -e trace=network

# Trace file operations
strace -p <envoy-pid> -e trace=file

# Trace with timestamps
strace -tt -p <envoy-pid>

# Save to file
strace -o envoy.strace -p <envoy-pid>
```

### lsof

```bash
# List open files and sockets
lsof -p <envoy-pid>

# List network connections
lsof -p <envoy-pid> -a -i

# List listening ports
lsof -p <envoy-pid> -a -i -sTCP:LISTEN
```

### netstat/ss

```bash
# Show Envoy connections
netstat -anp | grep envoy

# Show connection states
ss -tan state established | grep :8000

# Show listening ports
ss -tlnp | grep envoy
```

---

## Configuration Debugging

### Validating Configuration

```bash
# Validate config file
./envoy-static --mode validate -c config.yaml

# Validate with detailed output
./envoy-static --mode validate -c config.yaml -l debug

# Check specific components
./envoy-static --mode validate -c config.yaml --component-log-level config:debug
```

### Configuration Test Server

```bash
# Start Envoy with test mode
./envoy-static -c config.yaml --mode serve --drain-time-s 1

# Test configuration changes
curl -X POST http://localhost:9901/drain_listeners
# Update config
# Restart Envoy
```

### Common Configuration Issues

```bash
# Check for typos in config dump
curl -s http://localhost:9901/config_dump | jq . > config.json
# Review config.json

# Verify listener is active
curl http://localhost:9901/listeners | grep -A 10 "listener_name"

# Check cluster configuration
curl -s http://localhost:9901/config_dump | \
  jq '.configs[1].dynamic_active_clusters[] | .cluster.name'

# Verify routes
curl -s http://localhost:9901/config_dump | \
  jq '.configs[3].dynamic_route_configs[].route_config.virtual_hosts[].routes'
```

### Schema Validation

```bash
# Use envoy configuration tools
bazel run //tools/config_validation:validate_config -- \
  --config-path config.yaml

# Check for deprecated fields
bazel run //tools:deprecated_features -- config.yaml
```

---

## Debugging Cheat Sheet

### Quick Diagnostics

```bash
# Is Envoy healthy?
curl http://localhost:9901/ready

# Recent errors
curl http://localhost:9901/stats | grep -E "_error|_failure" | grep -v ": 0"

# Connection issues
curl http://localhost:9901/stats | grep -E "cx_connect_fail|upstream_cx_connect_timeout"

# Timeout issues
curl http://localhost:9901/stats | grep timeout | grep -v ": 0"

# Cluster health
curl http://localhost:9901/clusters | grep "healthy"
```

### Live Debugging Session

```bash
# Terminal 1: Enable debug logging
curl -X POST "http://localhost:9901/logging?level=debug"

# Terminal 2: Tail logs
tail -f /var/log/envoy/envoy.log

# Terminal 3: Watch stats
watch -n 1 'curl -s http://localhost:9901/stats | grep http.ingress'

# Terminal 4: Generate traffic or issue
curl http://localhost:8000/api/endpoint
```

---

## Best Practices

1. **Use admin interface for runtime debugging** - Don't restart Envoy unnecessarily
2. **Enable appropriate sanitizers during development** - Catch bugs early
3. **Always validate configuration before deployment**
4. **Use structured logging** - Makes parsing easier
5. **Profile before optimizing** - Know where the bottlenecks are
6. **Keep symbols for production binaries** - Enables post-mortem debugging
7. **Monitor stats continuously** - Catch issues before they become critical
8. **Use core dumps wisely** - Can consume significant disk space

---

## Related Documentation

- [Error Handling Patterns](01-error-handling-patterns.md)
- [Logging and Tracing](02-logging-and-tracing.md)
- [Common Issues](04-common-issues.md)

## References

- Envoy Admin Interface Documentation
- GDB Documentation: https://sourceware.org/gdb/documentation/
- ASAN Documentation: https://github.com/google/sanitizers
- perf Documentation: https://perf.wiki.kernel.org/
