# Testing Overview

This document provides an overview of Envoy's testing philosophy, infrastructure, and the different types of tests used throughout the codebase.

## Testing Philosophy

Envoy follows a comprehensive testing strategy with several core principles:

1. **Test at Multiple Levels**: Unit tests for logic, integration tests for end-to-end flows, benchmarks for performance, and fuzz tests for robustness
2. **Deterministic Tests**: Tests should be reliable and not flaky, using simulated time and controlled environments
3. **Fast Feedback**: Tests should run quickly to enable rapid development iteration
4. **Real-World Scenarios**: Integration tests should reflect actual usage patterns
5. **Defense in Depth**: Multiple layers of testing to catch different types of bugs

## Test Types

### Unit Tests

Unit tests verify individual classes, functions, and modules in isolation.

**Purpose**: Test logic, edge cases, error handling, and specific behaviors

**Location**: Typically in `test/common/`, `test/extensions/`, etc.

**Characteristics**:
- Fast execution (milliseconds)
- Test single components in isolation
- Use mock objects for dependencies
- No network I/O or file system access
- Deterministic behavior

**Example**:
```cpp
TEST_F(BufferTest, AddBufferFragmentNoCleanup) {
  char input[] = "hello world";
  BufferFragmentImpl* frag = new BufferFragmentImpl(
      input, 11, nullptr);
  Buffer::OwnedImpl buffer;
  buffer.addBufferFragment(*frag);
  EXPECT_EQ(11, buffer.length());
  EXPECT_STREQ("hello world", buffer.toString().c_str());
}
```

### Integration Tests

Integration tests verify complete request flows through Envoy from downstream client to upstream server.

**Purpose**: Test end-to-end behavior, filter chains, protocol handling, and real interactions

**Location**: `test/integration/`

**Characteristics**:
- Tests downstream-Envoy-upstream communication
- Uses fake upstream servers
- Tests configuration and runtime behavior
- Slower than unit tests (seconds)
- Tests filter chains and protocol handling

**Example**:
```cpp
TEST_P(IntegrationTest, RouterRequestAndResponseWithBodyNoBuffer) {
  initialize();
  
  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = codec_client_->makeRequestWithBody(
      default_request_headers_, 1024);
  
  waitForNextUpstreamRequest();
  upstream_request_->encodeHeaders(default_response_headers_, false);
  upstream_request_->encodeData(512, true);
  
  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_TRUE(response->complete());
  EXPECT_EQ("200", response->headers().getStatusValue());
}
```

### Benchmark Tests

Benchmark tests measure performance characteristics and identify performance regressions.

**Purpose**: Measure throughput, latency, memory usage, and other performance metrics

**Location**: Throughout test directories, typically named `*_benchmark_test.cc`

**Characteristics**:
- Uses Google Benchmark framework
- Runs with optimizations enabled (`-c opt`)
- Reports timing and iteration counts
- Designed for repeatability
- Can skip expensive benchmarks in CI

**Example**:
```cpp
static void BM_BufferCreate(benchmark::State& state) {
  for (auto _ : state) {
    Buffer::OwnedImpl buffer;
    benchmark::DoNotOptimize(buffer);
  }
}
BENCHMARK(BM_BufferCreate);
```

### Fuzz Tests

Fuzz tests use automated input generation to find crashes, hangs, and security issues.

**Purpose**: Find bugs through random/mutated inputs, discover edge cases, improve robustness

**Location**: `test/fuzz/`, `test/common/`, `test/extensions/`

**Characteristics**:
- Uses libFuzzer or libprotobuf-mutator
- Continuous fuzzing via OSS-Fuzz
- Requires corpus of valid inputs
- Finds crashes and undefined behavior
- Can use protobuf-based inputs

**Example**:
```cpp
DEFINE_FUZZER(const uint8_t* data, size_t size) {
  std::string input(reinterpret_cast<const char*>(data), size);
  try {
    Base64::decode(input);
  } catch (const EnvoyException& e) {
    // Expected for invalid input
  }
}
```

