# Common Network and HTTP Filter APIs

This is a local reference for the core filter interfaces Envoy extensions implement.
Canonical definitions live in:

- Network: [`envoy/network/filter.h`](../../envoy/network/filter.h)
- HTTP: [`envoy/http/filter.h`](../../envoy/http/filter.h)

Related docs: [async_http_filters.md](async_http_filters.md), [flow_control.md](flow_control.md),
[upstream_filters.md](upstream_filters.md).

---

## Network filters

Network filters operate on a byte stream for a single connection (L4). Filters may be
read-only, write-only, or both.

### Status

```cpp
enum class FilterStatus {
  Continue,       // Continue to further filters
  StopIteration,  // Stop executing further filters
};
```

### `NetworkFilterCallbacks`

Shared by read and write filter callbacks:

```cpp
virtual Connection& connection() = 0;
virtual const ConnectionSocket& socket() = 0;
```

### `ReadFilter` (inbound / read path)

```cpp
virtual FilterStatus onData(Buffer::Instance& data, bool end_stream) = 0;
virtual FilterStatus onNewConnection() = 0;
virtual void initializeReadFilterCallbacks(ReadFilterCallbacks& callbacks) = 0;
virtual bool startUpstreamSecureTransport();  // default: false
```

Helper used by many read-only filters ([`source/common/network/filter_impl.h`](../common/network/filter_impl.h)):

```cpp
class ReadFilterBaseImpl : public ReadFilter {
  void initializeReadFilterCallbacks(ReadFilterCallbacks&) override {}
  FilterStatus onNewConnection() override { return FilterStatus::Continue; }
};
```

### `ReadFilterCallbacks`

```cpp
virtual void continueReading() = 0;
virtual void injectReadDataToFilterChain(Buffer::Instance& data, bool end_stream) = 0;
virtual Upstream::HostDescriptionConstSharedPtr upstreamHost() = 0;
virtual void upstreamHost(Upstream::HostDescriptionConstSharedPtr host) = 0;
virtual bool startUpstreamSecureTransport() = 0;
virtual void disableClose(bool disable) = 0;
```

Notes:

- If `onData()` / `onNewConnection()` returns `StopIteration`, call `continueReading()` to resume.
- Prefer `injectReadDataToFilterChain()` when the filter manages its own buffering and wants to
  pass an explicit buffer / `end_stream` to subsequent filters.
- Do not do outbound networking in `initializeReadFilterCallbacks()`; use `onNewConnection()`.

### `WriteFilter` (outbound / write path)

```cpp
virtual FilterStatus onWrite(Buffer::Instance& data, bool end_stream) = 0;
virtual void initializeWriteFilterCallbacks(WriteFilterCallbacks&);  // default: no-op
```

### `WriteFilterCallbacks`

```cpp
virtual void injectWriteDataToFilterChain(Buffer::Instance& data, bool end_stream) = 0;
virtual void disableClose(bool disabled) = 0;
virtual void onAboveWriteBufferHighWatermark() = 0;
virtual void onBelowWriteBufferLowWatermark() = 0;
```

### `Filter` (read + write)

```cpp
class Filter : public WriteFilter, public ReadFilter {};
```

### `FilterManager`

```cpp
virtual void addWriteFilter(WriteFilterSharedPtr filter) = 0;
virtual void addReadFilter(ReadFilterSharedPtr filter) = 0;
virtual void addFilter(FilterSharedPtr filter) = 0;
virtual void removeReadFilter(ReadFilterSharedPtr filter) = 0;
virtual bool initializeReadFilters() = 0;  // calls onNewConnection() on installed read filters
virtual void addAccessLogHandler(AccessLog::InstanceSharedPtr handler) = 0;
```

Invocation order:

- Read filters: FIFO (first added is called first)
- Write filters: LIFO (last added is called first)

### Typical read-only filter shape

Example pattern (see network RBAC filter):

