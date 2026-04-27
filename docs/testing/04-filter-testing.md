# Filter Testing Guide

This guide covers testing patterns specific to Envoy filters, including HTTP filters, network filters, and listener filters.

## Table of Contents

- [HTTP Filter Testing](#http-filter-testing)
- [Network Filter Testing](#network-filter-testing)
- [Listener Filter Testing](#listener-filter-testing)
- [Filter Factory Testing](#filter-factory-testing)
- [Integration Testing for Filters](#integration-testing-for-filters)

## HTTP Filter Testing

HTTP filters process HTTP requests and responses. They implement `StreamDecoderFilter`, `StreamEncoderFilter`, or both.

### Basic HTTP Filter Test Structure

```cpp
#include "test/mocks/http/mocks.h"
#include "test/mocks/server/factory_context.h"

class MyHttpFilterTest : public testing::Test {
public:
  MyHttpFilterTest() {
    // Create filter configuration
    setupConfig();
    
    // Create filter instance
    filter_ = std::make_unique<MyHttpFilter>(config_);
    
    // Set up filter callbacks
    filter_->setDecoderFilterCallbacks(decoder_callbacks_);
    filter_->setEncoderFilterCallbacks(encoder_callbacks_);
  }

protected:
  void setupConfig() {
    const std::string yaml = R"EOF(
      my_setting: value
      timeout: 30s
    )EOF";
    
    envoy::extensions::filters::http::my_filter::v3::MyFilter proto_config;
    TestUtility::loadFromYaml(yaml, proto_config);
    config_ = std::make_shared<MyHttpFilterConfig>(proto_config);
  }

  MyHttpFilterConfigSharedPtr config_;
  std::unique_ptr<MyHttpFilter> filter_;
  NiceMock<Http::MockStreamDecoderFilterCallbacks> decoder_callbacks_;
  NiceMock<Http::MockStreamEncoderFilterCallbacks> encoder_callbacks_;
};
```

### Testing Request Processing (Decoder)

```cpp
TEST_F(MyHttpFilterTest, PassthroughRequest) {
  // Create request headers
  Http::TestRequestHeaderMapImpl headers{
      {":method", "GET"},
      {":path", "/api/v1/resource"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  // Process headers
  EXPECT_EQ(Http::FilterHeadersStatus::Continue,
            filter_->decodeHeaders(headers, false));

  // Process body
  Buffer::OwnedImpl data("request body");
  EXPECT_EQ(Http::FilterDataStatus::Continue,
            filter_->decodeData(data, false));

  // Process trailers
  Http::TestRequestTrailerMapImpl trailers{{"trailer-key", "value"}};
  EXPECT_EQ(Http::FilterTrailersStatus::Continue,
            filter_->decodeTrailers(trailers));
}
```

### Testing Response Processing (Encoder)

```cpp
TEST_F(MyHttpFilterTest, ProcessResponse) {
  // Create response headers
  Http::TestResponseHeaderMapImpl headers{
      {":status", "200"},
      {"content-type", "application/json"}};

  // Process headers
  EXPECT_EQ(Http::FilterHeadersStatus::Continue,
            filter_->encodeHeaders(headers, false));

  // Process body
  Buffer::OwnedImpl data("{\"key\":\"value\"}");
  EXPECT_EQ(Http::FilterDataStatus::Continue,
            filter_->encodeData(data, true));
}
```

### Testing Filter Stops Iteration

```cpp
TEST_F(MyHttpFilterTest, StopsIterationForProcessing) {
  Http::TestRequestHeaderMapImpl headers{
      {":method", "POST"},
      {":path", "/api/process"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  // Filter stops iteration to buffer request
  EXPECT_EQ(Http::FilterHeadersStatus::StopIteration,
            filter_->decodeHeaders(headers, false));

  // Send data
  Buffer::OwnedImpl data1("chunk1");
  EXPECT_EQ(Http::FilterDataStatus::StopIterationAndBuffer,
            filter_->decodeData(data1, false));

  Buffer::OwnedImpl data2("chunk2");
  EXPECT_EQ(Http::FilterDataStatus::StopIterationAndBuffer,
            filter_->decodeData(data2, false));

  // End stream - filter processes and continues
  Buffer::OwnedImpl data3("chunk3");
  EXPECT_EQ(Http::FilterDataStatus::Continue,
            filter_->decodeData(data3, true));
}
```

### Testing Header Manipulation

```cpp
TEST_F(MyHttpFilterTest, AddsCustomHeaders) {
  Http::TestRequestHeaderMapImpl headers{
      {":method", "GET"},
      {":path", "/api"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  EXPECT_EQ(Http::FilterHeadersStatus::Continue,
            filter_->decodeHeaders(headers, true));

  // Verify filter added custom header
  EXPECT_EQ("filter-value",
            headers.get_("x-custom-header"));
  
  // Verify filter modified existing header
  EXPECT_EQ("/api/v2",
            headers.getPathValue());
}
```

### Testing with Route Configuration

```cpp
TEST_F(MyHttpFilterTest, UsesRouteConfiguration) {
  // Set up route with per-route filter config
  auto route = std::make_shared<NiceMock<Router::MockRoute>>();
  ON_CALL(*decoder_callbacks_.route_, entry())
      .WillByDefault(Return(&route->route_entry_));

  // Create per-route config
  envoy::extensions::filters::http::my_filter::v3::MyFilterPerRoute 
      per_route_config;
  per_route_config.set_override_value("route-specific");
  
  MyHttpFilterPerRouteConfig route_config(per_route_config);
  ON_CALL(*decoder_callbacks_.route_, 
          mostSpecificPerFilterConfig(_))
      .WillByDefault(Return(&route_config));

  // Filter should use route-specific config
  Http::TestRequestHeaderMapImpl headers{
      {":method", "GET"},
      {":path", "/api"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  EXPECT_EQ(Http::FilterHeadersStatus::Continue,
            filter_->decodeHeaders(headers, true));

  // Verify route config was used
  EXPECT_EQ("route-specific",
            headers.get_("x-configured-value"));
}
```

### Testing Local Replies

```cpp
TEST_F(MyHttpFilterTest, SendsLocalReply) {
  Http::TestRequestHeaderMapImpl headers{
      {":method", "GET"},
      {":path", "/forbidden"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  // Expect local reply to be sent
  EXPECT_CALL(decoder_callbacks_, 
              sendLocalReply(Http::Code::Forbidden, _, _, _, _))
      .WillOnce(Invoke([](Http::Code code, absl::string_view body,
                          std::function<void(Http::ResponseHeaderMap&)> modify,
                          const absl::optional<Grpc::Status::GrpcStatus>,
                          absl::string_view) {
        // Verify reply details
        EXPECT_EQ(Http::Code::Forbidden, code);
        EXPECT_EQ("Access denied", body);
      }));

  EXPECT_EQ(Http::FilterHeadersStatus::StopIteration,
            filter_->decodeHeaders(headers, true));
}
```

### Testing Async Operations

```cpp
TEST_F(MyHttpFilterTest, AsyncOperation) {
  Http::TestRequestHeaderMapImpl headers{
      {":method", "GET"},
      {":path", "/api/async"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  // Filter stops to perform async operation
  EXPECT_EQ(Http::FilterHeadersStatus::StopIteration,
            filter_->decodeHeaders(headers, true));

  // Simulate async completion
  filter_->onAsyncComplete(/* result */);

  // Filter should continue iteration
  EXPECT_CALL(decoder_callbacks_, continueDecoding());
  
  // Trigger continuation
  filter_->completeProcessing();
}
```

### Testing Watermarks

```cpp
TEST_F(MyHttpFilterTest, HandlesBackpressure) {
  Http::TestRequestHeaderMapImpl headers{
      {":method", "POST"},
      {":path", "/api/upload"},
      {":scheme", "https"},
      {":authority", "example.com"}};

  EXPECT_EQ(Http::FilterHeadersStatus::StopIteration,
            filter_->decodeHeaders(headers, false));

  // Simulate high watermark
  filter_->onDecoderFilterAboveWriteBufferHighWatermark();
  
  // Filter should pause processing
  EXPECT_TRUE(filter_->isPaused());

  // Simulate low watermark
  filter_->onDecoderFilterBelowWriteBufferLowWatermark();
  
  // Filter should resume
  EXPECT_FALSE(filter_->isPaused());
}
```

## Network Filter Testing

Network filters operate on raw TCP connections. They implement `ReadFilter`, `WriteFilter`, or both.

### Basic Network Filter Test Structure

```cpp
#include "test/mocks/network/mocks.h"

class MyNetworkFilterTest : public testing::Test {
public:
  MyNetworkFilterTest() {
    setupConfig();
    filter_ = std::make_unique<MyNetworkFilter>(config_);
    filter_->initializeReadFilterCallbacks(read_callbacks_);
  }

protected:
  void setupConfig() {
    const std::string yaml = R"EOF(
      stat_prefix: test
      max_connections: 100
    )EOF";
    
    envoy::extensions::filters::network::my_filter::v3::MyFilter proto_config;
    TestUtility::loadFromYaml(yaml, proto_config);
    config_ = std::make_shared<MyNetworkFilterConfig>(proto_config, context_);
  }

  NiceMock<Server::Configuration::MockFactoryContext> context_;
  MyNetworkFilterConfigSharedPtr config_;
  std::unique_ptr<MyNetworkFilter> filter_;
  NiceMock<Network::MockReadFilterCallbacks> read_callbacks_;
  NiceMock<Network::MockConnection> connection_;
};
```

### Testing Read Operations

```cpp
TEST_F(MyNetworkFilterTest, ProcessesIncomingData) {
  // Set up connection
  ON_CALL(read_callbacks_, connection())
      .WillByDefault(ReturnRef(connection_));

  // Receive data
  Buffer::OwnedImpl data("incoming data");
  EXPECT_EQ(Network::FilterStatus::Continue,
            filter_->onData(data, false));

  // Verify data was processed
  EXPECT_EQ(0, data.length()); // Consumed
}
```

### Testing Write Operations

```cpp
class MyNetworkFilterTest : public testing::Test {
public:
  MyNetworkFilterTest() {
    setupConfig();
    filter_ = std::make_unique<MyNetworkFilter>(config_);
    filter_->initializeReadFilterCallbacks(read_callbacks_);
    filter_->initializeWriteFilterCallbacks(write_callbacks_);
  }

protected:
  NiceMock<Network::MockWriteFilterCallbacks> write_callbacks_;
};

TEST_F(MyNetworkFilterTest, ProcessesOutgoingData) {
  Buffer::OwnedImpl data("outgoing data");
  
  EXPECT_EQ(Network::FilterStatus::Continue,
            filter_->onWrite(data, false));

  // Verify data transformation
  EXPECT_EQ("OUTGOING DATA", data.toString()); // Uppercased
}
```

### Testing Connection Events

```cpp
TEST_F(MyNetworkFilterTest, HandlesConnectionClose) {
  ON_CALL(read_callbacks_, connection())
      .WillByDefault(ReturnRef(connection_));

  // Simulate connection close
  filter_->onEvent(Network::ConnectionEvent::RemoteClose);

  // Verify cleanup
  EXPECT_TRUE(filter_->isClosed());
}

TEST_F(MyNetworkFilterTest, HandlesConnectionFailure) {
  ON_CALL(read_callbacks_, connection())
      .WillByDefault(ReturnRef(connection_));

  // Simulate connection failure
  filter_->onEvent(Network::ConnectionEvent::LocalClose);

  // Verify stats updated
  EXPECT_EQ(1, context_.scope_.counter("test.connection_closed").value());
}
```

### Testing Protocol Detection

```cpp
TEST_F(MyNetworkFilterTest, DetectsProtocol) {
  ON_CALL(read_callbacks_, connection())
      .WillByDefault(ReturnRef(connection_));

  // Receive protocol header
  Buffer::OwnedImpl data("\x16\x03\x01"); // TLS handshake
  
  EXPECT_EQ(Network::FilterStatus::Continue,
            filter_->onData(data, false));

  // Verify protocol detected
  EXPECT_EQ("tls", filter_->detectedProtocol());
}
```

## Listener Filter Testing

Listener filters operate on new connections before they reach network filters.

### Basic Listener Filter Test

```cpp
#include "test/mocks/network/mocks.h"

class MyListenerFilterTest : public testing::Test {
public:
  MyListenerFilterTest() {
    setupConfig();
    filter_ = std::make_unique<MyListenerFilter>(config_);
  }

protected:
  void setupConfig() {
    envoy::extensions::filters::listener::my_filter::v3::MyFilter 
        proto_config;
    config_ = std::make_shared<MyListenerFilterConfig>(proto_config);
  }

  MyListenerFilterConfigSharedPtr config_;
  std::unique_ptr<MyListenerFilter> filter_;
  NiceMock<Network::MockListenerFilterCallbacks> callbacks_;
  NiceMock<Network::MockListenerFilterManager> manager_;
};
```

### Testing Connection Acceptance

```cpp
TEST_F(MyListenerFilterTest, AcceptsConnection) {
  NiceMock<Network::MockConnectionSocket> socket;
  
  ON_CALL(callbacks_, socket())
      .WillByDefault(ReturnRef(socket));

  // Filter accepts connection
  EXPECT_EQ(Network::FilterStatus::Continue,
            filter_->onAccept(callbacks_));

  // Verify connection allowed
  EXPECT_FALSE(filter_->isRejected());
}
```

### Testing Connection Rejection

```cpp
TEST_F(MyListenerFilterTest, RejectsConnection) {
  NiceMock<Network::MockConnectionSocket> socket;
  
  // Set up blacklisted IP
  auto address = Network::Utility::parseInternetAddress("192.168.1.100");
  ON_CALL(socket, connectionInfoProvider())
      .WillByDefault(Return(std::make_shared<Network::ConnectionInfoImpl>(
          address, address)));
  
  ON_CALL(callbacks_, socket())
      .WillByDefault(ReturnRef(socket));

  // Filter rejects connection
  EXPECT_EQ(Network::FilterStatus::StopIteration,
            filter_->onAccept(callbacks_));

  // Verify stats
  EXPECT_EQ(1, config_->stats().connections_rejected_.value());
}
```

## Filter Factory Testing

Filter factories create filter instances from configuration.

### HTTP Filter Factory Test

```cpp
#include "test/mocks/server/factory_context.h"

class MyHttpFilterFactoryTest : public testing::Test {
protected:
  NiceMock<Server::Configuration::MockFactoryContext> context_;
};

TEST_F(MyHttpFilterFactoryTest, CreatesFilter) {
  const std::string yaml = R"EOF(
    my_setting: value
    timeout: 30s
  )EOF";

  envoy::extensions::filters::http::my_filter::v3::MyFilter proto_config;
  TestUtility::loadFromYaml(yaml, proto_config);

  MyHttpFilterFactory factory;
  
  NiceMock<Http::MockFilterChainFactoryCallbacks> callbacks;
  Http::FilterFactoryCb filter_factory = 
      factory.createFilterFactoryFromProto(proto_config, "stats", context_);

  EXPECT_CALL(callbacks, addStreamDecoderFilter(_));
  filter_factory(callbacks);
}
```

### Network Filter Factory Test

```cpp
TEST_F(MyNetworkFilterFactoryTest, CreatesFilter) {
  const std::string yaml = R"EOF(
    stat_prefix: test
  )EOF";

  envoy::extensions::filters::network::my_filter::v3::MyFilter proto_config;
  TestUtility::loadFromYaml(yaml, proto_config);

  MyNetworkFilterFactory factory;
  
  NiceMock<Network::MockFilterChainFactoryCallbacks> callbacks;
  Network::FilterFactoryCb filter_factory = 
      factory.createFilterFactoryFromProto(proto_config, context_);

  EXPECT_CALL(callbacks, addReadFilter(_));
  filter_factory(callbacks);
}
```

### Config Validation Test

```cpp
TEST_F(MyFilterFactoryTest, ValidatesConfig) {
  const std::string invalid_yaml = R"EOF(
    timeout: -1s  # Invalid negative timeout
  )EOF";

  envoy::extensions::filters::http::my_filter::v3::MyFilter proto_config;
  
  EXPECT_THROW_WITH_MESSAGE(
      TestUtility::loadFromYaml(invalid_yaml, proto_config),
      EnvoyException,
      "timeout must be positive");
}
```

## Integration Testing for Filters

### HTTP Filter Integration Test

```cpp
class MyHttpFilterIntegrationTest : public HttpIntegrationTest {
public:
  MyHttpFilterIntegrationTest()
      : HttpIntegrationTest(Http::CodecType::HTTP2, GetParam()) {}

  void SetUp() override {
    // Add filter to config
    config_helper_.prependFilter(R"EOF(
      name: envoy.filters.http.my_filter
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.http.my_filter.v3.MyFilter
        my_setting: integration_test
    )EOF");
  }
};

TEST_P(MyHttpFilterIntegrationTest, ProcessesRequest) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = sendRequestAndWaitForResponse(
      default_request_headers_, 0,
      default_response_headers_, 0);

  // Verify filter processed request
  EXPECT_TRUE(upstream_request_->headers().has("x-filter-processed"));
  
  EXPECT_TRUE(response->complete());
  EXPECT_EQ("200", response->headers().getStatusValue());
}

INSTANTIATE_TEST_SUITE_P(
    IpVersions, MyHttpFilterIntegrationTest,
    testing::ValuesIn(TestEnvironment::getIpVersionsForTest()));
```

### Network Filter Integration Test

```cpp
class MyNetworkFilterIntegrationTest : public BaseIntegrationTest {
public:
  MyNetworkFilterIntegrationTest()
      : BaseIntegrationTest(GetParam(), getConfig()) {}

  static std::string getConfig() {
    return absl::StrCat(
        ConfigHelper::baseConfig(),
        R"EOF(
          filter_chains:
            filters:
              - name: envoy.filters.network.my_filter
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.network.my_filter.v3.MyFilter
                  stat_prefix: test
              - name: envoy.filters.network.tcp_proxy
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
                  stat_prefix: tcp
                  cluster: cluster_0
        )EOF");
  }
};

TEST_P(MyNetworkFilterIntegrationTest, ProcessesConnection) {
  initialize();

  IntegrationTcpClientPtr tcp_client = 
      makeTcpConnection(lookupPort("listener_0"));
  
  ASSERT_TRUE(tcp_client->write("test data"));
  
  FakeRawConnectionPtr fake_upstream_connection;
  ASSERT_TRUE(fake_upstreams_[0]->waitForRawConnection(
      fake_upstream_connection));
  
  // Verify data was processed by filter
  std::string data;
  ASSERT_TRUE(fake_upstream_connection->waitForData(
      FakeRawConnection::waitForInexactMatch("TEST DATA"), &data));
  
  EXPECT_EQ("TEST DATA", data); // Filter uppercased
}
```

## Best Practices

1. **Test all filter methods**: Don't just test happy path
2. **Test with various header combinations**: Edge cases matter
3. **Test end_stream handling**: Both true and false
4. **Test error conditions**: Invalid input, timeouts, failures
5. **Use NiceMock for callbacks**: Reduces brittleness
6. **Test per-route config**: If filter supports it
7. **Test stats**: Verify counters are updated
8. **Test filter chain position**: Verify filter order matters
9. **Test with both unit and integration tests**: Different perspectives
10. **Document complex behavior**: Help future maintainers

## Common Pitfalls

1. **Forgetting end_stream flag**: Can cause timeouts
2. **Not handling all FilterStatus values**: Incomplete state machine
3. **Memory leaks with buffers**: Use proper RAII
4. **Not testing filter destruction**: onDestroy() cleanup
5. **Ignoring watermarks**: Can cause backpressure issues
6. **Not testing with different protocols**: HTTP/1, HTTP/2, HTTP/3
7. **Forgetting to call continueDecoding()**: After async operations
8. **Not validating configuration**: Can cause runtime errors

## Next Steps

- [Integration Testing Guide](03-integration-testing.md): End-to-end testing
- [Unit Testing Guide](02-unit-testing.md): General unit test patterns
- [HTTP Filter Examples](../../test/extensions/filters/http/): Real filter tests
- [Network Filter Examples](../../test/extensions/filters/network/): Network filter tests
