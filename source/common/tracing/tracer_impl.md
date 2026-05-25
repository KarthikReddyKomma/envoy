# `tracer_impl.{h,cc}` + `null_span_impl.h`

These three files define the **protocol-neutral runtime** of Envoy tracing:

- **`TracerImpl`** — concrete `Tracer` that owns a `DriverSharedPtr` and a `LocalInfo&`.
- **`NullTracer`** — `Tracer` that returns `NullSpan` for everything (the "no driver" path).
- **`NullSpan`** — a no-op `Span` implementation.
- **`EgressConfigImpl`** + `EgressConfig` const-singleton — the default tracing `Config` used by async clients
  (gRPC), so they can record egress spans without a per-listener tracing config.
- **`TracerUtility`** — static helpers: operation-name → string, sampling decision, protocol-neutral tag
  population in `finalizeSpan`.

---

## `TracerImpl` — what `startSpan` actually does

```cpp
SpanPtr TracerImpl::startSpan(const Config& config,
                              TraceContext& trace_context,
                              const StreamInfo::StreamInfo& stream_info,
                              const Tracing::Decision tracing_decision) {
  std::string span_name = TracerUtility::toString(config.operationName()); // "ingress" or "egress"

  if (config.operationName() == OperationName::Egress) {
    span_name.append(" ");
    span_name.append(std::string(trace_context.host()));   // "egress example.com"
  }

  SpanPtr active_span =
      driver_->startSpan(config, trace_context, stream_info, span_name, tracing_decision);

  if (active_span) {
    active_span->setTag(Tracing::Tags::get().NodeId, local_info_.nodeName());
    active_span->setTag(Tracing::Tags::get().Zone,   local_info_.zoneName());
  }
  return active_span;
}
```

### Operation name

- `Ingress` → `"ingress"` (HCM uses this for the downstream span).
- `Egress`  → `"egress <host>"` where `<host>` is `trace_context.host()`, which (for HTTP) is the value of the
  `Host`/`:authority` header. This is the *upstream* span name spawned by the router.

### Universal tags

`node_id` and `zone` are stamped on every span regardless of driver. They come from the static `LocalInfo`
captured at server startup, so they're constant for the process and free to read.

### `Tracing::Decision`

Passed straight through to the driver. A driver may *override* the decision (e.g., Zipkin honoring an explicit
`x-b3-sampled: 1` header), but that's its business.

### The driver returning `nullptr`

Some drivers return `nullptr` when they decide not to trace at all (e.g., upstream context says "don't trace" and
the driver is `noPropagateOnly`). `TracerImpl::startSpan` returns `nullptr` in that case — callers must `if
(span)` before dereferencing. HCM and the router both do.

---

## `NullTracer` and `NullSpan` — the disabled path

```cpp
class NullTracer : public Tracer {
  SpanPtr startSpan(const Config&, TraceContext&, const StreamInfo::StreamInfo&,
                    Tracing::Decision) override {
    return SpanPtr{new NullSpan()};
  }
};
```

`NullSpan` implements every `Span` method as a no-op. Notable:

| Method                                | Return                                         |
|---------------------------------------|------------------------------------------------|
| `setOperation`, `setTag`, `log`       | no-op                                          |
| `finishSpan`                          | no-op                                          |
| `injectContext`                       | no-op (no headers added — propagation off)     |
| `setBaggage`, `getBaggage`            | no-op / empty                                  |
| `getSpanId`, `getTraceId`             | empty string                                   |
| `spawnChild`                          | returns another `new NullSpan()`               |
| `setSampled`                          | no-op                                          |
| `exportedSpan` / `useLocalDecision`   | `false`                                        |

Two important behavioural effects:

1. **`HttpTracerUtility::finalizeDownstreamSpan`** checks `if (!span.exportedSpan()) { span.finishSpan(); return; }`
   as an early-exit, so when given a `NullSpan` it skips the entire tag-population block. Zero cost when tracing
   is disabled.

2. **`spawnChild`** returns *another* `NullSpan`, not the same instance. There's no leak — each `new NullSpan()`
   is owned by the returned `SpanPtr` — but it would be sub-optimal if it were a hot path. It isn't: when the
   tracer is `NullTracer` the codepath that calls `spawnChild` is usually short-circuited upstream by
   `exportedSpan()` checks.

`NullSpan::instance()` exists as a static singleton for places that need to *reference* a span (`Span&`) without
creating one; it is not what `NullTracer::startSpan` returns (that returns a freshly heap-allocated one because
the caller takes ownership of a `SpanPtr`).

---

## `EgressConfigImpl` and `EgressConfig`