```cpp
class MyNetworkFilter : public Network::ReadFilter,
                        public Network::ConnectionCallbacks {
  Network::FilterStatus onData(Buffer::Instance& data, bool end_stream) override;
  Network::FilterStatus onNewConnection() override;
  void initializeReadFilterCallbacks(Network::ReadFilterCallbacks& callbacks) override;

  void onEvent(Network::ConnectionEvent event) override;
  void onAboveWriteBufferHighWatermark() override {}
  void onBelowWriteBufferLowWatermark() override {}
};
```

---

## HTTP filters

HTTP filters operate on a request/response stream (L7) inside the HTTP connection manager.
Filters may be decoder-only, encoder-only, or both (`StreamFilter`).

### Status enums

```cpp
enum class FilterHeadersStatus {
  Continue,
  StopIteration,
  ContinueAndDontEndStream,
  StopAllIterationAndBuffer,
  StopAllIterationAndWatermark,
};

enum class FilterDataStatus {
  Continue,
  StopIterationAndBuffer,
  StopIterationAndWatermark,
  StopIterationNoBuffer,
};

enum class FilterTrailersStatus { Continue, StopIteration };
enum class Filter1xxHeadersStatus { Continue, StopIteration };

enum class FilterMetadataStatus {
  Continue,
  ContinueAll,
  StopIterationForLocalReply,
};

enum class LocalErrorStatus {
  Continue,
  ContinueAndResetStream,
};
```

Common patterns:

- Return `StopIteration` / `StopIterationAnd*` while waiting for async work, then call
  `continueDecoding()` / `continueEncoding()` (or `sendLocalReply()`).
- `StopIterationAndBuffer` buffers body data; exceeding limits yields 413 (request) / 500 (response).
- `StopIterationAndWatermark` applies backpressure without unbounded buffering.

### `StreamFilterBase`

```cpp
virtual void onStreamComplete();  // enrich StreamInfo before access logs; default no-op
virtual void onDestroy() = 0;     // cancel timers / outstanding async work; no more callbacks after this
virtual void onMatchCallback(const Matcher::Action&);  // default no-op
virtual LocalErrorStatus onLocalReply(const LocalReplyData&);  // default Continue
```

Do not call encoder/decoder callbacks after `onDestroy()`.

### `StreamDecoderFilter` (request / decode path)

```cpp
virtual FilterHeadersStatus decodeHeaders(RequestHeaderMap& headers, bool end_stream) = 0;
virtual FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) = 0;
virtual FilterTrailersStatus decodeTrailers(RequestTrailerMap& trailers) = 0;
virtual FilterMetadataStatus decodeMetadata(MetadataMap& metadata_map);  // default Continue
virtual void setDecoderFilterCallbacks(StreamDecoderFilterCallbacks& callbacks) = 0;
virtual void decodeComplete();  // default no-op
```

### `StreamEncoderFilter` (response / encode path)

```cpp
virtual Filter1xxHeadersStatus encode1xxHeaders(ResponseHeaderMap& headers) = 0;
virtual FilterHeadersStatus encodeHeaders(ResponseHeaderMap& headers, bool end_stream) = 0;
virtual FilterDataStatus encodeData(Buffer::Instance& data, bool end_stream) = 0;
virtual FilterTrailersStatus encodeTrailers(ResponseTrailerMap& trailers) = 0;
virtual FilterMetadataStatus encodeMetadata(MetadataMap& metadata_map) = 0;
virtual void setEncoderFilterCallbacks(StreamEncoderFilterCallbacks& callbacks) = 0;
virtual void encodeComplete();  // default no-op
```

### `StreamFilter` (decode + encode)

```cpp
class StreamFilter : public virtual StreamDecoderFilter,
                     public virtual StreamEncoderFilter {};
```

### `StreamFilterCallbacks` (shared)

