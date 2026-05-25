# Overview, Part 4 — Streams, codecs, and HTTP/3 integration

> *Read [`OVERVIEW_PART1`](OVERVIEW_PART1_architecture_and_layering.md), [`OVERVIEW_PART2`](OVERVIEW_PART2_listener_session_connection.md), [`OVERVIEW_PART3`](OVERVIEW_PART3_crypto_and_tls.md) first.*

By this point the server has accepted a session, the client has connected and the TLS handshake is complete. This document covers what happens **per stream / per request**: how QUIC stream events become HTTP requests and how HTTP responses become QUIC stream writes.

## Three layers, one request

```mermaid
flowchart TB
  subgraph App["L6 — application"]
    R["Router / filter chain"]
  end
  subgraph HCM["L6 — HCM"]
    H["ConnectionManagerImpl"]
  end
  subgraph Codec["L5 — HTTP/3 codec adapters"]
    SCDC["QuicHttpServerConnectionImpl"]
    CCDC["QuicHttpClientConnectionImpl"]
  end
  subgraph Streams["L4 — streams"]
    SSTR["EnvoyQuicServerStream<br/>(Http::ResponseEncoder)"]
    CSTR["EnvoyQuicClientStream<br/>(Http::RequestEncoder)"]
  end
  subgraph Sessions["L3 — sessions"]
    SSESS["EnvoyQuicServerSession"]
    CSESS["EnvoyQuicClientSession"]
  end

  R -->|"router/filters"| H
  H <--> SCDC
  H <--> CCDC
  SCDC <--> SSESS
  CCDC <--> CSESS
  SSESS -->|"CreateIncomingStream"| SSTR
  CSESS -->|"CreateClientStream"| CSTR
  SSTR <--> SSESS
  CSTR <--> CSESS
```

The codec adapters at L5 are intentionally **thin** — most of the heavy lifting is in the stream classes at L4. The codec mostly exists to satisfy the `Http::Connection` interface (`goAway`, `wantsToWrite`, watermark forwarding) so HCM and the conn pool can drive the connection without knowing about QUIC.

---

## Stream class diagram

```mermaid
classDiagram
    class quic_QuicSpdyStream {
        <<QUICHE>>
        +OnBodyAvailable()
        +OnInitialHeadersComplete()
        +OnTrailingHeadersComplete()
        +OnClose()
        +OnCanWrite()
        +WriteHeaders()
        +WriteOrBufferBody()
    }
    class quic_QuicSpdyServerStreamBase {
        <<QUICHE>>
    }
    class quic_QuicSpdyClientStream {
        <<QUICHE>>
    }
    class Http_StreamEncoder {
        <<Envoy>>
        +encodeData()
        +encodeMetadata()
    }
    class Http_RequestEncoder {
        <<Envoy>>
        +encodeHeaders(req)
        +encodeTrailers(req)
    }
    class Http_ResponseEncoder {
        <<Envoy>>
        +encode1xxHeaders()
        +encodeHeaders(resp)
        +encodeTrailers(resp)
    }
    class Http_MultiplexedStreamImplBase {
        <<Envoy>>
        +readDisable()
        +addCallbacks()
        +bufferLimit()
    }
    class SendBufferMonitor {
        +updateBytesBuffered()
    }
    class EnvoyQuicStream {
        +encodeData()
        +readDisable()
        +validateHeader()
        #send_buffer_simulation_
        #stats_gatherer_
    }
    class EnvoyQuicServerStream {
        +setRequestDecoder()
        +encodeHeaders(resp,fin)
    }
    class EnvoyQuicClientStream {
        +setResponseDecoder()
        +encodeHeaders(req,fin)
    }

    quic_QuicSpdyStream <|-- quic_QuicSpdyServerStreamBase
    quic_QuicSpdyStream <|-- quic_QuicSpdyClientStream
    Http_StreamEncoder <|-- Http_RequestEncoder
    Http_StreamEncoder <|-- Http_ResponseEncoder
    Http_StreamEncoder <|-- EnvoyQuicStream
    Http_MultiplexedStreamImplBase <|-- EnvoyQuicStream
    SendBufferMonitor <|-- EnvoyQuicStream
    EnvoyQuicStream <|-- EnvoyQuicServerStream
    quic_QuicSpdyServerStreamBase <|-- EnvoyQuicServerStream
    Http_ResponseEncoder <|-- EnvoyQuicServerStream
    EnvoyQuicStream <|-- EnvoyQuicClientStream
    quic_QuicSpdyClientStream <|-- EnvoyQuicClientStream
    Http_RequestEncoder <|-- EnvoyQuicClientStream
```

