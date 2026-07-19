# Factory Context Layer — Class Hierarchy

UML-style class diagrams for the factory-context interfaces and implementations.
Documentation aids, not exhaustive.

## 1. Interfaces

```mermaid
classDiagram
    class CommonFactoryContext {
        <<interface>>
        +serverFactoryContext() ServerFactoryContext&
        +scope() Scope&
        +messageValidationVisitor() ValidationVisitor&
        +api() Api&
        +runtime() Loader&
    }
    class ServerFactoryContext {
        <<interface>>
        +clusterManager() ClusterManager&
        +mainThreadDispatcher() Dispatcher&
        +threadLocal() ThreadLocal::Instance&
        +singletonManager() Singleton::Manager&
        +options() Options&
        +regexEngine() Regex::Engine&
    }
    class GenericFactoryContext {
        <<interface>>
        +initManager() Init::Manager&
    }
    class FactoryContext {
        <<interface>>
        +listenerInfo() ListenerInfo&
        +listenerScope() Scope&
        +drainDecision() DrainDecision&
    }
    class TransportSocketFactoryContext { <<interface>> }
    class ResourceMonitorFactoryContext {
        <<interface>>
        +mainThreadDispatcher() Dispatcher&
        +options() Options&
        +api() Api&
        +runtime() Loader&
    }

    CommonFactoryContext <|-- ServerFactoryContext
    ServerFactoryContext <|-- GenericFactoryContext
    ServerFactoryContext <|-- FactoryContext
```

## 2. The root adapter and generic wrapper

```mermaid
classDiagram
    class ServerFactoryContext { <<interface>> }
    class TransportSocketFactoryContext { <<interface>> }
    class ServerFactoryContextImpl {
        -server_ : Instance&
        +clusterManager() forwards
        +mainThreadDispatcher() forwards
        +options() forwards
        +regexEngine() forwards
        note: lives in server.h
    }

    class GenericFactoryContext { <<interface>> }
    class GenericFactoryContextImpl {
        -server_context_ : ServerFactoryContext&
        -stats_scope_ : Scope&
        -validation_visitor_ : ValidationVisitor&
        -init_manager_ : Init::Manager*
        +setInitManager(mgr) void
    }

    ServerFactoryContext <|-- ServerFactoryContextImpl
    TransportSocketFactoryContext <|-- ServerFactoryContextImpl
    GenericFactoryContext <|-- GenericFactoryContextImpl
    GenericFactoryContextImpl o-- ServerFactoryContext : wraps
```

## 3. Transport socket context (alias)

```mermaid
classDiagram
    class GenericFactoryContextImpl
    class TransportSocketFactoryContextImpl {
        note: using = GenericFactoryContextImpl
    }
    GenericFactoryContextImpl <.. TransportSocketFactoryContextImpl : type alias
```

## 4. Listener-scoped context

```mermaid
classDiagram
    class FactoryContext { <<interface>> }
    class FactoryContextImplBase {
        #server_ : Instance&
        #validation_visitor_ : ValidationVisitor&
        #scope_ : ScopeSharedPtr
        #listener_scope_ : ScopeSharedPtr
        #listener_info_ : ListenerInfoConstSharedPtr
        +serverFactoryContext() override
        +scope() override
        +listenerScope() override
    }
    class FactoryContextImpl {
        -drain_decision_ : DrainDecision&
        +initManager() override
        +drainDecision() override
    }

    FactoryContext <|-- FactoryContextImplBase
    FactoryContextImplBase <|-- FactoryContextImpl
```

## 5. Resource monitor context

```mermaid
classDiagram
    class ResourceMonitorFactoryContext { <<interface>> }
    class ResourceMonitorFactoryContextImpl {
        -dispatcher_ : Dispatcher&
        -options_ : Options&
        -api_ : Api&
        -validation_visitor_ : ValidationVisitor&
        -runtime_ : Loader&
        note: header-only
    }
    ResourceMonitorFactoryContext <|-- ResourceMonitorFactoryContextImpl
```

## 6. How they relate at runtime

```mermaid
classDiagram
    class Instance { <<interface>> }
    class ServerFactoryContextImpl
    class GenericFactoryContextImpl
    class FactoryContextImpl
    class ResourceMonitorFactoryContextImpl

    Instance <.. ServerFactoryContextImpl : forwards to
    ServerFactoryContextImpl <.. GenericFactoryContextImpl : wraps
    ServerFactoryContextImpl <.. FactoryContextImpl : via serverFactoryContext()
    Instance <.. FactoryContextImpl : holds directly
```

Every implementation ultimately routes back to the single `ServerFactoryContextImpl` (and thus
the real `Server::Instance`), but each exposes only the slice of services its factory category
needs.