```cpp
class EgressConfigImpl : public Config {
  Tracing::OperationName operationName() const override { return Tracing::OperationName::Egress; }
  void modifySpan(Tracing::Span&, bool) const override {}
  bool verbose() const override { return false; }
  uint32_t maxPathTagLength() const override { return Tracing::DefaultMaxPathTagLength; }
  bool spawnUpstreamSpan() const override { return false; }
  bool noContextPropagation() const override { return false; }
};
using EgressConfig = ConstSingleton<EgressConfigImpl>;
```

This is the **default `Config`** used by anything that wants to start a span without inventing its own tracing
config — primarily the **gRPC async client** (`Grpc::AsyncClientImpl::sendRaw`). It says:

- This is an egress span (operation name will be `"egress <host>"`).
- No verbose timing logs.
- No upstream child span (the async client itself *is* the upstream).
- Default path-tag truncation length.
- Context propagation **on** (async clients always propagate so the downstream of the called service can
  continue the trace).
- `modifySpan` does nothing — no per-config tag mutation.

A single immutable instance lives for the life of the process.

---

## `TracerUtility::shouldTraceRequest` in detail

```cpp
Decision TracerUtility::shouldTraceRequest(const StreamInfo::StreamInfo& stream_info) {
  if (stream_info.healthCheck()) {                       // never trace health-checks
    return {Reason::HealthCheck, false};
  }
  const Tracing::Reason trace_reason = stream_info.traceReason();
  switch (trace_reason) {
    case Reason::ClientForced:
    case Reason::ServiceForced:
    case Reason::Sampling:
      return {trace_reason, true};                       // sampled
    default:
      return {trace_reason, false};                      // not sampled (Default, NotTraceable, …)
  }
}
```

The `traceReason()` value was set earlier (typically by `Http::ConnectionManagerUtility::mutateRequestHeaders`)
according to:

- `x-client-trace-id` presence → `Reason::ClientForced`
- runtime/config force-trace → `Reason::ServiceForced`
- `client_sampling` / `random_sampling` / `overall_sampling` math → `Reason::Sampling` (or `NotTraceable`)

So `shouldTraceRequest` is essentially a **decoder**, not a sampler. The sampling math runs once at request
arrival; the rest of the pipeline reads the cached decision.

---

## `TracerUtility::finalizeSpan` (protocol-neutral)

Used by the **TCP proxy** tracing path (no HTTP request headers) and by anything else that wants the canonical
tag set without HTTP-specific tags:

```cpp
span.setTag(Tags::Component, Tags::Proxy);
span.setTag(Tags::ResponseFlags, ResponseFlagUtils::toShortString(stream_info));
if (!upstream_span)
  span.setTag(Tags::PeerAddress, downstream_direct_remote);
if (cluster_info) {
  span.setTag(Tags::UpstreamCluster, cluster_info->name());
  span.setTag(Tags::UpstreamClusterName, cluster_info->observabilityName());
}
if (upstream_info && upstream_host) {
  span.setTag(Tags::UpstreamAddress, upstream_host->address()->asStringView());
  if (upstream_span)
    span.setTag(Tags::PeerAddress, upstream_host->address()->asStringView());
}
if (tracing_config.verbose()) annotateVerbose(span, stream_info);
tracing_config.modifySpan(span, upstream_span);
span.finishSpan();
```

The HTTP-specific twin in `http_tracer_impl.cc::HttpTracerUtility::setCommonTags` adds the HTTP-flavoured tags
on top of this same skeleton.

---

## Common testing patterns

- `TracerImpl::driverForTest()` returns the `DriverSharedPtr`. Test fixtures usually inject a
  `MockTracingDriver` and assert which `Span` calls were made.
- `NullTracer` is the easiest way to satisfy "I need a tracer to construct this thing in a unit test, but I
  don't care about its behaviour".
- Custom drivers in extensions are tested by passing them to `TracerImpl(driver, local_info)` and exercising
  `startSpan`.

---

## Gotchas

1. **`startSpan` may return `nullptr`** — drivers can opt out. Always `if (span)` before using.
2. **`operationName()` is `Ingress` for both the downstream span *and* the spawned upstream span when
   `spawnUpstreamSpan()` is on** — only the *downstream* `Config` is `Ingress`; the upstream child gets a
   separate `Config` whose `operationName` is `Egress`. (This is wired by HCM/router, not by anything in this
   folder.)
3. **`node_id` and `zone` are set after the driver creates the span.** If the driver wants to override them
   based on incoming trace headers, it must do so during `driver_->startSpan(...)`; our `setTag` calls happen
   *after* and will win on duplicate keys.
4. **`finalizeSpan` does `finishSpan` unconditionally.** Don't call it twice on the same span.