The detailed class graph including all base classes lives in [`CLASS_HIERARCHY.md`](CLASS_HIERARCHY.md). The take‑away here: each stream is **both** a `QuicSpdyStream` (so QUICHE can hand it frames) **and** a Stream/Encoder (so HCM can read its requests / write its responses).

---

## Server side — request flow

### Receiving a request

```mermaid
sequenceDiagram
  autonumber
  participant Cn as EnvoyQuicServerConnection
  participant Sess as EnvoyQuicServerSession
  participant Str as EnvoyQuicServerStream
  participant Cdc as QuicHttpServerConnectionImpl
  participant HCM as ConnectionManagerImpl
  participant Dec as Http::RequestDecoder

  Cn->>Sess: OnStreamFrame(frame{stream_id, data})
  alt new stream id
    Sess->>Sess: CreateIncomingStream(id)
    Sess->>Str: new EnvoyQuicServerStream(id, this, BIDIRECTIONAL, stats, ...)
    Sess->>Sess: ActivateStream(unique_ptr)
    Sess->>Sess: setUpRequestDecoder(*stream)
    Sess->>Cdc: http_connection_callbacks_->newStream(*stream)
    Cdc-->>Sess: Http::RequestDecoder& decoder
    Sess->>Str: setRequestDecoder(decoder)
  end
  Str->>Str: OnStreamFrame(frame)  // QPACK incrementally

  Note over Str: When QPACK has full :method/:scheme/:authority/:path
  Str->>Str: OnInitialHeadersComplete(fin, frame_len, header_list)
  Str->>Dec: decodeHeaders(header_map, end_stream)
  Dec->>HCM: route, run filter chain, create stream

  Note over Cn,Str: As body frames arrive
  Str->>Str: OnBodyAvailable()
  Str->>Dec: decodeData(buffer, end_stream)

  alt trailers present
    Str->>Str: OnTrailingHeadersComplete(fin, ...)
    Str->>Dec: decodeTrailers(trailer_map)
  end
```

### Sending a response

```mermaid
sequenceDiagram
  autonumber
  participant HCM as ConnectionManagerImpl
  participant Str as EnvoyQuicServerStream
  participant Q as quic::QuicSpdyServerStreamBase
  participant Cn as EnvoyQuicServerConnection
  participant PW as EnvoyQuicPacketWriter

  HCM->>Str: encodeHeaders(headers, end_stream)
  Str->>Str: addDecompressedHeaderBytesSent(headers)
  Str->>Q: WriteHeaders(http_header_block, fin, listener)
  Q->>Cn: enqueue STREAM/HEADERS frames
  Cn->>PW: WritePacket() (paced)

  loop body chunks
    HCM->>Str: encodeData(data, end_stream)
    Str->>Q: WriteOrBufferBody(data, fin)
    Note over Str: ScopedWatermarkBufferUpdater<br/>tracks buffered bytes change
    Str->>Str: updateBytesBuffered(old, new)
    alt above high
      Str->>HCM: onAboveWriteBufferHighWatermark()
    end
  end

  opt trailers
    HCM->>Str: encodeTrailers(trailers)
    Str->>Q: WriteTrailers(...)
  end

  Note over Cn: Once response FIN is acked,<br/>QuicStatsGatherer flushes access logs
```

### Deferred access logging

QUIC access logs are emitted **after the FIN is acked** so logged byte counts and RTTs reflect what actually reached the peer. The mechanism:

