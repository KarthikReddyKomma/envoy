# Unit Testing Guide

This guide covers unit testing patterns in Envoy, including GoogleTest framework usage, mock objects, test fixtures, and common patterns found throughout the codebase.

## Table of Contents

- [GoogleTest Basics](#googletest-basics)
- [Test Fixtures](#test-fixtures)
- [Mock Objects](#mock-objects)
- [Common Mock Classes](#common-mock-classes)
- [Testing Patterns](#testing-patterns)
- [Examples](#examples)

## GoogleTest Basics

Envoy uses Google Test (gtest) as its testing framework.

### Simple Tests

```cpp
#include "gtest/gtest.h"

TEST(ComponentTest, BasicBehavior) {
  MyComponent component;
  EXPECT_EQ(42, component.getValue());
  EXPECT_TRUE(component.isValid());
}
```

### Test Macros

**EXPECT vs ASSERT**:
- `EXPECT_*`: Test continues after failure (preferred for most cases)
- `ASSERT_*`: Test stops after failure (use when continuation doesn't make sense)

```cpp
TEST(BufferTest, Operations) {
  Buffer::OwnedImpl buffer;
  
  // Test continues even if this fails
  EXPECT_EQ(0, buffer.length());
  
  buffer.add("hello");
  
  // Test stops if this fails (buffer would be invalid)
  ASSERT_NE(nullptr, buffer.linearize(5));
  
  // This won't run if ASSERT_NE fails
  EXPECT_STREQ("hello", buffer.toString().c_str());
}
```

### Common Assertions

```cpp
// Equality
EXPECT_EQ(expected, actual);
EXPECT_NE(a, b);

// Boolean
EXPECT_TRUE(condition);
EXPECT_FALSE(condition);

// Comparison
EXPECT_LT(a, b);  // Less than
EXPECT_LE(a, b);  // Less or equal
EXPECT_GT(a, b);  // Greater than
EXPECT_GE(a, b);  // Greater or equal

// Floating point
EXPECT_FLOAT_EQ(expected, actual);
EXPECT_DOUBLE_EQ(expected, actual);

// Strings
EXPECT_STREQ("expected", actual.c_str());
EXPECT_STRCASEEQ("Expected", "expected");

// Pointers
EXPECT_EQ(nullptr, ptr);
EXPECT_NE(nullptr, ptr);
```

### Using Matchers

Google Mock provides powerful matchers for more expressive assertions:

```cpp
#include "gmock/gmock.h"

using testing::HasSubstr;
using testing::StartsWith;
using testing::EndsWith;
using testing::MatchesRegex;

TEST(StringTest, Matchers) {
  std::string value = "Hello, World!";
  
  EXPECT_THAT(value, HasSubstr("World"));
  EXPECT_THAT(value, StartsWith("Hello"));
  EXPECT_THAT(value, EndsWith("!"));
  EXPECT_THAT(value, MatchesRegex("Hello.*World"));
}
```

## Test Fixtures

Test fixtures provide reusable setup and teardown code for related tests.

### Basic Fixture

```cpp
class BufferTest : public testing::Test {
protected:
  void SetUp() override {
    // Runs before each test
    buffer_.add("initial data");
  }
  
  void TearDown() override {
    // Runs after each test
    buffer_.drain(buffer_.length());
  }
  
  // Shared test state
  Buffer::OwnedImpl buffer_;
};

TEST_F(BufferTest, AddData) {
  // buffer_ already contains "initial data"
  buffer_.add(" more data");
  EXPECT_EQ(18, buffer_.length());
}

TEST_F(BufferTest, DrainData) {
  buffer_.drain(5);
  EXPECT_EQ(7, buffer_.length());
}
```

### Fixture with Helper Methods

```cpp
class HttpFilterTest : public testing::Test {
public:
  HttpFilterTest() : filter_(config_) {
    filter_.setDecoderFilterCallbacks(callbacks_);
  }
  
protected:
  // Helper to create headers
  Http::TestRequestHeaderMapImpl createHeaders(
      const std::string& path = "/test") {
    return Http::TestRequestHeaderMapImpl{
        {":method", "GET"},
        {":path", path},
        {":scheme", "http"},
        {":authority", "host"}};
  }
  
  // Helper to create body data
  Buffer::OwnedImpl createBody(const std::string& content) {
    return Buffer::OwnedImpl(content);
  }
  
  // Shared state
  NiceMock<Http::MockStreamDecoderFilterCallbacks> callbacks_;
  MyFilterConfigSharedPtr config_;
  MyFilter filter_;
};

TEST_F(HttpFilterTest, PassthroughRequest) {
  auto headers = createHeaders("/api/endpoint");
  EXPECT_EQ(Http::FilterHeadersStatus::Continue, 
            filter_.decodeHeaders(headers, true));
}
```

## Mock Objects

### Creating Mocks

Google Mock allows creating mock implementations of interfaces:

```cpp
class Connection {
public:
  virtual ~Connection() = default;
  virtual void write(Buffer::Instance& data, bool end_stream) = 0;
  virtual void close(ConnectionCloseType type) = 0;
  virtual bool connected() const = 0;
};

class MockConnection : public Connection {
public:
  MOCK_METHOD(void, write, (Buffer::Instance& data, bool end_stream));
  MOCK_METHOD(void, close, (ConnectionCloseType type));
  MOCK_METHOD(bool, connected, (), (const));
};
```

### Setting Expectations

```cpp
TEST(NetworkTest, WriteData) {
  MockConnection connection;
  
  // Expect write to be called once with any arguments
  EXPECT_CALL(connection, write(_, _));
  
  // Expect close to be called with specific argument
  EXPECT_CALL(connection, close(ConnectionCloseType::FlushWrite));
  
  // Code under test
  NetworkHandler handler(connection);
  handler.sendAndClose();
}
```

### NiceMock vs StrictMock

**NiceMock** (Most Common):
- Silently allows unexpected calls
- Use when you don't care about all method calls
- Reduces test brittleness

```cpp
TEST(FilterTest, BasicFlow) {
  NiceMock<Http::MockStreamDecoderFilterCallbacks> callbacks;
  // Won't fail if unexpected methods are called
  MyFilter filter(callbacks);
  filter.processRequest();
}
```

**StrictMock**:
- Fails on any unexpected call
- Use when exact call sequence matters
- More strict verification

```cpp
TEST(ProtocolTest, ExactSequence) {
  StrictMock<MockConnection> connection;
  
  // Must call exactly these methods
  EXPECT_CALL(connection, write(_, _));
  EXPECT_CALL(connection, close(_));
  
  // Will fail if any other method is called
  Protocol protocol(connection);
  protocol.execute();
}
```

### Return Values and Actions

```cpp
TEST(LookupTest, ReturnValues) {
  MockDatabase db;
  
  // Return a specific value
  EXPECT_CALL(db, getValue("key"))
      .WillOnce(Return("value"));
  
  // Return different values on repeated calls
  EXPECT_CALL(db, getCount())
      .WillOnce(Return(1))
      .WillOnce(Return(2))
      .WillRepeatedly(Return(3));
  
  // Execute custom action
  EXPECT_CALL(db, query(_))
      .WillOnce(Invoke([](const std::string& q) {
        return processQuery(q);
      }));
}
```

### Argument Matchers

```cpp
using testing::_;
using testing::Eq;
using testing::Gt;
using testing::HasSubstr;

TEST(MatcherTest, ArgumentMatching) {
  MockConnection connection;
  
  // Match any argument
  EXPECT_CALL(connection, write(_, _));
  
  // Match specific value
  EXPECT_CALL(connection, write(_, Eq(true)));
  
  // Match with condition
  EXPECT_CALL(connection, write(_, Gt(100)));
  
  // Match with matcher
  EXPECT_CALL(logger, log(HasSubstr("error")));
}
```

### Saving Arguments

```cpp
TEST(ArgumentTest, SaveForLater) {
  MockCallback callback;
  Buffer::Instance* saved_buffer = nullptr;
  
  EXPECT_CALL(callback, onData(_))
      .WillOnce(SaveArgAddress(&saved_buffer));
  
  // Trigger callback
  handler.processData();
  
  // Verify captured argument
  ASSERT_NE(nullptr, saved_buffer);
  EXPECT_EQ(100, saved_buffer->length());
}
```

## Common Mock Classes

Envoy provides pre-built mocks in `test/mocks/`.

### HTTP Mocks

```cpp
#include "test/mocks/http/mocks.h"

TEST(HttpFilterTest, DecoderCallbacks) {
  // Mock filter callbacks
  NiceMock<Http::MockStreamDecoderFilterCallbacks> callbacks;
  
  // Set up route
  auto route = std::make_shared<NiceMock<Router::MockRoute>>();
  ON_CALL(*callbacks.route_, entry()).WillByDefault(Return(&route->route_entry_));
  
  // Set expectations
  EXPECT_CALL(callbacks, encodeHeaders_(_, _));
  
  MyFilter filter;
  filter.setDecoderFilterCallbacks(callbacks);
}
```

### Network Mocks

```cpp
#include "test/mocks/network/mocks.h"

TEST(NetworkFilterTest, ReadData) {
  NiceMock<Network::MockReadFilterCallbacks> callbacks;
  NiceMock<Network::MockConnection> connection;
  
  ON_CALL(callbacks, connection()).WillByDefault(ReturnRef(connection));
  
  MyNetworkFilter filter;
  filter.initializeReadFilterCallbacks(callbacks);
  
  Buffer::OwnedImpl data("test data");
  EXPECT_EQ(Network::FilterStatus::Continue, 
            filter.onData(data, false));
}
```

### Server Mocks

```cpp
#include "test/mocks/server/mocks.h"

TEST(ExtensionTest, FactoryContext) {
  NiceMock<Server::Configuration::MockFactoryContext> context;
  
  // Mock dependencies
  EXPECT_CALL(context, scope()).WillRepeatedly(ReturnRef(scope_));
  EXPECT_CALL(context, threadLocal()).WillRepeatedly(ReturnRef(tls_));
  
  MyExtensionFactory factory;
  auto extension = factory.createExtension(config, context);
  EXPECT_NE(nullptr, extension);
}
```

### Event Mocks

```cpp
#include "test/mocks/event/mocks.h"

TEST(TimerTest, Schedule) {
  NiceMock<Event::MockDispatcher> dispatcher;
  Event::MockTimer* timer = new NiceMock<Event::MockTimer>();
  
  EXPECT_CALL(dispatcher, createTimer_(_))
      .WillOnce(Invoke([&](Event::TimerCb cb) {
        return timer;
      }));
  
  EXPECT_CALL(*timer, enableTimer(std::chrono::seconds(10), _));
  
  MyComponent component(dispatcher);
  component.startTimer();
}
```

## Testing Patterns

### Testing with Protobufs

```cpp
#include "test/test_common/utility.h"

TEST(ConfigTest, LoadFromYaml) {
  const std::string yaml = R"EOF(
    name: test_config
    timeout: 30s
    retry_policy:
      num_retries: 3
  )EOF";
  
  envoy::config::v3::MyConfig config;
  TestUtility::loadFromYaml(yaml, config);
  
  EXPECT_EQ("test_config", config.name());
  EXPECT_EQ(30, config.timeout().seconds());
  EXPECT_EQ(3, config.retry_policy().num_retries());
}
```

### Testing Exception Handling

```cpp
TEST(ErrorHandlingTest, ThrowsOnInvalidInput) {
  // Expect exact message
  EXPECT_THROW_WITH_MESSAGE(
      component.process("invalid"),
      EnvoyException,
      "Invalid input: invalid");
  
  // Expect message with regex
  EXPECT_THROW_WITH_REGEX(
      component.process("bad"),
      EnvoyException,
      "Invalid.*bad");
}
```

### Testing with Simulated Time

```cpp
#include "test/test_common/simulated_time_system.h"

TEST(TimeoutTest, ExpiresAfterDelay) {
  Event::SimulatedTimeSystem time_system;
  
  MyComponent component(time_system);
  component.startTimer(std::chrono::seconds(30));
  
  // Not expired yet
  EXPECT_FALSE(component.hasExpired());
  
  // Advance time
  time_system.advanceTimeWait(std::chrono::seconds(30));
  
  // Now expired
  EXPECT_TRUE(component.hasExpired());
}
```

### Testing Callbacks

```cpp
TEST(CallbackTest, InvokesCallback) {
  bool callback_invoked = false;
  std::string received_data;
  
  auto callback = [&](const std::string& data) {
    callback_invoked = true;
    received_data = data;
  };
  
  MyComponent component;
  component.onData(callback);
  component.processData("test data");
  
  EXPECT_TRUE(callback_invoked);
  EXPECT_EQ("test data", received_data);
}
```

### Parameterized Tests

```cpp
class ProtocolTest : public testing::TestWithParam<Http::CodecType> {
protected:
  Http::CodecType codecType() { return GetParam(); }
};

TEST_P(ProtocolTest, HandlesRequest) {
  auto codec = createCodec(codecType());
  
  auto result = codec->encodeHeaders(headers_, true);
  EXPECT_TRUE(result.ok());
}

INSTANTIATE_TEST_SUITE_P(
    Protocols, ProtocolTest,
    testing::Values(
        Http::CodecType::HTTP1,
        Http::CodecType::HTTP2,
        Http::CodecType::HTTP3));
```

## Examples

### Example 1: Buffer Filter Test

From `test/extensions/filters/http/buffer/buffer_filter_test.cc`:

```cpp
class BufferFilterTest : public testing::Test {
public:
  BufferFilterConfigSharedPtr setupConfig() {
    envoy::extensions::filters::http::buffer::v3::Buffer proto_config;
    proto_config.mutable_max_request_bytes()->set_value(1024 * 1024);
    return std::make_shared<BufferFilterConfig>(proto_config);
  }

  BufferFilterTest() : config_(setupConfig()), filter_(config_) {
    filter_.setDecoderFilterCallbacks(callbacks_);
  }

  NiceMock<Http::MockStreamDecoderFilterCallbacks> callbacks_;
  BufferFilterConfigSharedPtr config_;
  BufferFilter filter_;
};

TEST_F(BufferFilterTest, HeaderOnlyRequest) {
  Http::TestRequestHeaderMapImpl headers;
  EXPECT_EQ(Http::FilterHeadersStatus::Continue, 
            filter_.decodeHeaders(headers, true));
}

TEST_F(BufferFilterTest, RequestWithData) {
  InSequence s;

  Http::TestRequestHeaderMapImpl headers;
  EXPECT_EQ(Http::FilterHeadersStatus::StopIteration, 
            filter_.decodeHeaders(headers, false));

  Buffer::OwnedImpl data1("hello");
  EXPECT_EQ(Http::FilterDataStatus::StopIterationAndBuffer, 
            filter_.decodeData(data1, false));

  Buffer::OwnedImpl data2(" world");
  EXPECT_EQ(Http::FilterDataStatus::Continue, 
            filter_.decodeData(data2, true));
}

TEST_F(BufferFilterTest, ContentLengthPopulation) {
  InSequence s;

  Http::TestRequestHeaderMapImpl headers;
  EXPECT_EQ(Http::FilterHeadersStatus::StopIteration, 
            filter_.decodeHeaders(headers, false));

  Buffer::OwnedImpl data1("hello");
  EXPECT_EQ(Http::FilterDataStatus::StopIterationAndBuffer, 
            filter_.decodeData(data1, false));

  Buffer::OwnedImpl data2(" world");
  EXPECT_EQ(Http::FilterDataStatus::Continue, 
            filter_.decodeData(data2, true));
  
  EXPECT_EQ(headers.getContentLengthValue(), "11");
}
```

### Example 2: File Event Test

From `test/common/event/file_event_impl_test.cc`:

```cpp
class FileEventImplTest : public testing::Test {
public:
  FileEventImplTest() 
      : api_(Api::createApiForTest()),
        dispatcher_(api_->allocateDispatcher("test_thread")),
        os_sys_calls_(Api::OsSysCallsSingleton::get()) {}

  void SetUp() override {
    ASSERT_EQ(0, os_sys_calls_.socketpair(
        AF_UNIX, SOCK_DGRAM, 0, fds_).return_value_);
    
    int data = 1;
    const Api::SysCallSizeResult result = 
        os_sys_calls_.write(fds_[1], &data, sizeof(data));
    ASSERT_EQ(sizeof(data), static_cast<size_t>(result.return_value_));
  }

  void TearDown() override {
    os_sys_calls_.close(fds_[0]);
    os_sys_calls_.close(fds_[1]);
  }

protected:
  os_fd_t fds_[2];
  Api::ApiPtr api_;
  DispatcherPtr dispatcher_;
  Api::OsSysCalls& os_sys_calls_;
};

TEST_F(FileEventImplTest, EdgeTrigger) {
  testing::InSequence s;
  MockFunction<void()> read_callback;
  
  EXPECT_CALL(read_callback, Call()).Times(1);
  
  Event::FileEventPtr file_event = dispatcher_->createFileEvent(
      fds_[0],
      [&](uint32_t events) {
        if (events & Event::FileReadyType::Read) {
          read_callback.Call();
        }
        return absl::OkStatus();
      },
      Event::FileTriggerType::Edge,
      Event::FileReadyType::Read);
  
  dispatcher_->run(Event::Dispatcher::RunType::NonBlock);
}
```

### Example 3: Testing with Mock Factory Context

```cpp
#include "test/mocks/server/factory_context.h"

class MyExtensionTest : public testing::Test {
protected:
  void setupFactoryContext() {
    EXPECT_CALL(factory_context_, scope())
        .WillRepeatedly(ReturnRef(scope_));
    EXPECT_CALL(factory_context_, threadLocal())
        .WillRepeatedly(ReturnRef(tls_));
    EXPECT_CALL(factory_context_, clusterManager())
        .WillRepeatedly(ReturnRef(cm_));
  }
  
  NiceMock<Server::Configuration::MockFactoryContext> factory_context_;
  NiceMock<Stats::MockIsolatedStatsStore> scope_;
  NiceMock<ThreadLocal::MockInstance> tls_;
  NiceMock<Upstream::MockClusterManager> cm_;
};

TEST_F(MyExtensionTest, CreatesExtension) {
  setupFactoryContext();
  
  const std::string yaml = R"EOF(
    name: my_extension
    config:
      timeout: 10s
  )EOF";
  
  envoy::config::v3::ExtensionConfig config;
  TestUtility::loadFromYaml(yaml, config);
  
  MyExtensionFactory factory;
  auto extension = factory.createExtension(config, factory_context_);
  
  ASSERT_NE(nullptr, extension);
  EXPECT_EQ("my_extension", extension->name());
}
```

## Best Practices

1. **Use NiceMock by default**: Reduces test brittleness
2. **Test one thing per test**: Keep tests focused
3. **Use descriptive test names**: Explain what's being tested
4. **Use helper methods**: Reduce duplication in fixtures
5. **Test edge cases**: Not just the happy path
6. **Use InSequence when order matters**: Verify call order
7. **Prefer EXPECT over ASSERT**: Unless later code depends on it
8. **Use custom matchers**: ProtoEq, ContainsHeader, etc.
9. **Mock at boundaries**: Don't mock everything
10. **Keep tests fast**: Unit tests should run in milliseconds

## Common Patterns to Avoid

1. **Over-mocking**: Don't mock simple value objects
2. **Brittle expectations**: Don't specify every single call
3. **Testing implementation details**: Test behavior, not internals
4. **Complex test setup**: If setup is complex, the code might be too coupled
5. **Ignoring test failures**: Fix or understand every failure

## Next Steps

- [Integration Testing Guide](03-integration-testing.md): Learn end-to-end testing
- [Filter Testing Guide](04-filter-testing.md): Specific patterns for filters
- [Test Common Utilities](../../test/test_common/): Explore test utilities
- [Mock Implementations](../../test/mocks/): Browse available mocks
