# Integration Testing Guide

This guide covers Envoy's integration testing framework, which tests complete request flows from downstream clients through Envoy to upstream servers.

## Table of Contents

- [Overview](#overview)
- [Test Class Hierarchy](#test-class-hierarchy)
- [Basic Integration Test](#basic-integration-test)
- [Configuration Helpers](#configuration-helpers)
- [Fake Upstream](#fake-upstream)
- [Common Patterns](#common-patterns)
- [Advanced Features](#advanced-features)
- [Debugging](#debugging)

## Overview

Integration tests verify end-to-end behavior of Envoy by:
1. Starting a real Envoy instance
2. Creating fake upstream servers
3. Sending requests from downstream clients
4. Verifying request/response flow through Envoy
5. Validating headers, body, and connection behavior

### When to Use Integration Tests

Use integration tests to verify:
- Complete request/response flows
- Filter chain behavior
- Protocol handling (HTTP/1.1, HTTP/2, HTTP/3)
- Configuration changes
- Connection management
- TLS/SSL behavior
- Timeout and retry behavior

### Integration Test Structure

```cpp
class MyIntegrationTest : public HttpIntegrationTest {
public:
  MyIntegrationTest() 
      : HttpIntegrationTest(Http::CodecType::HTTP2, GetParam()) {}
  
  void SetUp() override {
    // Optional: Configure test environment
  }
};

TEST_P(MyIntegrationTest, BasicFlow) {
  // Initialize Envoy with configuration
  initialize();
  
  // Create client connection
  codec_client_ = makeHttpConnection(lookupPort("http"));
  
  // Send request and wait for response
  auto response = sendRequestAndWaitForResponse(
      default_request_headers_, 0,
      default_response_headers_, 0);
  
  // Verify results
  EXPECT_TRUE(response->complete());
  EXPECT_EQ("200", response->headers().getStatusValue());
}

INSTANTIATE_TEST_SUITE_P(
    IpVersions, MyIntegrationTest,
    testing::ValuesIn(TestEnvironment::getIpVersionsForTest()));
```

## Test Class Hierarchy

### BaseIntegrationTest

Base class providing core integration test functionality:
- Envoy server lifecycle management
- Fake upstream creation
- Configuration helpers
- Stats and counter access
- Wait utilities

```cpp
class BaseIntegrationTest {
protected:
  // Create fake upstreams
  void createUpstream();
  
  // Initialize and start Envoy
  virtual void initialize();
  
  // Wait for counter to reach value
  void waitForCounterGe(const std::string& name, uint64_t value);
  
  // Access to test server
  IntegrationTestServerPtr test_server_;
  
  // Fake upstreams
  std::vector<std::unique_ptr<FakeUpstream>> fake_upstreams_;
  
  // Configuration helper
  ConfigHelper config_helper_;
};
```

### HttpIntegrationTest

Extends BaseIntegrationTest with HTTP-specific functionality:

```cpp
class HttpIntegrationTest : public BaseIntegrationTest {
public:
  HttpIntegrationTest(
      Http::CodecType downstream_protocol,
      Network::Address::IpVersion version);

protected:
  // Create HTTP connection
  IntegrationCodecClientPtr makeHttpConnection(uint32_t port);
  
  // Send request and wait for response
  IntegrationStreamDecoderPtr sendRequestAndWaitForResponse(
      const Http::RequestHeaderMap& request_headers,
      uint32_t request_body_size,
      const Http::ResponseHeaderMap& response_headers,
      uint32_t response_body_size);
  
  // Wait for next upstream request
  void waitForNextUpstreamRequest();
  
  // Default headers
  Http::TestRequestHeaderMapImpl default_request_headers_;
  Http::TestResponseHeaderMapImpl default_response_headers_;
  
  // Active client and connections
  IntegrationCodecClientPtr codec_client_;
  FakeHttpConnectionPtr fake_upstream_connection_;
  FakeStreamPtr upstream_request_;
};
```

## Basic Integration Test

### Simple Request/Response Test

```cpp
TEST_P(IntegrationTest, RouterRequestAndResponseWithBodyNoBuffer) {
  // Start Envoy and fake upstreams
  initialize();

  // Create downstream client connection
  codec_client_ = makeHttpConnection(lookupPort("http"));

  // Create request headers
  Http::TestRequestHeaderMapImpl request_headers{
      {":method", "POST"},
      {":path", "/test/long/url"},
      {":scheme", "http"},
      {":authority", "host"},
      {"x-lyft-user-id", "123"}};

  // Send request with body, wait for response
  auto response = sendRequestAndWaitForResponse(
      request_headers, 1024,           // Request: headers + 1024 bytes
      default_response_headers_, 512); // Response: headers + 512 bytes

  // Verify proxied request upstream
  EXPECT_TRUE(upstream_request_->complete());
  EXPECT_EQ(1024U, upstream_request_->bodyLength());
  EXPECT_EQ("host", 
      upstream_request_->headers().getHostValue());

  // Verify proxied response downstream
  EXPECT_TRUE(response->complete());
  EXPECT_EQ("200", response->headers().getStatusValue());
  EXPECT_EQ(512U, response->body().size());
}
```

### Header-Only Test

```cpp
TEST_P(IntegrationTest, RouterHeaderOnlyRequestAndResponse) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));

  // Make header-only request
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  // Wait for request to arrive upstream
  waitForNextUpstreamRequest();

  // Send header-only response
  upstream_request_->encodeHeaders(default_response_headers_, true);

  // Wait for response downstream
  ASSERT_TRUE(response->waitForEndStream());

  // Verify
  EXPECT_TRUE(upstream_request_->complete());
  EXPECT_EQ(0U, upstream_request_->bodyLength());
  EXPECT_TRUE(response->complete());
  EXPECT_EQ("200", response->headers().getStatusValue());
  EXPECT_EQ(0U, response->body().size());
}
```

### Manual Request Control

```cpp
TEST_P(IntegrationTest, ManualRequestControl) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));

  // Start request but don't send body yet
  auto encoder_decoder = 
      codec_client_->startRequest(default_request_headers_);
  auto& request_encoder = encoder_decoder.first;
  auto& response = encoder_decoder.second;

  // Send body in chunks
  Buffer::OwnedImpl data1("hello");
  codec_client_->sendData(request_encoder, data1, false);

  Buffer::OwnedImpl data2(" world");
  codec_client_->sendData(request_encoder, data2, true);

  // Wait for upstream request
  waitForNextUpstreamRequest();
  
  EXPECT_EQ("hello world", upstream_request_->body().toString());

  // Send response
  upstream_request_->encodeHeaders(default_response_headers_, true);

  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_TRUE(response->complete());
}
```

## Configuration Helpers

The `ConfigHelper` class provides utilities for modifying Envoy configuration.

### Basic Configuration

```cpp
TEST_P(IntegrationTest, BasicConfig) {
  // Use default HTTP proxy config
  initialize();
  
  // Test uses ConfigHelper::httpProxyConfig() by default
}
```

### Set Downstream Protocol

```cpp
TEST_P(IntegrationTest, Http2Downstream) {
  // Configure downstream as HTTP/2
  setDownstreamProtocol(Http::CodecType::HTTP2);
  initialize();
  
  // Downstream connection uses HTTP/2
  codec_client_ = makeHttpConnection(lookupPort("http"));
}
```

### Add Filter to Chain

```cpp
TEST_P(IntegrationTest, AddBufferFilter) {
  // Add buffer filter before router
  config_helper_.prependFilter(ConfigHelper::DEFAULT_BUFFER_FILTER);
  initialize();
  
  // Requests now buffered
}
```

### Modify Bootstrap Configuration

```cpp
TEST_P(IntegrationTest, ModifyBootstrap) {
  config_helper_.addConfigModifier(
      [](envoy::config::bootstrap::v3::Bootstrap& bootstrap) {
        // Add runtime override
        auto* runtime = bootstrap.mutable_layered_runtime();
        auto* layer = runtime->add_layers();
        layer->set_name("test_override");
        (*layer->mutable_static_layer()
            ->mutable_fields())["feature.enabled"]
            .set_bool_value(true);
      });
  
  initialize();
}
```

### Modify HttpConnectionManager

```cpp
TEST_P(IntegrationTest, ModifyHCM) {
  config_helper_.addConfigModifier(
      [](envoy::extensions::filters::network::http_connection_manager::v3::
             HttpConnectionManager& hcm) {
        // Enable tracing
        auto* tracing = hcm.mutable_tracing();
        tracing->mutable_client_sampling()->set_value(100);
        tracing->mutable_random_sampling()->set_value(100);
        tracing->mutable_overall_sampling()->set_value(100);
      });
  
  initialize();
}
```

### Add Cluster

```cpp
TEST_P(IntegrationTest, AddCluster) {
  config_helper_.addConfigModifier(
      [](envoy::config::bootstrap::v3::Bootstrap& bootstrap) {
        // Add new cluster
        auto* cluster = bootstrap.mutable_static_resources()
                           ->add_clusters();
        cluster->set_name("extra_cluster");
        cluster->set_type(envoy::config::cluster::v3::Cluster::STATIC);
        cluster->mutable_load_assignment()->set_cluster_name("extra_cluster");
        
        auto* endpoint = cluster->mutable_load_assignment()
                            ->add_endpoints()
                            ->add_lb_endpoints();
        endpoint->mutable_endpoint()->mutable_address()
            ->mutable_socket_address()
            ->set_address("127.0.0.1");
        endpoint->mutable_endpoint()->mutable_address()
            ->mutable_socket_address()
            ->set_port_value(8080);
      });
  
  initialize();
}
```

### Runtime Overrides

```cpp
TEST_P(IntegrationTest, RuntimeOverride) {
  config_helper_.addRuntimeOverride(
      "envoy.reloadable_features.my_feature", "true");
  
  initialize();
}
```

## Fake Upstream

Fake upstreams simulate backend servers for testing.

### Basic Fake Upstream Usage

```cpp
TEST_P(IntegrationTest, FakeUpstreamUsage) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  // Wait for connection to upstream
  waitForNextUpstreamRequest();
  
  // fake_upstream_connection_ and upstream_request_ are now available
  
  // Check received request
  EXPECT_EQ("/test/long/url", 
      upstream_request_->headers().getPathValue());
  
  // Send custom response
  Http::TestResponseHeaderMapImpl response_headers{
      {":status", "200"},
      {"content-type", "text/plain"},
      {"x-custom-header", "value"}};
  
  upstream_request_->encodeHeaders(response_headers, false);
  upstream_request_->encodeData(100, true);

  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_EQ("200", response->headers().getStatusValue());
  EXPECT_EQ(100U, response->body().size());
}
```

### Multiple Upstreams

```cpp
TEST_P(IntegrationTest, MultipleUpstreams) {
  // Create 3 upstream servers
  setUpstreamCount(3);
  initialize();

  // fake_upstreams_[0], [1], [2] are now available
  
  codec_client_ = makeHttpConnection(lookupPort("http"));
  
  // Send requests - will load balance across upstreams
  auto response1 = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);
  auto response2 = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);
  auto response3 = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  // Handle on different upstreams
  waitForNextUpstreamRequest(0);
  upstream_request_->encodeHeaders(default_response_headers_, true);
  
  waitForNextUpstreamRequest(1);
  upstream_request_->encodeHeaders(default_response_headers_, true);
  
  waitForNextUpstreamRequest(2);
  upstream_request_->encodeHeaders(default_response_headers_, true);

  ASSERT_TRUE(response1->waitForEndStream());
  ASSERT_TRUE(response2->waitForEndStream());
  ASSERT_TRUE(response3->waitForEndStream());
}
```

### Autonomous Upstream

Autonomous upstreams automatically respond to requests:

```cpp
TEST_P(IntegrationTest, AutonomousUpstream) {
  // Enable autonomous mode
  autonomous_upstream_ = true;
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));

  // Request automatically gets "200 OK" response
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_EQ("200", response->headers().getStatusValue());
  
  // No need to manually handle upstream request!
}
```

#### Controlling Autonomous Response

```cpp
TEST_P(IntegrationTest, AutonomousCustomResponse) {
  autonomous_upstream_ = true;
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));

  // Control response via request headers
  Http::TestRequestHeaderMapImpl request_headers{
      {":method", "GET"},
      {":path", "/test"},
      {":scheme", "http"},
      {":authority", "host"},
      {"x-envoy-upstream-rq-timeout-ms", "1000"},
      // Autonomous upstream control headers
      {AutonomousStream::RESPONSE_STATUS, "404"},
      {AutonomousStream::RESPONSE_SIZE_BYTES, "100"},
      {AutonomousStream::EXPECT_REQUEST_SIZE_BYTES, "0"}};

  auto response = codec_client_->makeHeaderOnlyRequest(request_headers);

  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_EQ("404", response->headers().getStatusValue());
  EXPECT_EQ(100U, response->body().size());
}
```

## Common Patterns

### Waiting for Conditions

```cpp
// Wait for response to complete
ASSERT_TRUE(response->waitForEndStream());

// Wait for upstream request
waitForNextUpstreamRequest();

// Wait for counter value
test_server_->waitForCounterGe("cluster.cluster_0.upstream_cx_total", 1);
test_server_->waitForGaugeEq("cluster.cluster_0.membership_total", 1);

// Wait with custom timeout
ASSERT_TRUE(response->waitForEndStream(
    std::chrono::milliseconds(5000)));
```

### Checking Stats

```cpp
TEST_P(IntegrationTest, CheckStats) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = sendRequestAndWaitForResponse(
      default_request_headers_, 0,
      default_response_headers_, 0);

  // Check specific counters
  test_server_->waitForCounterGe(
      "http.config_test.downstream_rq_2xx", 1);
  test_server_->waitForCounterGe(
      "cluster.cluster_0.upstream_rq_200", 1);
  
  // Get counter value
  uint64_t requests = test_server_->counter(
      "http.config_test.downstream_rq_total")->value();
  EXPECT_EQ(1, requests);
}
```

### Testing Timeouts

```cpp
TEST_P(IntegrationTest, Timeout) {
  config_helper_.addConfigModifier(
      [](envoy::extensions::filters::network::http_connection_manager::v3::
             HttpConnectionManager& hcm) {
        // Set short timeout
        hcm.mutable_stream_idle_timeout()->set_seconds(1);
      });
  
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  waitForNextUpstreamRequest();
  
  // Don't send response - let it timeout
  // Upstream will timeout after 1 second
  
  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_EQ("504", response->headers().getStatusValue());
}
```

### Testing Connection Closure

```cpp
TEST_P(IntegrationTest, ConnectionClose) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  waitForNextUpstreamRequest();
  
  // Close upstream connection
  ASSERT_TRUE(fake_upstream_connection_->close());
  
  // Wait for downstream connection to close
  ASSERT_TRUE(fake_upstream_connection_->waitForDisconnect());

  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_EQ("503", response->headers().getStatusValue());
}
```

### Testing Retries

```cpp
TEST_P(IntegrationTest, RetryOnFailure) {
  config_helper_.addConfigModifier(
      [](envoy::extensions::filters::network::http_connection_manager::v3::
             HttpConnectionManager& hcm) {
        auto* route = hcm.mutable_route_config()
                          ->mutable_virtual_hosts(0)
                          ->mutable_routes(0);
        auto* retry = route->mutable_route()
                          ->mutable_retry_policy();
        retry->set_retry_on("5xx");
        retry->mutable_num_retries()->set_value(2);
      });
  
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  // First attempt - return 503
  waitForNextUpstreamRequest();
  Http::TestResponseHeaderMapImpl error_response{
      {":status", "503"}};
  upstream_request_->encodeHeaders(error_response, true);

  // Second attempt (retry) - return 200
  waitForNextUpstreamRequest();
  upstream_request_->encodeHeaders(default_response_headers_, true);

  ASSERT_TRUE(response->waitForEndStream());
  EXPECT_EQ("200", response->headers().getStatusValue());
  
  // Verify retry stats
  test_server_->waitForCounterGe(
      "cluster.cluster_0.upstream_rq_retry", 1);
}
```

## Advanced Features

### Custom Validation

```cpp
TEST_P(IntegrationTest, CustomValidation) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);

  waitForNextUpstreamRequest();
  
  // Validate custom headers were added
  EXPECT_TRUE(upstream_request_->headers().has("x-envoy-expected-rq-timeout-ms"));
  
  upstream_request_->encodeHeaders(default_response_headers_, true);

  ASSERT_TRUE(response->waitForEndStream());
}
```

### Using Access Logs

```cpp
TEST_P(IntegrationTest, AccessLog) {
  useAccessLog();
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto response = sendRequestAndWaitForResponse(
      default_request_headers_, 0,
      default_response_headers_, 0);

  // Wait for access log entry
  std::string log = waitForAccessLog(
      TestEnvironment::temporaryPath("access.log"));
  
  EXPECT_THAT(log, HasSubstr("200"));
  EXPECT_THAT(log, HasSubstr("/test/long/url"));
}
```

### Protocol-Specific Tests

```cpp
// HTTP/2 specific
TEST_P(Http2IntegrationTest, Trailers) {
  initialize();

  codec_client_ = makeHttpConnection(lookupPort("http"));
  auto encoder_decoder = 
      codec_client_->startRequest(default_request_headers_);
  auto& request_encoder = encoder_decoder.first;
  auto& response = encoder_decoder.second;

  Buffer::OwnedImpl data("body");
  codec_client_->sendData(request_encoder, data, false);

  // Send trailers
  Http::TestRequestTrailerMapImpl trailers{
      {"trailer-key", "trailer-value"}};
  codec_client_->sendTrailers(request_encoder, trailers);

  waitForNextUpstreamRequest();
  
  EXPECT_EQ("trailer-value", 
      upstream_request_->trailers()->get("trailer-key")[0]->value().getStringView());

  upstream_request_->encodeHeaders(default_response_headers_, true);
  ASSERT_TRUE(response->waitForEndStream());
}
```

## Debugging

### Enable Debug Logging

```bash
# Run test with debug logs
bazel test //test/integration:integration_test --test_arg="-l debug"

# Run with trace logs
bazel test //test/integration:integration_test --test_arg="-l trace"

# Filter to specific test
bazel test //test/integration:integration_test \
  --test_arg=--gtest_filter="*MySpecificTest*" \
  --test_arg="-l trace"
```

### Common Debugging Techniques

1. **Add logging to test**:
```cpp
TEST_P(IntegrationTest, Debug) {
  initialize();
  
  std::cerr << "About to create connection\n";
  codec_client_ = makeHttpConnection(lookupPort("http"));
  
  std::cerr << "Sending request\n";
  auto response = codec_client_->makeHeaderOnlyRequest(
      default_request_headers_);
  
  std::cerr << "Waiting for upstream\n";
  waitForNextUpstreamRequest();
}
```

2. **Check timeouts**:
   - Tests timing out? Check for missing `waitForEndStream()` or `waitForNextUpstreamRequest()`
   - Response not completing? Verify `end_stream` flags

3. **Inspect fake upstream**:
```cpp
waitForNextUpstreamRequest();

// Print received headers
for (const auto& header : upstream_request_->headers()) {
  std::cerr << header.first.getStringView() << ": " 
            << header.second.getStringView() << "\n";
}

// Print body
std::cerr << "Body: " << upstream_request_->body().toString() << "\n";
```

4. **Check stats for clues**:
```cpp
// Dump all stats
test_server_->statStore().forEachCounter(
    [](std::size_t, Stats::Counter& counter) {
      std::cerr << counter.name() << ": " << counter.value() << "\n";
      return true;
    });
```

### Flaky Test Debugging

```bash
# Run test repeatedly
bazel test //test/integration:integration_test \
  --runs_per_test=100 \
  --test_arg=--gtest_filter="*FlakyTest*"

# Run with more parallelism to stress test
bazel test //test/integration:integration_test \
  --jobs=60 --local_test_jobs=60 \
  --runs_per_test=100
```

## Best Practices

1. **Use autonomous_upstream for simple tests**: Reduces boilerplate
2. **Verify both upstream and downstream**: Check request was correctly proxied
3. **Test realistic scenarios**: Use patterns from production
4. **Clean up resources**: Let RAII handle cleanup
5. **Use waitFor helpers**: Don't poll manually
6. **Test error cases**: Not just happy path
7. **Keep tests focused**: One scenario per test
8. **Use parameterized tests**: Test multiple IP versions, protocols
9. **Document complex tests**: Explain non-obvious logic
10. **Check stats**: Verify counters match expected behavior

## Next Steps

- [Filter Testing Guide](04-filter-testing.md): Specific filter testing patterns
- [Unit Testing Guide](02-unit-testing.md): Unit test patterns
- [Integration Test README](../../test/integration/README.md): More details
- [ConfigHelper](../../test/config/utility.h): Configuration helpers