1. `EnvoyQuicServerStream::setDeferredLoggingHeadersAndTrailers()` (called by HCM at end of response) hands a copy of headers/trailers/stream_info to `QuicStatsGatherer`.
2. `QuicStatsGatherer` is also installed as a `quic::QuicAckListenerInterface` for the stream's writes, so QUICHE calls `OnPacketAcked()` for every chunk that gets acked.
3. When all outstanding bytes are acked **and** FIN has been sent, `maybeDoDeferredLog()` walks `access_log_handlers_` and emits the logs.

---

## Client side — request flow

The mirror picture:

```mermaid
sequenceDiagram
  autonumber
  participant Pool as Http::ConnectionPool
  participant Cdc as QuicHttpClientConnectionImpl
  participant Sess as EnvoyQuicClientSession
  participant Str as EnvoyQuicClientStream
  participant Dec as Http::ResponseDecoder

  Pool->>Cdc: newStream(decoder)
  Cdc->>Sess: CreateOutgoingBidirectionalStream()
  Sess->>Str: new EnvoyQuicClientStream(id, this, BIDIRECTIONAL, ...)
  Cdc->>Str: setResponseDecoder(decoder)
  Cdc-->>Pool: RequestEncoder& = *Str

  Pool->>Str: encodeHeaders(req, end_stream)
  Str->>Sess: WriteHeaders(http_header_block, fin, listener)
  loop body
    Pool->>Str: encodeData(data, end_stream)
    Str->>Sess: WriteOrBufferBody(data, fin)
  end

  Note over Sess: Peer sends response on the same stream

  Str->>Str: OnInitialHeadersComplete(fin, len, header_list)
  Str->>Dec: decodeHeaders(headers, end_stream)
  Str->>Str: OnBodyAvailable()
  Str->>Dec: decodeData(buffer, end_stream)
  opt trailers
    Str->>Str: OnTrailingHeadersComplete(...)
    Str->>Dec: decodeTrailers(trailers)
  end
```

Differences from server side:

- The stream is **outgoing‑bidirectional**, allocated up‑front from the pool, not in response to an incoming frame.
- `ShouldCreateOutgoingBidirectionalStream()` is overridden to always return `true` — the conn pool already gated stream creation, so re‑gating here would crash.
- The client side handles `OnHttp3GoAway(stream_id)`: any stream id ≥ the limit must not be issued again; the pool is told via `Http::ConnectionCallbacks::onGoAway`.

---

## Read‑disable and back‑pressure (per stream)

Envoy filters can call `readDisable(true)` to stop reading body bytes (e.g. when a downstream is slow). On QUIC streams this becomes flow‑control stop, not a TCP backpressure thing. The trick is QUICHE doesn't tolerate stream‑state changes from inside its own callstack, so the change is **scheduled** for the next event loop:

```mermaid
sequenceDiagram
  participant F as Filter
  participant Str as EnvoyQuicStream
  participant CB as SchedulableCallback
  participant D as Event::Dispatcher

  F->>Str: readDisable(true)
  Str->>Str: ++read_disable_counter_
  Str->>CB: scheduleCallbackNextIteration()
  Note over Str,D: returns immediately
  D->>CB: fire next iteration
  CB->>Str: switchStreamBlockState()
  Str->>Str: read_disable_counter_ > 0 ? block : unblock
```

`read_disable_counter_` lets multiple `readDisable(true)` calls compose; only the count‑to‑zero transition actually unblocks.

---

## Header validation and pseudo‑header order

`EnvoyQuicStream::validateHeader()` is called for every header before it reaches HCM. It:

1. Runs `http2::adapter::HeaderValidator::ValidateSingleHeader()` — checks for forbidden chars, well‑formed names, etc.
2. Validates `content-length` (`Http::HeaderUtility::validateContentLength`) and stashes the parsed value so `updateReceivedContentBytes()` can sanity‑check the body length on stream end.
3. Enforces "all pseudo‑headers before any regular header".
4. Returns `REJECT` to make QUICHE skip the stream and call `onStreamError()`.

The connection‑close vs stream‑reset behaviour on a bad header is configurable via `http3_options.override_stream_error_on_invalid_http_message`.

---

## Codec adapters — what they actually do

`QuicHttpServerConnectionImpl` and `QuicHttpClientConnectionImpl` look thin because they are. They mostly:

