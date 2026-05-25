# `tracing/` — Class hierarchy (UML)

Interfaces from `envoy/tracing/` and `envoy/server/tracer_config.h` are in italics. Concrete classes in this
folder are bold.

```mermaid
classDiagram
    direction LR

    %% =========================
    %% Tracer / Driver / Span
    %% =========================
    class Tracer {
        <<interface>>
        +startSpan(Config, TraceContext, StreamInfo, Decision) SpanPtr
    }
    class Driver {
        <<interface>>
        +startSpan(Config, TraceContext, StreamInfo, op_name, Decision) SpanPtr
    }
    class Span {
        <<interface>>
        +setOperation(name)
        +setTag(key, value)
        +log(ts, event)
        +finishSpan()
        +injectContext(TraceContext, UpstreamContext)
        +setBaggage(k, v)
        +getBaggage(k) string
        +getSpanId() string
        +getTraceId() string
        +spawnChild(Config, op_name, ts) SpanPtr
        +setSampled(bool)
        +exportedSpan() bool
        +useLocalDecision() bool
    }

    class TracerImpl {
        -driver_ DriverSharedPtr
        -local_info_ LocalInfo&
        +startSpan(...)
        +driverForTest() DriverSharedPtr
    }
    class NullTracer {
        +startSpan(...) NullSpan
    }
    class NullSpan {
        +(no-op everything)
        +static instance() NullSpan&
    }

    Tracer <|.. TracerImpl
    Tracer <|.. NullTracer
    Span <|.. NullSpan
    TracerImpl --> Driver : driver_

    %% =========================
    %% TracerManager
    %% =========================
    class TracerManager {
        <<interface>>
        +getOrCreateTracer(Tracing_Http*) TracerSharedPtr
    }
    class TracerManagerImpl {
        -factory_context_ TracerFactoryContextPtr
        -null_tracer_ TracerSharedPtr
        -tracers_ flat_hash_map~size_t, weak_ptr~Tracer~~
        +getOrCreateTracer(cfg) TracerSharedPtr
        +singleton(FactoryContext) shared_ptr~TracerManager~
        -removeExpiredCacheEntries()
    }
    TracerManager <|.. TracerManagerImpl
    TracerManagerImpl --> TracerImpl : creates / caches weak
    TracerManagerImpl --> NullTracer : null_tracer_

    %% =========================
    %% TracerFactoryContext
    %% =========================
    class TracerFactoryContext {
        <<interface>>
        +serverFactoryContext() ServerFactoryContext&
        +messageValidationVisitor() ValidationVisitor&
    }
    class TracerFactoryContextImpl {
        -server_factory_context_ ServerFactoryContext&
        -validation_visitor_ ValidationVisitor&
        +serverFactoryContext()
        +messageValidationVisitor()
    }
    TracerFactoryContext <|.. TracerFactoryContextImpl
    TracerManagerImpl --> TracerFactoryContextImpl : factory_context_

    %% =========================
    %% Config
    %% =========================
    class Config {
        <<interface>>
        +operationName() OperationName
        +customTags() CustomTagMap*
        +verbose() bool
        +maxPathTagLength() uint32_t
        +spawnUpstreamSpan() bool
        +noContextPropagation() bool
        +modifySpan(Span, upstream) const
    }
    class EgressConfigImpl {
        +operationName() Egress
        +verbose() false
        +maxPathTagLength() Default
        +spawnUpstreamSpan() false
        +noContextPropagation() false
        +modifySpan(Span, upstream) no-op
    }
    Config <|.. EgressConfigImpl

    class ConnectionManagerTracingConfig {
        +operation_name_ OperationName
        +custom_tags_ CustomTagMap
        +client_sampling_ FractionalPercent
        +random_sampling_ FractionalPercent
        +overall_sampling_ FractionalPercent
        +operation_ FormatterPtr
        +upstream_operation_ FormatterPtr
        +max_path_tag_length_ uint32
        +verbose_ bool
        +spawn_upstream_span_ bool
        +no_context_propagation_ bool
    }

    %% =========================
    %% Utilities
    %% =========================
    class TracerUtility {
        <<utility>>
        +toString(OperationName) string
        +shouldTraceRequest(StreamInfo) Decision
        +finalizeSpan(Span, StreamInfo, Config, upstream)
    }
    class HttpTracerUtility {
        <<utility>>
        +finalizeDownstreamSpan(Span, req, resp, trail, SI, Config)
        +finalizeUpstreamSpan(Span, SI, Config)
        +onUpstreamResponseHeaders(Span, headers)
        +onUpstreamResponseTrailers(Span, trailers)
    }

    %% =========================
    %% TraceContext (HTTP)
    %% =========================
    class TraceContext {
        <<interface>>
        +protocol() string_view
        +host() string_view
        +path() string_view
        +method() string_view
        +get(key) optional~string_view~
        +set(k, v)
        +remove(k)
        +forEach(callback)
        +requestHeaders() OptRef
    }
    class HttpTraceContextBase~T~ {
        -request_headers_ T&
        +protocol() / host() / path() / method()
        +get/set/remove/forEach
    }
    class HttpTraceContext {
        +set(k, v) → headers_.setCopy
        +remove(k) → headers_.remove
    }
    class ReadOnlyHttpTraceContext {
        +set / remove are no-ops
    }
    TraceContext <|.. HttpTraceContextBase
    HttpTraceContextBase <|-- HttpTraceContext
    HttpTraceContextBase <|-- ReadOnlyHttpTraceContext

    class TraceContextHandler {
        -key_ LowerCaseString
        -handle_ optional~InlineHandle~
        +get(ctx) optional~string_view~
        +getAll(ctx) InlinedVector
        +set(ctx, value)
        +setRefKey(ctx, value)
        +setRef(ctx, value)
        +remove(ctx)
    }

    %% =========================
    %% Custom Tags
    %% =========================
    class CustomTag {
        <<interface>>
        +tag() string_view
        +applySpan(Span, ctx)
        +applyLog(AccessLogCommon, ctx)
    }
    class CustomTagBase {
        -tag_ string
        +applySpan(...)
        +applyLog(...)
        +value(ctx) string_view
    }
    class LiteralCustomTag {
        -value_ string
    }
    class EnvironmentCustomTag {
        -name_ string
        -default_value_ string
        -final_value_ string  (resolved once)
    }
    class RequestHeaderCustomTag {
        -name_ TraceContextHandler
        -header_name_ LowerCaseString
        -default_value_ string
    }
    class MetadataCustomTag {
        -kind_ MetadataKind
        -metadata_key_ MetadataKey
        -default_value_ string
        +metadata(ctx) Metadata*
        +metadataToString(Metadata*) optional~string~
    }
    class FormatterCustomTag {
        -tag_ string
        -formatter_ FormatterPtr
    }
    class CustomTagUtility {
        <<utility>>
        +createCustomTag(proto, command_parsers) CustomTagConstSharedPtr
    }
    CustomTag <|.. CustomTagBase
    CustomTagBase <|-- LiteralCustomTag
    CustomTagBase <|-- EnvironmentCustomTag
    CustomTagBase <|-- RequestHeaderCustomTag
    CustomTagBase <|-- MetadataCustomTag
    CustomTag <|.. FormatterCustomTag
    CustomTagUtility ..> CustomTag : factory

    %% =========================
    %% Singletons of constants
    %% =========================
    class TracingTagValues {
        +Component / HttpUrl / HttpMethod / ...
    }
    class TracingLogValues {
        +FirstUpstreamTxByteSent / LastUpstreamRxByteReceived / ...
    }
    HttpTracerUtility ..> TracingTagValues : Tags::get()
    TracerUtility ..> TracingTagValues : Tags::get()
    HttpTracerUtility ..> TracingLogValues : Logs::get()
```

The folder is essentially **one tree of interfaces** rooted at `Tracer`/`Driver`/`Span`/`TraceContext`/`Config`,
implemented by `TracerImpl`/`NullTracer`/`NullSpan`/`HttpTraceContext`/`(Egress|HCM)Config*`, plus **two helper
families** — utilities (`TracerUtility`, `HttpTracerUtility`, `TraceContextHandler`) and custom tags
(`CustomTagBase` and friends + `CustomTagUtility`).
