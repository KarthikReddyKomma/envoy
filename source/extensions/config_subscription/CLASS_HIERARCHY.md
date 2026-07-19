# Config Subscription — Class Hierarchy

UML-style class diagrams for the xDS config subscription extensions. Documentation aids, not
exhaustive.

## 1. Core interfaces

```mermaid
classDiagram
    class Subscription {
        <<interface>>
        +start(names) void
        +updateResourceInterest(names) void
        +requestOnDemandUpdate(add) void
    }
    class SubscriptionCallbacks {
        <<interface>>
        +onConfigUpdate(resources, version) Status
        +onConfigUpdate(added, removed, sys_version) Status
        +onConfigUpdateFailed(reason, e) void
    }
    class OpaqueResourceDecoder {
        <<interface>>
        +decodeResource(any) DecodedResource
    }
    class ConfigSubscriptionFactory {
        <<interface>>
        +create(data) SubscriptionPtr
        +name() string
    }

    Subscription ..> SubscriptionCallbacks : delivers updates
    SubscriptionCallbacks ..> OpaqueResourceDecoder : uses
    ConfigSubscriptionFactory ..> Subscription : creates
```

## 2. The transport implementations

```mermaid
classDiagram
    class Subscription { <<interface>> }
    class GrpcSubscriptionImpl
    class GrpcCollectionSubscriptionImpl
    class HttpSubscriptionImpl
    class FilesystemSubscriptionImpl
    class FilesystemCollectionSubscriptionImpl

    Subscription <|-- GrpcSubscriptionImpl
    GrpcSubscriptionImpl <|-- GrpcCollectionSubscriptionImpl
    Subscription <|-- HttpSubscriptionImpl
    Subscription <|-- FilesystemSubscriptionImpl
    FilesystemSubscriptionImpl <|-- FilesystemCollectionSubscriptionImpl
```

## 3. gRPC class layering

```mermaid
classDiagram
    class GrpcSubscriptionImpl {
        -grpc_mux_ : GrpcMuxSharedPtr
        -watch_ : GrpcMuxWatchPtr
        -is_aggregated_ : bool
        +start(names) void
    }
    class GrpcMux {
        <<interface>>
        +addWatch(type_url, names, cb, decoder) GrpcMuxWatchPtr
        +start() void
        +pause(type_url) ScopedResume
    }
    class GrpcMuxImpl {
        note: legacy SotW ADS<br/>iterates ApiState.watches_
    }
    class NewGrpcMuxImpl {
        note: legacy Delta<br/>uses WatchMap
    }
    class GrpcMuxSotw {
        note: unified SotW
    }
    class GrpcMuxDelta {
        note: unified Delta
    }

    GrpcSubscriptionImpl o-- GrpcMux
    GrpcMux <|-- GrpcMuxImpl
    GrpcMux <|-- NewGrpcMuxImpl
    GrpcMux <|-- GrpcMuxSotw
    GrpcMux <|-- GrpcMuxDelta
```

## 4. gRPC stream and watch map

```mermaid
classDiagram
    class GrpcStreamInterface { <<interface>> }
    class GrpcStream~Req,Resp~ {
        -async_client_ : AsyncClient
        -backoff_strategy_ : BackOffStrategy
        -retry_timer_ : Timer
        -rate_limiter_ : TokenBucket
        +establishNewStream() void
        +checkRateLimitAllowsDrain() bool
        +onReceiveMessage(resp) void
        +onRemoteClose(...) void
    }
    class WatchMap {
        -watch_interest_ : map~name, set~Watch~~
        -wildcard_watches_ : set~Watch~
        +addWatch(cb, decoder) Watch
        +updateWatchInterest(watch, names) AddedRemoved
        +watchesInterestedIn(name) set~Watch~
        +onConfigUpdate(...) Status
    }
    class GrpcMuxFailover {
        note: primary + failover streams
    }

    GrpcStreamInterface <|-- GrpcStream
    GrpcMux ..> GrpcStream : owns
    GrpcMux ..> WatchMap : owns (delta + unified)
    GrpcStream ..> GrpcMuxFailover : optional wrap
```

## 5. REST classes

```mermaid
classDiagram
    class Subscription { <<interface>> }
    class RestApiFetcher {
        -refresh_timer_ : Timer
        -refresh_interval_ : ms
        +refresh() void
        +requestComplete() void
        #createRequest(req)* PURE
        #parseResponse(resp)* PURE
    }
    class HttpSubscriptionImpl {
        -request_ : DiscoveryRequest
        +start(names) void
        +parseResponse(resp) void
    }
    class HttpSubscriptionFactory {
        +name() "envoy.config_subscription.rest"
    }

    Subscription <|-- HttpSubscriptionImpl
    RestApiFetcher <|-- HttpSubscriptionImpl
    HttpSubscriptionFactory ..> HttpSubscriptionImpl : creates
```

## 6. Filesystem classes

```mermaid
classDiagram
    class Subscription { <<interface>> }
    class FilesystemSubscriptionImpl {
        -path_ : string
        -file_watcher_ : Filesystem::Watcher
        -directory_watcher_ : WatchedDirectory
        +start(names) void
        +refresh() void
        #refreshInternal(out) string
    }
    class FilesystemCollectionSubscriptionImpl {
        +refreshInternal(out) string
        note: reflects over Collection entries
    }
    class FilesystemSubscriptionFactory {
        +name() "envoy.config_subscription.filesystem"
    }

    Subscription <|-- FilesystemSubscriptionImpl
    FilesystemSubscriptionImpl <|-- FilesystemCollectionSubscriptionImpl
    FilesystemSubscriptionFactory ..> FilesystemSubscriptionImpl : creates
```

## 7. Factory registry summary

```mermaid
classDiagram
    class ConfigSubscriptionFactory { <<interface>> }
    class GrpcConfigSubscriptionFactory
    class DeltaGrpcConfigSubscriptionFactory
    class AdsConfigSubscriptionFactory
    class HttpSubscriptionFactory
    class FilesystemSubscriptionFactory
    class MuxFactory { <<interface>> }
    class GrpcMuxFactory

    ConfigSubscriptionFactory <|-- GrpcConfigSubscriptionFactory
    ConfigSubscriptionFactory <|-- DeltaGrpcConfigSubscriptionFactory
    ConfigSubscriptionFactory <|-- AdsConfigSubscriptionFactory
    ConfigSubscriptionFactory <|-- HttpSubscriptionFactory
    ConfigSubscriptionFactory <|-- FilesystemSubscriptionFactory
    MuxFactory <|-- GrpcMuxFactory
```

`ConfigSubscriptionFactory` instances are selected by `SubscriptionFactoryImpl` from the config
source type; `GrpcMuxFactory` (a separate `MuxFactory` registry) builds the cluster manager's
shared ADS mux.
