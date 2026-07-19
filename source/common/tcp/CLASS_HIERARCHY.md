# TCP — class hierarchy (UML)

UML-style Mermaid for the TCP pool and async client. See [`OVERVIEW.md`](OVERVIEW.md) for behavior.

---

## The connection pool

```mermaid
classDiagram
    class ConnPoolImplBase {
        <<generic engine>>
        +maybePreconnectImpl(ratio)
        +newPendingStream(ctx, earlyData)
        +drainConnections(behavior) / closeConnections()
        +instantiateActiveClient()* (hook)
        +onPoolReady(...)* / onPoolFailure(...)* (hooks)
    }
    class Instance {
        <<interface (Tcp::ConnectionPool)>>
        +newConnection(callbacks)*
        +drainConnections(...)* / closeConnections()*
    }
    class ConnPoolImpl {
        -idle_timeout_
        +newConnection(cb)
        +instantiateActiveClient() ActiveClientPtr
        +onPoolReady(...) / onPoolFailure(...)
    }
    ConnPoolImplBase <|-- ConnPoolImpl
    Instance <|.. ConnPoolImpl
    note for ConnPoolImpl "reuses HTTP pool engine;\noverrides only TCP specifics"
```

```mermaid
classDiagram
    class ActiveClient { <<generic, from base>> }
    class ConnectionCallbacks { <<interface (Network)>> }
    class ActiveTcpClient {
        -connection_ : ClientConnectionPtr
        -callbacks_ : UpstreamCallbacks*
        -read_filter_handle_ : ConnReadFilter
        -idle_timer_ : TimerPtr
        +onEvent(event) / onUpstreamData(...)
        +setIdleTimer() / onIdleTimeout()
        +clearCallbacks()
    }
    class TcpConnectionData {
        -parent_ : ActiveTcpClient*
        -connection_ : ClientConnection&
        +connection() / addUpstreamCallbacks(cb)
        +setConnectionState(...) / release()
    }
    class ConnReadFilter { onData → onUpstreamData }

    ActiveClient <|-- ActiveTcpClient
    ConnectionCallbacks <|.. ActiveTcpClient
    ActiveTcpClient *-- ConnReadFilter
    ActiveTcpClient *-- TcpConnectionData : hands out
    note for TcpConnectionData "caller's checkout handle;\nnull-checks parent_ (unordered teardown)"
```

---

## Pending stream & attach context

```mermaid
classDiagram
    class PendingStream { <<generic, from base>> }
    class AttachContext { <<generic, from base>> }
    class TcpPendingStream {
        -context_ : TcpAttachContext
    }
    class TcpAttachContext {
        +callbacks_ : Tcp::ConnectionPool::Callbacks*
    }
    PendingStream <|-- TcpPendingStream
    AttachContext <|-- TcpAttachContext
    TcpPendingStream *-- TcpAttachContext
    note for TcpAttachContext "carries caller callbacks\nthrough the generic queue"
```

---

## The async client

```mermaid
classDiagram
    class AsyncTcpClient { <<interface>> }
    class ConnectionCallbacks { <<interface (Network)>> }
    class AsyncTcpClientImpl {
        -connection_ : ClientConnectionPtr
        -thread_local_cluster_ : ThreadLocalCluster&
        -connect_timer_ : TimerPtr
        -callbacks_ : AsyncTcpClientCallbacks*
        -connected_ : bool
        -detected_close_ : DetectedCloseType
        +connect() / write(data, end) / close(type)
        +onConnectTimeout() / onEvent(event) / onData(...)
    }
    class NetworkReadFilter { onData → parent.onData }

    AsyncTcpClient <|.. AsyncTcpClientImpl
    ConnectionCallbacks <|.. AsyncTcpClientImpl
    AsyncTcpClientImpl *-- NetworkReadFilter
    note for AsyncTcpClientImpl "single connection +\nconnect timeout; no pooling"
```

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `ConnPoolImpl` → `ConnPoolImplBase` | inheritance | Reuses the shared pool engine. |
| `ConnPoolImpl` → `Tcp::ConnectionPool::Instance` | implements | The public pool interface. |
| `ActiveTcpClient` → `ActiveClient` + `ConnectionCallbacks` | inheritance | A pooled connection that hears its events. |
| `ActiveTcpClient` → `TcpConnectionData` | hands out | The caller's checkout handle. |
| `TcpPendingStream`/`TcpAttachContext` → base types | inheritance | Carry TCP callbacks through the queue. |
| `AsyncTcpClientImpl` → `AsyncTcpClient` | implements | Standalone single-connection client. |
| both → `NetworkReadFilter`/`ConnReadFilter` | composition | Forward upstream `onData` to callbacks. |
| both → `ThreadLocalCluster` | uses | Host selection via the cluster LB. |
