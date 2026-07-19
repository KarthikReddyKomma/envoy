# io_uring — class hierarchy (UML)

UML-style Mermaid for the io_uring types. See [`OVERVIEW.md`](OVERVIEW.md) for behavior.

---

## Ring & worker

```mermaid
classDiagram
    class IoUring {
        <<interface>>
        +registerEventfd()* os_fd_t
        +forEveryCompletion(cb)*
        +prepareAccept/Connect/Readv/Writev/Close/Cancel/Shutdown(...)* IoUringResult
        +submit()* IoUringResult
        +injectCompletion(fd, req, result)*
    }
    class ThreadLocalObject { <<interface>> }
    class IoUringImpl {
        -ring_ : io_uring
        -cqes_ : vector
        -event_fd_ : os_fd_t
        -injected_completions_ : list~InjectedCompletion~
    }
    IoUring <|.. IoUringImpl
    ThreadLocalObject <|.. IoUringImpl

    class IoUringWorker {
        <<interface>>
        +addServerSocket(...)* / addClientSocket(...)*
        +submitReadRequest/WriteRequest/...(socket)* Request*
        +dispatcher()* Dispatcher&
    }
    class IoUringWorkerImpl {
        -io_uring_ : IoUringPtr
        -sockets_ : list~IoUringSocketEntryPtr~
        -file_event_ : FileEventPtr (eventfd)
        -dispatcher_ : Dispatcher&
        -delay_submit_ : bool
        -onFileEvent() / submit()
    }
    IoUringWorker <|.. IoUringWorkerImpl
    IoUringWorkerImpl *-- IoUringImpl : owns ring
    IoUringWorkerImpl o-- IoUringSocketEntry : sockets_

    note for IoUringImpl "one ring per thread"
    note for IoUringWorkerImpl "eventfd FileEvent bridges to dispatcher"
```

---

## Socket entry & requests

```mermaid
classDiagram
    class IoUringSocket {
        <<interface>>
        +enableRead() / disableRead() / close(...)
        +onAccept/onConnect/onRead/onWrite/onClose/onCancel/onShutdown(...)*
        +getStatus()* IoUringSocketStatus
    }
    class LinkedObject~T~ { <<mixin>> }
    class DeferredDeletable { <<interface>> }
    class IoUringSocketEntry {
        -fd_ : os_fd_t
        -parent_ : IoUringWorkerImpl&
        -status_ : IoUringSocketStatus
        -injected_completions_ : uint8_t (bitmask)
        #cleanup() / onReadCompleted() / onWriteCompleted() / onRemoteClose()
    }
    IoUringSocket <|.. IoUringSocketEntry
    LinkedObject <|-- IoUringSocketEntry
    DeferredDeletable <|.. IoUringSocketEntry

    class Request {
        +RequestType type()
        +IoUringSocket& socket()
    }
    class RequestType {
        <<enum>>
        Accept Connect Read Write Close Cancel Shutdown
    }
    class ReadRequest { +buf_ +iov_ }
    class WriteRequest { +iov_[] }
    Request <|-- ReadRequest
    Request <|-- WriteRequest
    Request --> RequestType
    Request --> IoUringSocket : belongs to

    note for IoUringSocketEntry "state machine + intrusive list +\nsafe teardown via deferredDelete"
    note for Request "user-data correlation token\nstamped into SQE, returned in CQE"
```

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `IoUringImpl` → `IoUring` + `ThreadLocalObject` | implements | One ring per worker thread. |
| `IoUringWorkerImpl` → `IoUringImpl` | composition | Owns the ring. |
| `IoUringWorkerImpl` → `IoUringSocketEntry` | aggregation | Tracks all sockets (intrusive list). |
| `IoUringWorkerImpl` → `FileEvent` (eventfd) | composition | Bridges completions into the dispatcher. |
| `IoUringSocketEntry` → `IoUringSocket` | implements | Upper-layer-facing socket abstraction. |
| `IoUringSocketEntry` → `DeferredDeletable` | implements | Safe teardown via dispatcher. |
| `ReadRequest`/`WriteRequest` → `Request` | inheritance | Per-op user-data with payload. |
| `Request` → `IoUringSocket` | reference | Routes completion back to its socket. |
