# Envoy Testing Documentation

This directory contains comprehensive documentation for testing Envoy, covering the testing philosophy, frameworks, patterns, and best practices used throughout the codebase.

## Quick Start

If you're new to Envoy testing:

1. Start with [Testing Overview](01-testing-overview.md) to understand Envoy's testing philosophy and infrastructure
2. Read [Unit Testing](02-unit-testing.md) to learn how to write unit tests
3. Review [Integration Testing](03-integration-testing.md) for end-to-end testing patterns
4. Check [Filter Testing](04-filter-testing.md) when implementing filters
5. See [Benchmarking](05-benchmarking.md) for performance testing

## Documentation Structure

### [01-testing-overview.md](01-testing-overview.md)
Overview of Envoy's comprehensive testing strategy:
- Testing philosophy and principles
- Test types (unit, integration, benchmark, fuzz)
- Test infrastructure and tooling
- Google Test and Google Mock frameworks
- Test utilities and helpers

### [02-unit-testing.md](02-unit-testing.md)
Detailed guide to writing unit tests:
- GoogleTest framework basics
- Mock objects (NiceMock, StrictMock)
- Test fixtures and setup patterns
- Common mock classes and utilities
- Real-world examples from test/common/

### [03-integration-testing.md](03-integration-testing.md)
Integration testing framework and patterns:
- BaseIntegrationTest class hierarchy
- HttpIntegrationTest patterns
- Fake upstream usage
- Configuration helpers
- Testing downstream-Envoy-upstream flows

### [04-filter-testing.md](04-filter-testing.md)
Filter-specific testing patterns:
- HTTP filter testing
- Network filter testing
- Listener filter testing
- Mock callbacks and contexts
- Filter-specific test patterns

### [05-benchmarking.md](05-benchmarking.md)
Performance and benchmark testing:
- Google Benchmark framework
- Writing benchmark tests
- Benchmark best practices
- Performance measurement patterns
- CI integration

## Test Locations

```
test/
├── common/              # Unit tests for common/ source code
├── integration/         # Integration tests
├── mocks/              # Mock implementations
├── test_common/        # Test utilities and helpers
├── extensions/         # Extension-specific tests
├── benchmark/          # Benchmark tests
└── fuzz/               # Fuzz tests
```

## Running Tests

### Run all tests
```bash
bazel test //test/...
```

### Run specific test
```bash
bazel test //test/common/buffer:buffer_test
```

### Run with debug logging
```bash
bazel test //test/integration:integration_test --test_arg="-l debug"
```

### Run integration test with filter
```bash
bazel test //test/integration:http2_upstream_integration_test \
  --test_arg=--gtest_filter="IpVersions/Http2UpstreamIntegrationTest.RouterRequestAndResponseWithBodyNoBuffer/IPv6"
```

### Run benchmark test
```bash
bazel run -c opt //test/common/buffer:buffer_speed_test
```

### Run fuzz test
```bash
bazel run //test/common/common:base64_fuzz_test --config asan-fuzzer \
  -- test/common/common/base64_corpus -runs=1000
```

## Key Testing Concepts

### Test Types

1. **Unit Tests**: Test individual classes and functions in isolation
2. **Integration Tests**: Test downstream-Envoy-upstream communication flows
3. **Benchmark Tests**: Measure performance characteristics
4. **Fuzz Tests**: Find bugs through automated input generation

### Mock Objects

Envoy uses Google Mock extensively:
- `NiceMock<>`: Mock that allows unexpected calls (doesn't fail on them)
- `StrictMock<>`: Mock that fails on any unexpected call
- `test/mocks/`: Pre-built mock implementations for common interfaces

### Test Fixtures

Test fixtures provide reusable test setup:
```cpp
class MyComponentTest : public testing::Test {
public:
  void SetUp() override {
    // Common setup code
  }
  
  void TearDown() override {
    // Common cleanup code
  }
  
  // Shared test state
  NiceMock<MockDependency> dependency_;
  MyComponent component_{dependency_};
};
```

### Time Management

Use `Event::TestTimeSystem` for deterministic time:
- `Event::SimulatedTimeSystem`: Simulated time (recommended)
- `Event::TestRealTimeSystem`: Real-time for specific cases
- Avoids flaky tests and makes tests faster

## Best Practices

1. **Prefer unit tests** for testing logic and edge cases
2. **Use integration tests** for validating end-to-end behavior
3. **Use NiceMock by default**, StrictMock when you need strict verification
4. **Use SimulatedTimeSystem** to avoid flaky tests
5. **Test both success and failure paths**
6. **Use helper utilities** from test_common/ to reduce boilerplate
7. **Keep tests focused and fast**
8. **Use descriptive test names** that explain what's being tested

## Common Patterns

### Testing with Protobufs
```cpp
envoy::config::v3::MyConfig config;
TestUtility::loadFromYaml(yaml_string, config);
// Test with config
```

### Waiting for Conditions in Integration Tests
```cpp
response->waitForEndStream();
test_server_->waitForCounterGe("cluster.cluster_0.upstream_cx_total", 1);
```

### Testing Exceptions
```cpp
EXPECT_THROW_WITH_MESSAGE(
  dangerousOperation(),
  EnvoyException,
  "Expected error message"
);
```

### Custom Matchers
Envoy provides custom matchers for common cases:
- `ContainsHeader`: Check HeaderMap contains specific header
- `HttpStatusIs`: Verify HTTP status code
- `ProtoEq`: Compare protobuf messages

## Resources

- [Google Test Documentation](https://github.com/google/googletest)
- [Google Mock Documentation](https://github.com/google/googletest/blob/master/googlemock/README.md)
- [Google Benchmark Documentation](https://github.com/google/benchmark)
- [Integration Test README](../../test/integration/README.md)
- [Fuzz Test README](../../test/fuzz/README.md)
- [Main Test README](../../test/README.md)

## Contributing

When adding new tests:
1. Place unit tests alongside the source code they test
2. Add integration tests to test/integration/
3. Create or update mock objects in test/mocks/ when needed
4. Add test utilities to test_common/ if they're reusable
5. Document complex test patterns
6. Ensure tests are deterministic and fast

## Questions?

If you have questions about testing in Envoy:
1. Check these documentation files
2. Look at similar existing tests for patterns
3. Review the test/README.md
4. Ask in the Envoy community channels