- Implement `Http::Connection::protocol()` → `Protocol::Http3`.
- Implement `Http::Connection::dispatch()` as `PANIC` — QUIC hands data directly to streams, the codec never sees raw bytes.
- Implement `wantsToWrite()` by asking the session if any bytes are buffered (`bytesToSend() > 0`).
- Implement `goAway()` by calling `quic_session_.OnGoAway()` (server) or sending the appropriate H/3 GOAWAY frame.
- Forward `onUnderlyingConnectionAboveWriteBufferHighWatermark` / `Below` to the session.

The class hierarchy:

```mermaid
classDiagram
    class Http_Connection {
        <<Envoy>>
        +dispatch(buf) Status
        +protocol() Protocol
        +wantsToWrite() bool
        +goAway()
    }
    class Http_ServerConnection {
        <<Envoy>>
        +shutdownNotice()
    }
    class Http_ClientConnection {
        <<Envoy>>
        +newStream(decoder) RequestEncoder&
    }
    class QuicHttpConnectionImplBase {
        +dispatch() panic
        +protocol() Http3
        +wantsToWrite()
        #quic_session_
    }
    class QuicHttpServerConnectionImpl {
        +goAway()
        +shutdownNotice()
    }
    class QuicHttpClientConnectionImpl {
        +newStream(decoder)
        +goAway()
    }

    Http_Connection <|-- Http_ServerConnection
    Http_Connection <|-- Http_ClientConnection
    Http_Connection <|-- QuicHttpConnectionImplBase
    QuicHttpConnectionImplBase <|-- QuicHttpServerConnectionImpl
    Http_ServerConnection <|-- QuicHttpServerConnectionImpl
    QuicHttpConnectionImplBase <|-- QuicHttpClientConnectionImpl
    Http_ClientConnection <|-- QuicHttpClientConnectionImpl
```

---

## HTTP/3 Datagrams and Capsule Protocol (optional)

When `ENVOY_ENABLE_HTTP_DATAGRAMS` is compiled in, `EnvoyQuicStream` can hold a `HttpDatagramHandler` that maps:

- Outgoing data passed to `encodeData()` is interpreted as **capsules** (RFC 9292) and sent over both stream bytes and QUIC DATAGRAM frames (RFC 9297).
- Incoming HTTP/3 datagrams arriving via the QUIC connection are turned into capsules and surfaced to filters as body.

This is the foundation for MASQUE / CONNECT‑UDP / WebTransport. The Envoy code in this folder is just the bridge; the actual MASQUE / WebTransport extensions live elsewhere.

---

## Watermark / back‑pressure flow end‑to‑end

```mermaid
flowchart LR
  subgraph PerStream["Per stream"]
    A1["encodeData"] --> A2["WriteOrBufferBody"]
    A2 --> A3["QUICHE per-stream send buffer grows"]
    A3 --> A4["ScopedWatermarkBufferUpdater dtor"]
    A4 --> A5["updateBytesBuffered(old, new)"]
    A5 --> A6["send_buffer_simulation_<br/>(per-stream high/low watermarks)"]
    A6 -->|"crossed high"| HCM_str["filter: onAboveWriteBufferHighWatermark()"]
  end
  A5 --> B1["QuicFilterManagerConnectionImpl::<br/>updateBytesBuffered"]
  subgraph PerConn["Per connection (aggregated)"]
    B1 --> B2["write_buffer_watermark_simulation_"]
    B2 -->|"crossed high"| HCM_conn["onSendBufferHighWatermark<br/>-> Network::ConnectionCallbacks"]
  end
```

Two independent watermark state machines (per stream + per connection), one set of `SendBufferMonitor::ScopedWatermarkBufferUpdater` measurements, no double‑counting.

---

## What's next

- [`envoy_quic_stream.md`](envoy_quic_stream.md) — Per‑file deep dive for both stream classes and the codec adapters.
- [`envoy_quic_session.md`](envoy_quic_session.md) — Where the streams are *owned*; the session's role in stream lifecycle.
- [`CLASS_HIERARCHY.md`](CLASS_HIERARCHY.md) — UML view of the full stream / session / codec graph.
