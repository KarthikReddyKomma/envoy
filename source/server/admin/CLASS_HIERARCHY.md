# Admin Console — Class Hierarchy

UML-style class diagrams for the admin subsystem. Documentation aids, not exhaustive.

## 1. `AdminImpl` and the interfaces it implements

```mermaid
classDiagram
    class Admin {
        <<interface>>
        +addHandler(...) bool
        +addStreamingHandler(...) bool
        +removeHandler(prefix) bool
        +startHttpListener(...) void
        +socket() Socket&
        +request(path, method, headers, body) Code
        +getConfigTracker() ConfigTracker&
    }
    class ConnectionManagerConfig { <<interface>> }
    class FilterChainManager { <<interface>> }
    class FilterChainFactory_Net { <<interface>> }
    class FilterChainFactory_Http { <<interface>> }

    class AdminImpl {
        -handlers_ : list~UrlHandler~
        -listener_ : AdminListenerPtr
        -allowlisted_paths_ : set
        -clusters_handler_ : ClustersHandler
        -config_dump_handler_ : ConfigDumpHandler
        -stats_handler_ : StatsHandler
        -server_info_handler_ : ServerInfoHandler
        ...other handlers...
        +makeRequest(AdminStream&) RequestPtr
        +createNetworkFilterChain(conn) bool
        +createFilterChain(callbacks) bool
        +createCodec(...) ServerConnectionPtr
    }

    Admin <|-- AdminImpl
    ConnectionManagerConfig <|-- AdminImpl
    FilterChainManager <|-- AdminImpl
    FilterChainFactory_Net <|-- AdminImpl
    FilterChainFactory_Http <|-- AdminImpl
```

## 2. The request / stream types

```mermaid
classDiagram
    class Request {
        <<interface>>
        +start(ResponseHeaderMap&) Code
        +nextChunk(Buffer&) bool
    }
    class StaticTextRequest {
        -response_text_
        +start(...) Code
        +nextChunk(...) bool
    }
    class RequestGasket {
        -callback_ : HandlerCb
        -response_ : Buffer
        +makeGen(HandlerCb)$ GenRequestFn
    }
    class StatsRequest {
        -stat_map_ : btree_map
        -phase_ : Phase
        -chunk_size_ : uint64 = 2MB
        -render_ : StatsRenderPtr
        +start(...) Code
        +nextChunk(...) bool
    }

    Request <|-- StaticTextRequest
    Request <|-- RequestGasket
    Request <|-- StatsRequest

    class AdminStream {
        <<interface>>
        +getRequestHeaders() RequestHeaderMap&
        +getRequestBody() Buffer*
        +queryParams() QueryParams
        +setEndStreamOnComplete(bool) void
        +addOnDestroyCallback(fn) void
    }
    class AdminFilter {
        -admin_ : AdminImpl&
        -request_headers_
        +decodeData(...) FilterDataStatus
        +onComplete() void
    }
    AdminStream <|-- AdminFilter
    AdminFilter ..> Request : drives start/nextChunk
```

## 3. The listener wiring (nested in `AdminImpl`)

```mermaid
classDiagram
    class AdminListener {
        <<ListenerConfig>>
        +filterChainManager() FilterChainManager&
        +filterChainFactory() FilterChainFactory&
        +shouldBypassOverloadManager() bool = true
    }
    class AdminListenSocketFactory
    class AdminFilterChain {
        +networkFilterFactories() : [HCM]
    }

    AdminImpl *-- AdminListener
    AdminListener *-- AdminListenSocketFactory
    AdminImpl ..> AdminFilterChain : provides single chain
    AdminListener ..> AdminImpl : filterChainManager = AdminImpl
```

## 4. Handler classes

All endpoint handlers derive from `HandlerContextBase` (holds `Server::Instance& server_`)
and are owned as direct members of `AdminImpl`.

```mermaid
classDiagram
    class HandlerContextBase {
        #server_ : Instance&
    }
    class StatsHandler
    class ClustersHandler
    class ConfigDumpHandler
    class InitDumpHandler
    class ServerInfoHandler
    class ServerCmdHandler
    class LogsHandler
    class RuntimeHandler
    class ListenersHandler

    HandlerContextBase <|-- StatsHandler
    HandlerContextBase <|-- ClustersHandler
    HandlerContextBase <|-- ConfigDumpHandler
    HandlerContextBase <|-- InitDumpHandler
    HandlerContextBase <|-- ServerInfoHandler
    HandlerContextBase <|-- ServerCmdHandler
    HandlerContextBase <|-- LogsHandler
    HandlerContextBase <|-- RuntimeHandler
    HandlerContextBase <|-- ListenersHandler
```

| Handler | Endpoint(s) |
|---------|-------------|
| `StatsHandler` | `/stats`, `/stats/prometheus`, `/stats/recentlookups*`, `/reset_counters`, `/contention` |
| `ClustersHandler` | `/clusters` |
| `ConfigDumpHandler` | `/config_dump` |
| `InitDumpHandler` | `/init_dump` |
| `ServerInfoHandler` | `/server_info`, `/ready`, `/certs`, `/hot_restart_version`, `/memory` |
| `ServerCmdHandler` | `/quitquitquit`, `/healthcheck/fail`, `/healthcheck/ok` (POST) |
| `LogsHandler` | `/logging` (POST), `/reopen_logs` (POST) |
| `RuntimeHandler` | `/runtime`, `/runtime_modify` (POST) |
| `ListenersHandler` | `/listeners`, `/drain_listeners` (POST) |
| `ProfilingHandler` / `TcmallocProfilingHandler` | `/cpuprofiler`, `/heapprofiler`, `/heap_dump`, `/allocprofiler` |

## 5. Stats rendering

```mermaid
classDiagram
    class StatsRender {
        <<interface>>
        +generate(...) void
        +finalize(Buffer&) void
    }
    class StatsTextRender
    class StatsJsonRender
    class StatsHtmlRender
    class PrometheusStatsFormatter

    StatsRender <|-- StatsTextRender
    StatsRender <|-- StatsJsonRender
    StatsRender <|-- StatsHtmlRender
    StatsRequest ..> StatsRender : uses
```

## 6. Out-of-network responses

```mermaid
classDiagram
    class AdminResponse {
        -opt_admin_ : Admin*
        -request_ : RequestPtr
        -response_headers_
        -ptr_set_ : shared_ptr~PtrSet~
        +getHeaders(HeadersFn) void
        +nextChunk(BodyFn) void
        +cancel() void
        +cancelled() bool
    }
    AdminResponse ..> AdminFilter : builds on main thread
    AdminResponse ..> Admin : makeRequest()
```

`AdminResponse` bridges the synchronous, main-thread admin machinery to a thread-safe,
flow-controlled, cancellable API usable from any thread. The shared `PtrSet` allows the
server to terminate all in-flight responses on shutdown.