```cpp
virtual OptRef<const Network::Connection> connection() = 0;
virtual Event::Dispatcher& dispatcher() = 0;
virtual void resetStream(StreamResetReason = LocalReset, absl::string_view = "") = 0;

virtual OptRef<const Router::Route> route() = 0;
virtual Router::RouteConstSharedPtr routeSharedPtr() = 0;
virtual OptRef<const Upstream::ClusterInfo> clusterInfo() = 0;
virtual Upstream::ClusterInfoConstSharedPtr clusterInfoSharedPtr() = 0;

virtual uint64_t streamId() const = 0;
virtual StreamInfo::StreamInfo& streamInfo() = 0;
virtual Tracing::Span& activeSpan() = 0;
virtual OptRef<const Tracing::Config> tracingConfig() const = 0;
virtual const ScopeTrackedObject& scope() = 0;
virtual void restoreContextOnContinue(ScopeTrackedObjectStack&) = 0;
virtual void resetIdleTimer() = 0;

virtual const Router::RouteSpecificFilterConfig* mostSpecificPerFilterConfig() const = 0;
virtual Router::RouteSpecificFilterConfigs perFilterConfigs() const = 0;
virtual Http1StreamEncoderOptionsOptRef http1StreamEncoderOptions() = 0;

virtual OptRef<UpstreamStreamFilterCallbacks> upstreamCallbacks() = 0;
virtual OptRef<DownstreamStreamFilterCallbacks> downstreamCallbacks() = 0;
virtual absl::string_view filterConfigName() const = 0;

virtual RequestHeaderMapOptRef requestHeaders() = 0;
virtual RequestTrailerMapOptRef requestTrailers() = 0;
virtual ResponseHeaderMapOptRef informationalHeaders() = 0;
virtual ResponseHeaderMapOptRef responseHeaders() = 0;
virtual ResponseTrailerMapOptRef responseTrailers() = 0;

virtual void setBufferLimit(uint64_t limit) = 0;
virtual uint64_t bufferLimit() = 0;
```

### `StreamDecoderFilterCallbacks` (extras)

```cpp
virtual void continueDecoding() = 0;
virtual const Buffer::Instance* decodingBuffer() = 0;
virtual void modifyDecodingBuffer(std::function<void(Buffer::Instance&)>) = 0;
virtual void addDecodedData(Buffer::Instance& data, bool streaming_filter) = 0;
virtual void injectDecodedDataToFilterChain(Buffer::Instance& data, bool end_stream) = 0;
virtual RequestTrailerMap& addDecodedTrailers() = 0;
virtual MetadataMapVector& addDecodedMetadata() = 0;

virtual void sendLocalReply(
    Code response_code, absl::string_view body_text,
    std::function<void(ResponseHeaderMap& headers)> modify_headers,
    const absl::optional<Grpc::Status::GrpcStatus> grpc_status,
    absl::string_view details) = 0;
virtual void sendGoAwayAndClose(bool graceful = false) = 0;

// Direct encode from a decoder filter (e.g. local response construction helpers)
virtual void encode1xxHeaders(ResponseHeaderMapPtr&& headers) = 0;
virtual void encodeHeaders(ResponseHeaderMapPtr&& headers, bool end_stream,
                           absl::string_view details) = 0;
virtual void encodeData(Buffer::Instance& data, bool end_stream) = 0;
virtual void encodeTrailers(ResponseTrailerMapPtr&& trailers) = 0;
virtual void encodeMetadata(MetadataMapPtr&& metadata_map) = 0;

virtual void onDecoderFilterAboveWriteBufferHighWatermark() = 0;
virtual void onDecoderFilterBelowWriteBufferLowWatermark() = 0;
virtual void addDownstreamWatermarkCallbacks(DownstreamWatermarkCallbacks&) = 0;
virtual void removeDownstreamWatermarkCallbacks(DownstreamWatermarkCallbacks&) = 0;

virtual bool recreateStream(const ResponseHeaderMap* original_response_headers) = 0;
virtual void addUpstreamSocketOptions(const Network::Socket::OptionsSharedPtr&) = 0;
virtual Network::Socket::OptionsSharedPtr getUpstreamSocketOptions() const = 0;
virtual void setUpstreamOverrideHost(Upstream::LoadBalancerContext::OverrideHost) = 0;
virtual OptRef<const Upstream::LoadBalancerContext::OverrideHost> upstreamOverrideHost() const = 0;
virtual bool shouldLoadShed() const = 0;
```

### `StreamEncoderFilterCallbacks` (extras)