## Test Infrastructure

### Google Test Framework

Envoy uses [Google Test](https://github.com/google/googletest) (gtest) as its primary testing framework.

**Key Features**:
- `TEST()` macro for simple tests
- `TEST_F()` macro for fixture-based tests
- `EXPECT_*` and `ASSERT_*` macros for assertions
- Test parameterization with `TEST_P()`
- Death tests for testing fatal errors

**Basic Structure**:
```cpp
TEST(TestSuiteName, TestName) {
  EXPECT_EQ(4, 2 + 2);
  EXPECT_TRUE(condition);
  ASSERT_NE(nullptr, pointer);  // Stops test if fails
}
```

### Google Mock Framework

Envoy uses [Google Mock](https://github.com/google/googletest/blob/master/googlemock/README.md) (gmock) for creating mock objects.

**Key Features**:
- `MOCK_METHOD()` for defining mock methods
- `EXPECT_CALL()` for setting expectations
- `NiceMock<>` and `StrictMock<>` for controlling behavior
- Matchers for flexible argument matching
- Actions for controlling return values

**Basic Structure**:
```cpp
class MockConnection : public Connection {
public:
  MOCK_METHOD(void, write, (Buffer::Instance& data, bool end_stream));
  MOCK_METHOD(void, close, (ConnectionCloseType type));
};

TEST(MyTest, CallsClose) {
  MockConnection connection;
  EXPECT_CALL(connection, close(ConnectionCloseType::FlushWrite));
  
  MyComponent component(connection);
  component.shutdown();
}
```

### Mock vs NiceMock vs StrictMock

**Mock (default)**:
- Warns on unexpected calls
- Allows unexpected calls to proceed
- Shows warnings in test output

**NiceMock**:
- Silent on unexpected calls
- Allows unexpected calls to proceed
- Use when you don't care about specific calls
- Most common in Envoy tests

**StrictMock**:
- Fails test on unexpected calls
- Strict verification of all calls
- Use when exact call sequence matters

```cpp
// Default mock - warns on unexpected calls
MockConnection connection;

// Nice mock - no warnings
NiceMock<MockConnection> nice_connection;

// Strict mock - fails on unexpected calls
StrictMock<MockConnection> strict_connection;
```

## Test Utilities

### test_common/ Directory

Contains shared test utilities used across all tests:

**Key Utilities**:

1. **utility.h/cc**: General test helpers
   - `TestUtility::loadFromYaml()`: Load protobuf from YAML
   - `TestUtility::findCounter()`: Find stats by name
   - `TestUtility::waitForCounterGe()`: Wait for counter value
   - Custom matchers (ProtoEq, ContainsHeader, etc.)

2. **test_time_system.h**: Time management
   - `Event::SimulatedTimeSystem`: Simulated time (recommended)
   - `Event::TestRealTimeSystem`: Real time when needed
   - Deterministic time for reliable tests

3. **environment.h**: Test environment setup
   - Path helpers
   - Temporary directory management
   - Environment variable handling

4. **printers.h**: Custom printers for gtest
   - Pretty printing for Envoy types
   - Better test failure messages

5. **network_utility.h**: Network test helpers
   - Address creation
   - Socket helpers
   - Connection utilities

### test/mocks/ Directory

Contains pre-built mock implementations for common Envoy interfaces:

**Common Mocks**:
- `test/mocks/http/mocks.h`: HTTP mocks (stream, connection, filter callbacks)
- `test/mocks/network/mocks.h`: Network mocks (connection, filter, listener)
- `test/mocks/server/mocks.h`: Server mocks (factory context, admin, etc.)
- `test/mocks/upstream/mocks.h`: Upstream mocks (cluster, host, load balancer)
- `test/mocks/event/mocks.h`: Event mocks (dispatcher, timer, etc.)

**Example Usage**:
```cpp
#include "test/mocks/http/mocks.h"

TEST(FilterTest, BasicFlow) {
  NiceMock<Http::MockStreamDecoderFilterCallbacks> callbacks;
  MyFilter filter(callbacks);
  
  Http::TestRequestHeaderMapImpl headers{{":method", "GET"}};
  EXPECT_EQ(Http::FilterHeadersStatus::Continue, 
            filter.decodeHeaders(headers, true));
}
```

## Time Management in Tests

Proper time management is crucial for reliable, deterministic tests.

### The Problem

Production Envoy uses real time for:
- Timeouts
- Rate limiting  
- Metrics and timestamps
- Delayed operations

Tests using real time can be:
- Flaky (timing-dependent)
- Slow (waiting for timeouts)
- Non-deterministic

### The Solution: TestTimeSystem

All time operations in Envoy go through `Event::TimeSystem`, which has test implementations:

**SimulatedTimeSystem** (Recommended):
```cpp
Event::SimulatedTimeSystem time_system;

// Instantly advance time
time_system.advanceTimeWait(std::chrono::seconds(10));

// Tests run fast and deterministically
```

**TestRealTimeSystem** (When needed):
```cpp
Event::TestRealTimeSystem time_system;

// Uses actual wall-clock time
// Needed for tests involving real I/O timing
```

**GlobalTimeSystem** (In shared infrastructure):
```cpp
// Automatically uses the active time system
// Lazy-initializes if none exists
```

### Usage Pattern

```cpp
TEST(MyTest, Timeout) {
  Event::SimulatedTimeSystem time_system;
  
  MyComponent component(time_system);
  component.startTimer(std::chrono::seconds(30));
  
  // Instantly trigger timeout
  time_system.advanceTimeWait(std::chrono::seconds(30));
  
  EXPECT_TRUE(component.timedOut());
}
```

## Custom Matchers

Envoy provides custom Google Mock matchers for common patterns:

### ContainsHeader
Tests that a HeaderMap contains a specific header:
```cpp
EXPECT_THAT(response->headers(), 
            ContainsHeader(Http::Headers::get().Server, "envoy"));
```

### HttpStatusIs
Tests HTTP status code:
```cpp
EXPECT_THAT(response->headers(), HttpStatusIs("200"));
EXPECT_THAT(response->headers(), HttpStatusIs(200));
```

### ProtoEq
Tests protobuf equality:
```cpp
envoy::config::v3::Config expected;
EXPECT_CALL(callback, onConfig(ProtoEq(expected)));
```

### ProtoEqIgnoringField
Tests protobuf equality ignoring specific fields:
```cpp
EXPECT_CALL(stream, sendMessage(
    ProtoEqIgnoringField(expected_request, "response_nonce"), false));
```

### HeaderMapEqualRef
Tests HeaderMap equality:
```cpp
EXPECT_THAT(response->headers(), HeaderMapEqualRef(expected_headers));
```

## Test Organization

### Directory Structure

```
test/
├── common/                  # Unit tests for source/common/
│   ├── buffer/
│   ├── http/
│   ├── network/
│   └── ...
├── extensions/              # Tests for extensions
│   ├── filters/
│   │   ├── http/
│   │   └── network/
│   └── ...
├── integration/            # Integration tests
│   ├── integration.h       # Base integration test class
│   ├── http_integration.h  # HTTP integration tests
│   └── *_integration_test.cc
├── mocks/                  # Mock implementations
│   ├── http/
│   ├── network/
│   └── ...
├── test_common/           # Test utilities
│   ├── utility.h
│   ├── test_time_system.h
│   └── ...
├── benchmark/             # Benchmark infrastructure
└── fuzz/                  # Fuzz test infrastructure
```

### Naming Conventions

- **Unit tests**: `*_test.cc` (e.g., `buffer_impl_test.cc`)
- **Integration tests**: `*_integration_test.cc`
- **Benchmark tests**: `*_benchmark_test.cc` or `*_speed_test.cc`
- **Fuzz tests**: `*_fuzz_test.cc`
- **Test fixtures**: Match source file name (e.g., `BufferImplTest` for `buffer_impl.cc`)

## Running Tests

### Basic Commands

```bash
# Run all tests
bazel test //test/...

# Run specific test
bazel test //test/common/buffer:buffer_test

# Run with specific gtest filter
bazel test //test/common/buffer:buffer_test --test_arg=--gtest_filter="BufferTest.*"

# Run with debug logging
bazel test //test/integration:integration_test --test_arg="-l debug"

# Run with trace logging
bazel test //test/integration:integration_test --test_arg="-l trace"
```

### Integration Test Commands

```bash
# Run integration test with filter
bazel test //test/integration:http2_upstream_integration_test \
  --test_arg=--gtest_filter="*RouterRequestAndResponseWithBodyNoBuffer*"

# Run repeatedly to catch flakes
bazel test //test/integration:integration_test --runs_per_test=100

# Run with more parallelism
bazel test //test/integration:integration_test \
  --jobs=60 --local_test_jobs=60
```

### Benchmark Commands

```bash
# Run benchmark with optimizations
bazel run -c opt //test/common/buffer:buffer_speed_test

# Run benchmark with limited iterations (for CI)
bazel test //test/common/buffer:buffer_benchmark_test
```

### Fuzz Test Commands

```bash
# Run against corpus only (CI mode)
bazel test //test/common/common:base64_fuzz_test

# Run with fuzzer (local fuzzing)
bazel run //test/common/common:base64_fuzz_test --config=asan-fuzzer \
  -- test/common/common/base64_corpus -runs=10000
```

## Test Best Practices

1. **Keep tests focused**: Each test should verify one specific behavior
2. **Use descriptive names**: Test names should explain what's being tested
3. **Test both success and failure**: Cover happy path and error cases
4. **Make tests deterministic**: Use SimulatedTimeSystem, avoid real I/O
5. **Keep tests fast**: Unit tests in milliseconds, integration tests in seconds
6. **Use appropriate test type**: Unit for logic, integration for end-to-end
7. **Clean up resources**: Use RAII and proper teardown
8. **Document complex tests**: Add comments for non-obvious test logic
9. **Use helper functions**: Extract common patterns to reduce duplication
10. **Test at the right level**: Don't write integration tests for unit-testable logic

## Common Testing Patterns

### Testing Exceptions

```cpp
// Exact message match
EXPECT_THROW_WITH_MESSAGE(
    dangerous_operation(),
    EnvoyException,
    "Expected error message");

// Regex match
EXPECT_THROW_WITH_REGEX(
    dangerous_operation(),
    EnvoyException,
    "error.*occurred");
```

### Testing with Protobufs

```cpp
// Load from YAML
const std::string yaml = R"EOF(
  name: my_config
  value: 123
)EOF";
envoy::config::v3::MyConfig config;
TestUtility::loadFromYaml(yaml, config);
```

### Parameterized Tests

```cpp
class ParameterizedTest : public testing::TestWithParam<int> {};

TEST_P(ParameterizedTest, WorksWithParameter) {
  int param = GetParam();
  EXPECT_GT(param, 0);
}

INSTANTIATE_TEST_SUITE_P(Values, ParameterizedTest, 
                         testing::Values(1, 2, 3));
```

## Next Steps

- [Unit Testing Guide](02-unit-testing.md): Deep dive into writing unit tests
- [Integration Testing Guide](03-integration-testing.md): Learn integration testing patterns
- [Filter Testing Guide](04-filter-testing.md): Specific patterns for filter tests
- [Benchmarking Guide](05-benchmarking.md): Performance testing techniques
