# Envoy Active Stream Listener Architecture - Part 2: Operations and Management

> **Note**: This document is Part 2 of 2. See [Part 1: Core Architecture and Lifecycle](ACTIVE_STREAM_LISTENER_ARCHITECTURE_PART1.md) for class hierarchy, connection lifecycle, state machines, and filter processing.

## Overview

This document covers the operational aspects of Active Stream Listeners including socket management, statistics collection, memory management, integration points, error handling, configuration options, and performance considerations.

---

## Table of Contents

**Part 1: Core Architecture and Lifecycle** (see [ACTIVE_STREAM_LISTENER_ARCHITECTURE_PART1.md](ACTIVE_STREAM_LISTENER_ARCHITECTURE_PART1.md))
1. Class Hierarchy
2. Connection Lifecycle
3. ActiveTcpSocket State Machine
4. Listener Filter Processing
5. Connection Management by Filter Chain
6. Filter Chain Draining
7. Connection Lifecycle Events
8. Timeout Handling

**Part 2 (this document): Operations and Management**
9. [Socket Management](#9-socket-management)
10. [Statistics and Logging](#10-statistics-and-logging)
11. [Memory Management](#11-memory-management)
12. [Integration Points](#12-integration-points)
13. [Error Handling](#13-error-handling)
14. [Configuration](#14-configuration)
15. [Performance Considerations](#15-performance-considerations)
16. [Key Design Patterns](#16-key-design-patterns)

---

## 9. Socket Management

### Sockets List Operations

```mermaid
graph TB
    subgraph "ActiveStreamListenerBase"
        SocketsList[sockets_<br/>list~unique_ptr~ActiveTcpSocket~~]
    end

    subgraph "Operations"
        Add[Add Socket]
        Remove[Remove Socket]
        Iterate[Iterate Sockets]
        Clear[Clear on Destroy]
    end

    Add -->|LinkedList::moveIntoListBack| SocketsList
    SocketsList -->|removeSocket| Remove
    SocketsList --> Iterate
    SocketsList --> Clear

    style Add fill:#ccffcc
    style Remove fill:#ffcccc
    style SocketsList fill:#e1f5ff
```

### Socket Addition and Removal

```mermaid
sequenceDiagram
    participant Filter as Listener Filter
    participant Socket as ActiveTcpSocket
    participant List as sockets_ list
    participant Listener as ActiveStreamListenerBase

    Note over Filter,List: Filter Returns StopIteration

    Filter->>Socket: Return StopIteration
    Socket->>Socket: Check isEndFilterIteration()
    Socket-->>Socket: false (not at end)

    Socket->>Socket: startTimer()
    Socket->>List: LinkedList::moveIntoListBack
    Note over List: Socket now owned by list

    Note over Socket,List: Later: Filter Chain Completes

    Socket->>Socket: continueFilterChain completes
    Socket->>Listener: newConnection()
    Listener->>List: removeSocket(socket)
    List->>List: Remove from list
    List-->>Listener: unique_ptr~ActiveTcpSocket~

    Note over Listener: Socket ownership transferred
    Listener->>Listener: Create ActiveTcpConnection
```

---

## 10. Statistics and Logging

### Listener Statistics

```mermaid
graph TB
    subgraph "ListenerStats (Per-Listener)"
        CxTotal[downstream_cx_total]
        CxActive[downstream_cx_active]
        CxDestroy[downstream_cx_destroy]
        CxLength[downstream_cx_length_ms]
        NoFilter[no_filter_chain_match]
    end

    subgraph "PerHandlerListenerStats (Per-Worker)"
        WorkerCxActive[downstream_cx_active]
        WorkerCxTotal[downstream_cx_total]
    end

    subgraph "Events"
        NewConn[New Connection] --> CxTotal
        NewConn --> CxActive
        NewConn --> WorkerCxTotal
        NewConn --> WorkerCxActive

        ConnClose[Connection Close] --> CxDestroy
        ConnClose --> CxLength
        ConnClose -.->|dec| CxActive
        ConnClose -.->|dec| WorkerCxActive

        NoMatch[No Filter Chain] --> NoFilter
    end

    style CxTotal fill:#ccffcc
    style CxActive fill:#ffffcc
    style CxDestroy fill:#ffcccc
```

### Log Emission

```mermaid
sequenceDiagram
    participant Event
    participant Listener as ActiveStreamListenerBase
    participant StreamInfo
    participant Logger as Access Logger

    Note over Event: Connection Event Occurs

    alt Socket rejected before connection
        Event->>Listener: emitLogs(config, stream_info)
        Note over Listener: Socket failed filters
    else Connection closed normally
        Event->>Listener: emitLogs(config, stream_info)
        Note over Listener: Connection terminated
    end

    Listener->>StreamInfo: Populate final fields
    StreamInfo->>StreamInfo: Set response code
    StreamInfo->>StreamInfo: Set response flags
    StreamInfo->>StreamInfo: Set bytes sent/received

    Listener->>Logger: Log access log entry
    Logger->>Logger: Format and write

    Note over Logger: Includes:<br/>- Connection duration<br/>- Bytes transferred<br/>- Filter state<br/>- Dynamic metadata
```

---

## 11. Memory Management

### Ownership Model

```mermaid
graph TB
    subgraph "ActiveStreamListenerBase"
        SocketsList[sockets_<br/>Owned by list]
    end

    subgraph "OwnedActiveStreamListenerBase"
        ConnsByContext[connections_by_context_<br/>Owned by map]
    end

    subgraph "ActiveConnections"
        ConnsList[connections_<br/>Owned by list]
    end

    ActiveSocket1[ActiveTcpSocket 1]
    ActiveSocket2[ActiveTcpSocket 2]

    ActiveConn1[ActiveTcpConnection 1]
    ActiveConn2[ActiveTcpConnection 2]
    ActiveConn3[ActiveTcpConnection 3]

    SocketsList -->|unique_ptr| ActiveSocket1
    SocketsList -->|unique_ptr| ActiveSocket2

    ConnsByContext -->|unique_ptr| ConnsList

    ConnsList -->|unique_ptr| ActiveConn1
    ConnsList -->|unique_ptr| ActiveConn2
    ConnsList -->|unique_ptr| ActiveConn3

    style SocketsList fill:#e1f5ff
    style ConnsByContext fill:#ffe1e1
    style ConnsList fill:#ccffcc
```

### Deferred Deletion

```mermaid
sequenceDiagram
    participant Object
    participant Dispatcher
    participant DeferredList as Deferred Delete List

    Note over Object: Object ready for deletion

    Object->>Dispatcher: Add to deferred delete list
    Dispatcher->>DeferredList: Push back

    Note over DeferredList: Object still alive<br/>References safe this iteration

    Note over Dispatcher: Current event loop iteration ends

    Dispatcher->>DeferredList: clearDeferredDeleteList()

    loop For each object
        DeferredList->>Object: unique_ptr destructor
        Note over Object: Object destroyed
    end

    Note over DeferredList: List cleared
```

**Objects using deferred deletion:**
- `ActiveTcpSocket`
- `ActiveTcpConnection`
- `ActiveConnections`

---

## 12. Integration Points

### Connection Handler Integration

```mermaid
graph TB
    subgraph "ConnectionHandler"
        Handler[ConnectionHandler]
        ActiveListener[ActiveListener Interface]
    end

    subgraph "Listener Implementation"
        StreamListener[ActiveStreamListenerBase]
        OwnedListener[OwnedActiveStreamListenerBase]
    end

    subgraph "Concrete Implementations"
        TcpListener[ActiveTcpListener]
        UdpListener[ActiveRawUdpListener]
    end

    Handler --> ActiveListener
    ActiveListener <|.. StreamListener
    StreamListener <|-- OwnedListener
    OwnedListener <|-- TcpListener
    OwnedListener <|-- UdpListener

    style Handler fill:#e1f5ff
    style StreamListener fill:#ffe1e1
    style TcpListener fill:#ccffcc
```

### Network Filter Chain Integration

```mermaid
sequenceDiagram
    participant Socket as ActiveTcpSocket
    participant Listener as ActiveStreamListenerBase
    participant FilterChain as Network::FilterChain
    participant ServerConn as ServerConnection
    participant NetworkFilters as Network Filters

    Socket->>Listener: newConnection()
    Listener->>Listener: Find appropriate FilterChain
    Listener->>ServerConn: Create ServerConnection
    Listener->>Listener: newActiveConnection(filter_chain, conn, info)

    Note over Listener: Abstract method implemented by derived class

    Listener->>FilterChain: buildFilterChain(connection)
    FilterChain->>NetworkFilters: Create filter instances
    NetworkFilters->>ServerConn: Add to read/write filters

    ServerConn->>ServerConn: Start processing
    Note over ServerConn: Connection now active
```

---

## 13. Error Handling

### Error Scenarios

```mermaid
flowchart TD
    Start[Error Occurs] --> CheckType{Error Type}

    CheckType -->|Filter Chain Creation Failed| NoECDS[ECDS config missing]
    CheckType -->|Filter Timeout| Timeout[Listener filter timeout]
    CheckType -->|Filter Rejection| Reject[Filter returns Close]
    CheckType -->|Connection Error| ConnErr[Connection event error]

    NoECDS --> CloseSocket[Close socket immediately]
    Timeout --> CheckConfig{continue_on_timeout?}
    Reject --> CloseSocket
    ConnErr --> CloseConn[Close connection]

    CheckConfig -->|false| CloseSocket
    CheckConfig -->|true| ContinueChain[Continue filter chain]

    CloseSocket --> EmitLog1[emitLogs]
    CloseConn --> EmitLog2[emitLogs]
    ContinueChain --> Process[Continue processing]

    EmitLog1 --> UpdateStats1[Update error stats]
    EmitLog2 --> UpdateStats2[Update error stats]

    UpdateStats1 --> Cleanup[Deferred deletion]
    UpdateStats2 --> Cleanup
    Process --> End[Continue]

    style CloseSocket fill:#ffcccc
    style CloseConn fill:#ff8888
    style Reject fill:#ff4444
```

### Error Statistics

| Error Type | Stat Name | Trigger |
|-----------|-----------|---------|
| No filter chain match | `no_filter_chain_match` | Cannot find matching filter chain |
| Connection create failed | `downstream_cx_destroy` | Connection failed during creation |
| Filter timeout | Custom filter stat | Listener filter timeout exceeded |
| Filter rejection | Custom filter stat | Filter explicitly rejects connection |

---

## 14. Configuration

### Key Configuration Parameters

```yaml
# Listener configuration
listener:
  # Listener filter timeout
  listener_filters_timeout: 15s

  # Continue processing after timeout
  continue_on_listener_filters_timeout: false

  # Listener filters
  listener_filters:
    - name: envoy.filters.listener.tls_inspector
    - name: envoy.filters.listener.http_inspector
    - name: envoy.filters.listener.original_dst

  # Filter chains
  filter_chains:
    - filter_chain_match:
        server_names: ["example.com"]
      filters:
        - name: envoy.filters.network.http_connection_manager
```

### Configuration Impact

```mermaid
graph TB
    Config[Listener Config] --> Timeout[listener_filters_timeout]
    Config --> Continue[continue_on_timeout]
    Config --> ListenerFilters[listener_filters]
    Config --> FilterChains[filter_chains]

    Timeout --> SocketTimer[ActiveTcpSocket timer]
    Continue --> TimeoutBehavior[Timeout handling]
    ListenerFilters --> FilterChain[Filter chain creation]
    FilterChains --> ConnGroups[Connection grouping]

    style Config fill:#e1f5ff
    style Timeout fill:#ffffcc
    style Continue fill:#ffffcc
```

---

## 15. Performance Considerations

### Optimization Strategies

1. **Lazy Filter Chain Creation**
   - Filter chains created only when needed
   - Reduces memory overhead for idle listeners

2. **Connection Grouping**
   - Connections grouped by filter chain
   - Efficient draining of specific filter chains
   - O(1) lookup by filter chain pointer

3. **Deferred Deletion**
   - Objects deleted at safe points
   - Prevents use-after-free
   - Minimal overhead (end of event loop)

4. **Linked Lists for Ordering**
   - O(1) insertion/removal
   - Maintains connection order
   - Cache-friendly iteration

### Performance Metrics

```mermaid
graph LR
    subgraph "Timing"
        Accept[Socket Accept: ~10μs]
        Filter[Filter Processing: ~100μs]
        Connect[Connection Create: ~50μs]
    end

    subgraph "Memory"
        SocketMem[ActiveTcpSocket: ~512 bytes]
        ConnMem[ActiveTcpConnection: ~1KB]
    end

    subgraph "Scalability"
        Conns[Connections: 100K+]
        Listeners[Listeners: 1000+]
    end

    style Accept fill:#ccffcc
    style Filter fill:#ffffcc
    style Connect fill:#ccffcc
```

---

## 16. Key Design Patterns

### Pattern 1: Template Method Pattern
- `ActiveStreamListenerBase` provides template
- Derived classes implement `newActiveConnection()`
- Separation of socket handling from connection management

### Pattern 2: Chain of Responsibility
- Listener filters form a chain
- Each filter can:
  - Pass to next filter
  - Stop and wait
  - Reject connection
- Flexible, extensible filtering

### Pattern 3: Composite Pattern
- `ActiveConnections` groups connections
- Organized by filter chain
- Enables batch operations (drain)

### Pattern 4: Deferred Deletion Pattern
- Objects marked for deletion
- Actual deletion deferred to safe point
- Prevents dangling references

---

## Summary

The Active Stream Listener architecture provides:

1. **Robust Socket Processing**
   - Flexible listener filter chain
   - Timeout handling
   - Graceful failure modes

2. **Efficient Connection Management**
   - Grouped by filter chain
   - O(1) operations for common paths
   - Memory-efficient storage

3. **Lifecycle Management**
   - Clear ownership model
   - Deferred deletion for safety
   - Comprehensive logging

4. **Extensibility**
   - Abstract base for customization
   - Template method pattern
   - Filter chain extensibility

5. **Production Ready**
   - Error handling at all layers
   - Detailed statistics
   - Graceful degradation

This design enables Envoy to efficiently handle millions of connections while maintaining safety, observability, and flexibility for diverse deployment scenarios.

---

## See Also

- [Part 1: Core Architecture and Lifecycle](ACTIVE_STREAM_LISTENER_ARCHITECTURE_PART1.md) - Class hierarchy, connection lifecycle, state machines
- [ACTIVE_TCP_LISTENER_INVOCATION.md](ACTIVE_TCP_LISTENER_INVOCATION.md) - How ActiveTcpListener is invoked
- [LISTENER_MANAGER_ARCHITECTURE.md](LISTENER_MANAGER_ARCHITECTURE.md) - Overall listener manager design
- [BASE_CLASSES_AND_INTERFACES.md](BASE_CLASSES_AND_INTERFACES.md) - Base class reference
