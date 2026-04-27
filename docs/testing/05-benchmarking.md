# Benchmarking Guide

This guide covers performance testing and benchmarking in Envoy using Google Benchmark framework.

## Table of Contents

- [Overview](#overview)
- [Writing Benchmarks](#writing-benchmarks)
- [Running Benchmarks](#running-benchmarks)
- [Benchmark Patterns](#benchmark-patterns)
- [Best Practices](#best-practices)
- [Interpreting Results](#interpreting-results)

## Overview

Envoy uses [Google Benchmark](https://github.com/google/benchmark/) for microbenchmarks to:
- Measure performance characteristics
- Detect performance regressions
- Compare implementation alternatives
- Optimize hot paths
- Validate optimization attempts

### Benchmark vs Test

**Benchmarks**:
- Measure performance (time, throughput, memory)
- Run with optimizations enabled (`-c opt`)
- Report timing statistics
- May be skipped in CI to save time

**Tests**:
- Verify correctness
- Run with sanitizers
- Pass/fail binary result
- Always run in CI

### Benchmark Infrastructure

Envoy provides two bazel rules for benchmarks:

1. **envoy_cc_benchmark_binary**: For local benchmarking
   - Run with `bazel run -c opt //test/path:benchmark_name`
   - Reports full timing statistics
   - Can run indefinitely

2. **envoy_benchmark_test**: For CI validation
   - Run with `bazel test //test/path:benchmark_test`
   - Runs minimal iterations
   - Skips expensive benchmarks
   - Verifies benchmark compiles and runs

## Writing Benchmarks

### Basic Benchmark Structure

```cpp
#include "benchmark/benchmark.h"

static void BM_MyOperation(benchmark::State& state) {
  // Setup (not timed)
  MyObject obj;
  
  // Benchmark loop (timed)
  for (auto _ : state) {
    obj.performOperation();
  }
}

// Register benchmark
BENCHMARK(BM_MyOperation);

// Benchmark main (required)
BENCHMARK_MAIN();
```

### Complete Example

```cpp
#include "source/common/buffer/buffer_impl.h"
#include "benchmark/benchmark.h"

namespace Envoy {
namespace Buffer {

static void BM_BufferCreate(benchmark::State& state) {
  for (auto _ : state) {
    OwnedImpl buffer;
    benchmark::DoNotOptimize(buffer);
  }
}
BENCHMARK(BM_BufferCreate);

static void BM_BufferAddAndDrain(benchmark::State& state) {
  OwnedImpl buffer;
  const std::string data(state.range(0), 'a');
  
  for (auto _ : state) {
    buffer.add(data);
    buffer.drain(data.size());
  }
  
  state.SetBytesProcessed(
      static_cast<int64_t>(state.iterations()) * data.size());
}
BENCHMARK(BM_BufferAddAndDrain)->Range(1, 1024 * 1024);

static void BM_BufferLinearize(benchmark::State& state) {
  OwnedImpl buffer;
  
  // Add fragmented data
  for (int i = 0; i < 10; i++) {
    buffer.add(std::string(100, 'a'));
  }
  
  for (auto _ : state) {
    void* data = buffer.linearize(buffer.length());
    benchmark::DoNotOptimize(data);
  }
}
BENCHMARK(BM_BufferLinearize);

} // namespace Buffer
} // namespace Envoy

BENCHMARK_MAIN();
```

### Preventing Optimization

Google Benchmark provides utilities to prevent compiler from optimizing away your code:

```cpp
static void BM_ComputeValue(benchmark::State& state) {
  for (auto _ : state) {
    int result = expensiveComputation();
    
    // Prevent compiler from eliminating computation
    benchmark::DoNotOptimize(result);
  }
}

static void BM_UsePointer(benchmark::State& state) {
  std::vector<int> vec = {1, 2, 3, 4, 5};
  
  for (auto _ : state) {
    int* ptr = vec.data();
    
    // Prevent compiler from optimizing away pointer
    benchmark::DoNotOptimize(ptr);
    
    // Ensure memory is actually accessed
    benchmark::ClobberMemory();
  }
}
```

### Parameterized Benchmarks

```cpp
// Test with different input sizes
static void BM_ProcessData(benchmark::State& state) {
  const size_t size = state.range(0);
  std::vector<int> data(size, 42);
  
  for (auto _ : state) {
    processVector(data);
  }
}

// Run with sizes 8, 64, 512, 4096
BENCHMARK(BM_ProcessData)
    ->Arg(8)
    ->Arg(64)
    ->Arg(512)
    ->Arg(4096);

// Or use Range (generates powers of 2)
BENCHMARK(BM_ProcessData)->Range(8, 8 << 10);

// Or use DenseRange (every value in range)
BENCHMARK(BM_ProcessData)->DenseRange(1, 10, 1);

// Multiple arguments
static void BM_TwoParams(benchmark::State& state) {
  const size_t rows = state.range(0);
  const size_t cols = state.range(1);
  Matrix m(rows, cols);
  
  for (auto _ : state) {
    m.process();
  }
}
BENCHMARK(BM_TwoParams)
    ->Args({10, 10})
    ->Args({100, 100})
    ->Args({1000, 1000});
```

### Setup and Teardown

```cpp
static void BM_WithSetup(benchmark::State& state) {
  // Setup before timing starts
  ExpensiveObject obj;
  obj.initialize();
  
  for (auto _ : state) {
    // This is timed
    obj.operation();
  }
  
  // Teardown after timing ends
  obj.cleanup();
}

// Pause timing for expensive setup
static void BM_PauseResume(benchmark::State& state) {
  for (auto _ : state) {
    state.PauseTiming();
    // Expensive setup not timed
    auto data = createLargeDataset();
    state.ResumeTiming();
    
    // This is timed
    processDataset(data);
  }
}
```

### Reporting Metrics

```cpp
static void BM_ReportMetrics(benchmark::State& state) {
  size_t total_bytes = 0;
  size_t total_items = 0;
  
  for (auto _ : state) {
    auto result = processData();
    total_bytes += result.bytes;
    total_items += result.items;
  }
  
  // Report custom metrics
  state.SetBytesProcessed(total_bytes);
  state.SetItemsProcessed(total_items);
  
  // Custom counters
  state.counters["cache_hits"] = cache_hit_count;
  state.counters["cache_misses"] = cache_miss_count;
}
```

## Running Benchmarks

### Local Benchmark Runs

```bash
# Run benchmark with optimizations
bazel run -c opt //test/common/buffer:buffer_speed_test

# Run specific benchmark
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_filter=BM_BufferCreate

# Run for specific duration
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_min_time=10.0

# Run specific number of iterations
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_repetitions=5

# Output format
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_format=json --benchmark_out=results.json

# Show counters
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_counters_tabular=true
```

### CI Test Runs

```bash
# Run minimal benchmark test (for CI)
bazel test //test/common/buffer:buffer_benchmark_test

# This runs benchmarks with minimal iterations
# to verify they compile and execute
```

### Comparing Results

```bash
# Run baseline
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_out=baseline.json --benchmark_format=json

# Make changes to code

# Run comparison
bazel run -c opt //test/common/buffer:buffer_speed_test \
  -- --benchmark_out=comparison.json --benchmark_format=json

# Compare results (requires compare.py from Google Benchmark)
compare.py benchmarks baseline.json comparison.json
```

## Benchmark Patterns

### Benchmarking Buffer Operations

```cpp
#include "source/common/buffer/buffer_impl.h"

static void BM_BufferAppend(benchmark::State& state) {
  Buffer::OwnedImpl buffer;
  const std::string data(1024, 'x');
  
  for (auto _ : state) {
    buffer.add(data);
  }
  
  state.SetBytesProcessed(
      state.iterations() * data.size());
}
BENCHMARK(BM_BufferAppend);
```

### Benchmarking Header Operations

```cpp
#include "source/common/http/header_map_impl.h"

static void BM_HeaderMapCreate(benchmark::State& state) {
  for (auto _ : state) {
    Http::TestRequestHeaderMapImpl headers{
        {":method", "GET"},
        {":path", "/"},
        {":scheme", "https"},
        {":authority", "example.com"}};
    benchmark::DoNotOptimize(headers);
  }
}
BENCHMARK(BM_HeaderMapCreate);

static void BM_HeaderMapLookup(benchmark::State& state) {
  Http::TestRequestHeaderMapImpl headers{
      {":method", "GET"},
      {":path", "/api/v1/endpoint"},
      {"x-custom-header", "value"}};
  
  for (auto _ : state) {
    auto result = headers.get(Http::LowerCaseString("x-custom-header"));
    benchmark::DoNotOptimize(result);
  }
}
BENCHMARK(BM_HeaderMapLookup);
```

### Benchmarking Stats

```cpp
#include "source/common/stats/isolated_store_impl.h"

static void BM_CounterIncrement(benchmark::State& state) {
  Stats::IsolatedStoreImpl store;
  Stats::Counter& counter = 
      store.counterFromString("test.benchmark.counter");
  
  for (auto _ : state) {
    counter.inc();
  }
}
BENCHMARK(BM_CounterIncrement);

static void BM_HistogramRecord(benchmark::State& state) {
  Stats::IsolatedStoreImpl store;
  Stats::Histogram& histogram = 
      store.histogramFromString("test.benchmark.histogram", 
                                Stats::Histogram::Unit::Milliseconds);
  
  for (auto _ : state) {
    histogram.recordValue(42);
  }
}
BENCHMARK(BM_HistogramRecord);
```

### Benchmarking with Fixtures

```cpp
class MyFixture : public benchmark::Fixture {
public:
  void SetUp(const benchmark::State& state) override {
    // Setup done once per benchmark
    expensive_object_ = std::make_unique<ExpensiveObject>();
    expensive_object_->initialize();
  }
  
  void TearDown(const benchmark::State& state) override {
    expensive_object_.reset();
  }
  
  std::unique_ptr<ExpensiveObject> expensive_object_;
};

BENCHMARK_F(MyFixture, BM_Operation)(benchmark::State& state) {
  for (auto _ : state) {
    expensive_object_->operation();
  }
}

BENCHMARK_REGISTER_F(MyFixture, BM_Operation);
```

### Skipping Expensive Benchmarks in CI

```cpp
#include "test/benchmark/main.h"

static void BM_ExpensiveOperation(benchmark::State& state) {
  // Skip in CI/testing environment
  if (benchmark::skipExpensiveBenchmarks()) {
    state.SkipWithError("Skipping expensive benchmark");
    return;
  }
  
  // Expensive setup
  auto large_dataset = createGiantDataset();
  
  for (auto _ : state) {
    processDataset(large_dataset);
  }
}
BENCHMARK(BM_ExpensiveOperation);
```

### Multi-threaded Benchmarks

```cpp
static void BM_MultiThreaded(benchmark::State& state) {
  ThreadLocal::InstanceImpl tls;
  
  for (auto _ : state) {
    tls.runOnAllThreads([&] {
      // Operation on each thread
      performWork();
    });
  }
}
BENCHMARK(BM_MultiThreaded)
    ->Threads(1)
    ->Threads(2)
    ->Threads(4)
    ->Threads(8);
```

## Best Practices

### 1. Measure What Matters

Focus on hot paths and operations that impact performance:
```cpp
// Good: Benchmarking frequently-called operation
static void BM_HeaderLookup(benchmark::State& state) {
  Http::TestRequestHeaderMapImpl headers;
  for (auto _ : state) {
    headers.get(Http::LowerCaseString(":path"));
  }
}

// Less useful: Benchmarking one-time setup
static void BM_CreateConfig(benchmark::State& state) {
  for (auto _ : state) {
    auto config = std::make_shared<Config>();
  }
}
```

### 2. Use Realistic Data

Use production-like data sizes and patterns:
```cpp
static void BM_ParseHeaders(benchmark::State& state) {
  // Realistic request with typical headers
  const std::string request = 
      "GET /api/v1/users HTTP/1.1\r\n"
      "Host: api.example.com\r\n"
      "User-Agent: Mozilla/5.0\r\n"
      "Accept: application/json\r\n"
      "Authorization: Bearer token123\r\n"
      "X-Request-ID: abc-123\r\n"
      "\r\n";
  
  for (auto _ : state) {
    parseHttpRequest(request);
  }
}
```

### 3. Avoid Measurement Bias

```cpp
// Bad: Setup inside timing loop
static void BM_BadSetup(benchmark::State& state) {
  for (auto _ : state) {
    // This setup is timed!
    std::vector<int> data(1000000);
    processData(data);
  }
}

// Good: Setup outside or paused
static void BM_GoodSetup(benchmark::State& state) {
  std::vector<int> data(1000000);
  
  for (auto _ : state) {
    processData(data);
  }
}
```

### 4. Prevent Dead Code Elimination

```cpp
static void BM_PreventOptimization(benchmark::State& state) {
  for (auto _ : state) {
    int result = compute();
    
    // Compiler might eliminate compute() without this
    benchmark::DoNotOptimize(result);
  }
}
```

### 5. Test Multiple Scenarios

```cpp
// Test different input sizes
BENCHMARK(BM_Process)->Range(1, 1 << 20);

// Test different configurations
static void BM_WithConfig(benchmark::State& state) {
  const bool use_cache = state.range(0);
  const size_t buffer_size = state.range(1);
  
  Config config{use_cache, buffer_size};
  
  for (auto _ : state) {
    process(config);
  }
}
BENCHMARK(BM_WithConfig)
    ->Args({false, 1024})
    ->Args({true, 1024})
    ->Args({false, 4096})
    ->Args({true, 4096});
```

### 6. Document Benchmark Purpose

```cpp
// Benchmark header map creation to ensure we don't regress
// on the common case of creating request headers.
// This operation happens on every request.
static void BM_CreateRequestHeaders(benchmark::State& state) {
  for (auto _ : state) {
    Http::TestRequestHeaderMapImpl headers{
        {":method", "GET"},
        {":path", "/api"},
        {":scheme", "https"},
        {":authority", "example.com"}};
    benchmark::DoNotOptimize(headers);
  }
}
BENCHMARK(BM_CreateRequestHeaders);
```

## Interpreting Results

### Reading Benchmark Output

```
-------------------------------------------------------------------
Benchmark                         Time             CPU   Iterations
-------------------------------------------------------------------
BM_BufferCreate                  45 ns           45 ns     15555556
BM_BufferAppend/1              123 ns          123 ns      5684211
BM_BufferAppend/1024          1234 ns         1234 ns       567123
BM_BufferLinearize             456 ns          456 ns      1534091
```

**Columns**:
- **Benchmark**: Name and parameters
- **Time**: Wall-clock time per iteration
- **CPU**: CPU time per iteration
- **Iterations**: Number of times benchmark ran

### Understanding Metrics

```
BM_Process    1234 ns  1234 ns  567123  bytes_per_second=812.3M/s
```

**Additional metrics**:
- **bytes_per_second**: Throughput measurement
- **items_processed**: Custom counter
- Custom counters appear as additional columns

### Comparing Results

```
Benchmark                          Baseline    Current    Change
--------------------------------------------------------------------
BM_Operation                        100 ns      95 ns    -5.00%  ✓
BM_SlowOperation                   1000 ns    1200 ns   +20.00%  ✗
```

**Interpretation**:
- Negative change = faster (good)
- Positive change = slower (bad)
- Look for significant changes (>5%)

### Statistical Significance

Run benchmarks multiple times to verify results:
```bash
bazel run -c opt //test:benchmark \
  -- --benchmark_repetitions=10 \
     --benchmark_report_aggregates_only=true
```

Output includes statistics:
```
BM_Operation_mean         100 ns      100 ns
BM_Operation_median       100 ns      100 ns
BM_Operation_stddev         2 ns        2 ns
```

## Common Pitfalls

1. **Not running with -c opt**: Benchmarks without optimization are meaningless
2. **Including setup in timing**: Use PauseTiming() or setup outside loop
3. **Measuring too little work**: Benchmark overhead dominates
4. **Not preventing optimization**: Compiler eliminates the work
5. **Running on busy system**: Other processes affect timing
6. **Not using realistic data**: Results don't match production
7. **Comparing different machines**: Hardware differences affect results
8. **Ignoring variance**: Run multiple times to verify consistency

## Advanced Topics

### Custom Main Function

```cpp
#include "test/benchmark/main.h"

int main(int argc, char** argv) {
  // Custom initialization
  Envoy::TestEnvironment::initializeOptions(argc, argv);
  
  // Run benchmarks
  benchmark::Initialize(&argc, argv);
  if (benchmark::ReportUnrecognizedArguments(argc, argv)) {
    return 1;
  }
  benchmark::RunSpecifiedBenchmarks();
  
  // Cleanup
  benchmark::Shutdown();
  return 0;
}
```

### Shared State Between Iterations

```cpp
class SharedStateFixture : public benchmark::Fixture {
public:
  void SetUp(const benchmark::State& state) override {
    if (state.thread_index() == 0) {
      // First thread does setup
      shared_data_ = createSharedData();
    }
  }
  
  static std::shared_ptr<Data> shared_data_;
};

BENCHMARK_F(SharedStateFixture, BM_UseSharedData)(
    benchmark::State& state) {
  for (auto _ : state) {
    processData(*shared_data_);
  }
}
```

## Next Steps

- [Google Benchmark Documentation](https://github.com/google/benchmark)
- [Benchmark Examples](../../test/benchmark/): Real benchmark tests
- [Testing Overview](01-testing-overview.md): General testing information
- [Performance Best Practices](../../source/docs/): Envoy performance guides

## Contributing Benchmarks

When adding benchmarks:
1. Focus on hot paths and common operations
2. Use realistic data and scenarios
3. Document what you're measuring and why
4. Verify results are stable across runs
5. Add both binary and test targets
6. Consider adding to CI for regression detection