```cpp
virtual void continueEncoding() = 0;
virtual const Buffer::Instance* encodingBuffer() = 0;
virtual void modifyEncodingBuffer(std::function<void(Buffer::Instance&)>) = 0;
virtual void addEncodedData(Buffer::Instance& data, bool streaming_filter) = 0;
virtual void injectEncodedDataToFilterChain(Buffer::Instance& data, bool end_stream) = 0;
virtual ResponseTrailerMap& addEncodedTrailers() = 0;
virtual void addEncodedMetadata(MetadataMapPtr&& metadata_map) = 0;

virtual void sendLocalReply(
    Code response_code, absl::string_view body_text,
    std::function<void(ResponseHeaderMap& headers)> modify_headers,
    const absl::optional<Grpc::Status::GrpcStatus> grpc_status,
    absl::string_view details) = 0;

virtual void onEncoderFilterAboveWriteBufferHighWatermark() = 0;
virtual void onEncoderFilterBelowWriteBufferLowWatermark() = 0;
```

Notes:

- `addDecodedData()` / `addEncodedData()` participate in HCM buffering and continuation.
- `injectDecodedDataToFilterChain()` / `injectEncodedDataToFilterChain()` pass data directly to
  later filters; call outside of `decodeData()` / `encodeData()`, and typically pair with
  `StopIterationNoBuffer`.

### `FilterChainFactoryCallbacks`

Used when a filter factory installs instances into the stream filter chain:

```cpp
virtual void addStreamDecoderFilter(StreamDecoderFilterSharedPtr filter) = 0;
virtual void addStreamEncoderFilter(StreamEncoderFilterSharedPtr filter) = 0;
virtual void addStreamFilter(StreamFilterSharedPtr filter) = 0;
virtual void addAccessLogHandler(AccessLog::InstanceSharedPtr handler) = 0;
virtual Event::Dispatcher& dispatcher() = 0;
virtual absl::string_view filterConfigName() const = 0;
virtual void setFilterConfigName(absl::string_view name) = 0;
```

### Typical dual HTTP filter shape

```cpp
class MyHttpFilter : public Http::StreamFilter,
                     public Logger::Loggable<Logger::Id::filter> {
  // StreamFilterBase
  void onDestroy() override;

  // StreamDecoderFilter
  Http::FilterHeadersStatus decodeHeaders(Http::RequestHeaderMap& headers,
                                          bool end_stream) override;
  Http::FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) override;
  Http::FilterTrailersStatus decodeTrailers(Http::RequestTrailerMap& trailers) override;
  void setDecoderFilterCallbacks(Http::StreamDecoderFilterCallbacks& callbacks) override;

  // StreamEncoderFilter
  Http::Filter1xxHeadersStatus encode1xxHeaders(Http::ResponseHeaderMap& headers) override;
  Http::FilterHeadersStatus encodeHeaders(Http::ResponseHeaderMap& headers,
                                          bool end_stream) override;
  Http::FilterDataStatus encodeData(Buffer::Instance& data, bool end_stream) override;
  Http::FilterTrailersStatus encodeTrailers(Http::ResponseTrailerMap& trailers) override;
  Http::FilterMetadataStatus encodeMetadata(Http::MetadataMap& metadata_map) override;
  void setEncoderFilterCallbacks(Http::StreamEncoderFilterCallbacks& callbacks) override;
};
```

Unused decode/encode methods commonly return `Continue` / `Continue`-equivalent statuses.

---

## Quick comparison

| Concern | Network filter | HTTP filter |
| --- | --- | --- |
| Unit of work | Connection byte stream | HTTP request/response stream |
| Read / request entrypoints | `onNewConnection()`, `onData()` | `decodeHeaders()`, `decodeData()`, `decodeTrailers()` |
| Write / response entrypoints | `onWrite()` | `encode1xxHeaders()`, `encodeHeaders()`, `encodeData()`, `encodeTrailers()` |
| Continue after stop | `continueReading()` | `continueDecoding()` / `continueEncoding()` |
| Local error response | Close connection / write directly | `sendLocalReply()` |
| Combined type | `Network::Filter` | `Http::StreamFilter` |
| Lifecycle cleanup | Filter / connection destruction | `onDestroy()` (required) |
