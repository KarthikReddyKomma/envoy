# Understanding Subscriptions in Envoy

## What is a Subscription?

A **subscription** is Envoy's mechanism for receiving **dynamic configuration updates** from a control plane server (like Istiod in Istio).

Think of it like:
- **Netflix subscription**: You subscribe, and new content appears automatically
- **Envoy subscription**: Envoy subscribes to config types, and Istiod pushes updates automatically

## Subscription with Istiod (Istio)

### Architecture

```mermaid
graph TB
    subgraph "Kubernetes Pod"
        App[Application Container<br/>:8080]
        Envoy[Envoy Sidecar<br/>:15001]
    end
    
    subgraph "Istio Control Plane"
        Istiod[Istiod<br/>xDS Server<br/>:15010]
    end
    
    Envoy -->|1. Subscribe<br/>gRPC Stream| Istiod
    Istiod -->|2. Config Updates<br/>DiscoveryResponse| Envoy
    
    Envoy -.->|Intercepts| App
    
    style Envoy fill:#e1f5ff
    style Istiod fill:#ffe1e1
    style App fill:#e1ffe1
```

### Complete Flow with Istiod

```mermaid
sequenceDiagram
    participant Pod as Pod Starts
    participant Envoy as Envoy Sidecar
    participant Sub as Subscription Manager
    participant gRPC as gRPC Stream
    participant Istiod as Istiod Control Plane
    participant K8s as Kubernetes API
    
    Pod->>Envoy: Start with bootstrap config
    Envoy->>Sub: Initialize XdsManager
    
    Note over Envoy,Istiod: Phase 1: Establish Connection
    
    Sub->>gRPC: Create gRPC connection to Istiod
    gRPC->>Istiod: Connect (mTLS)
    Istiod->>K8s: Get pod metadata
    
    Note over Envoy,Istiod: Phase 2: Subscribe to Resources
    
    Sub->>Istiod: DiscoveryRequest (LDS)<br/>Node: podname.namespace
    Istiod->>Istiod: Generate listener config<br/>based on VirtualServices
    Istiod->>Sub: DiscoveryResponse (Listeners)
    Sub->>Envoy: Apply listeners
    
    Sub->>Istiod: DiscoveryRequest (CDS)
    Istiod->>Istiod: Generate cluster config<br/>based on Services
    Istiod->>Sub: DiscoveryResponse (Clusters)
    Sub->>Envoy: Apply clusters
    
    Sub->>Istiod: DiscoveryRequest (EDS)
    Istiod->>K8s: Get pod endpoints
    Istiod->>Sub: DiscoveryResponse (Endpoints)
    Sub->>Envoy: Apply endpoints
    
    Note over Envoy,Istiod: Phase 3: Runtime Updates
    
    Note over K8s: Service scaled up
    K8s->>Istiod: Watch event (new pod)
    Istiod->>Sub: Push new EDS config
    Sub->>Envoy: Update endpoints
    
    Note over K8s: VirtualService changed
    K8s->>Istiod: Watch event (config change)
    Istiod->>Sub: Push new RDS config
    Sub->>Envoy: Update routes
```

## Subscription Types

### 1. ADS (Aggregated Discovery Service) - **Default in Istio**

**Single gRPC stream for all resource types**

```yaml
# Typical Istio sidecar config
dynamic_resources:
  ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
      - envoy_grpc:
          cluster_name: xds-grpc
    set_node_on_first_message_only: true
  
  # All resources use ADS
  lds_config: { ads: {} }
  cds_config: { ads: {} }
  rds_config: { ads: {} }
```

**Benefits:**
- Single connection (less overhead)
- Ordered updates (dependencies respected)
- Atomic updates across resource types

**Flow:**
```
┌─────────┐                    ┌─────────┐
│  Envoy  │◄──ADS gRPC Stream──┤ Istiod  │
│ Sidecar │    (multiplexed)   │ (xDS)   │
└─────────┘                    └─────────┘
     ▲                              │
     │     LDS, CDS, RDS, EDS      │
     └──────────────────────────────┘
```

### 2. Individual xDS

**Separate connection per resource type**

```yaml
dynamic_resources:
  lds_config:
    api_config_source:
      api_type: GRPC
      grpc_services:
        - envoy_grpc:
            cluster_name: lds-cluster
  
  cds_config:
    api_config_source:
      api_type: GRPC
      grpc_services:
        - envoy_grpc:
            cluster_name: cds-cluster
```

**Flow:**
```
┌─────────┐    LDS Stream    ┌─────────┐
│  Envoy  │◄─────────────────┤ Istiod  │
│         │    CDS Stream    │         │
│         │◄─────────────────┤         │
│         │    RDS Stream    │         │
│         │◄─────────────────┤         │
└─────────┘                  └─────────┘
```

### 3. Filesystem (Development/Testing)

**Watch local config files**

```yaml
dynamic_resources:
  lds_config:
    path_config_source:
      path: /etc/envoy/lds.yaml
      watched_directory:
        path: /etc/envoy
```

**Use Case:** Local development, static deployments

## What Gets Subscribed?

### Resource Types in Istio

```mermaid
graph TD
    Sub[Subscription Manager] --> LDS[LDS Subscription<br/>Listeners]
    Sub --> CDS[CDS Subscription<br/>Clusters]
    Sub --> RDS[RDS Subscription<br/>Routes]
    Sub --> EDS[EDS Subscription<br/>Endpoints]
    Sub --> SDS[SDS Subscription<br/>Secrets/Certs]
    
    LDS -->|Defines| L1[What ports to listen on<br/>0.0.0.0:15006<br/>virtualInbound]
    CDS -->|Defines| C1[Upstream services<br/>productpage.default<br/>reviews.default]
    RDS -->|Defines| R1[Routing rules<br/>VirtualService<br/>DestinationRule]
    EDS -->|Defines| E1[Pod IPs<br/>10.244.0.5:8080<br/>10.244.0.6:8080]
    SDS -->|Defines| S1[TLS certificates<br/>mTLS config]
    
    style Sub fill:#FFD700
    style LDS fill:#90EE90
    style CDS fill:#87CEEB
    style RDS fill:#DDA0DD
    style EDS fill:#F0E68C
    style SDS fill:#FFA07A
```

### Mapping Istio Resources to Subscriptions

| Istio Resource | Subscription Type | What Envoy Receives |
|----------------|-------------------|---------------------|
| **Service** | CDS, EDS | Cluster config + endpoint IPs |
| **VirtualService** | RDS, LDS | Routes and listeners |
| **DestinationRule** | CDS | Load balancing, circuit breakers |
| **Gateway** | LDS, RDS | Gateway listeners and routes |
| **ServiceEntry** | CDS, EDS | External service definitions |
| **PeerAuthentication** | LDS, CDS | mTLS settings |
| **AuthorizationPolicy** | LDS | RBAC filters |

## Subscription Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Init: Envoy starts
    Init --> Connecting: Create subscription
    Connecting --> Subscribed: gRPC stream established
    
    Subscribed --> WaitingForConfig: Send DiscoveryRequest
    WaitingForConfig --> Validating: Receive DiscoveryResponse
    
    Validating --> Applying: Config valid
    Validating --> WaitingForConfig: Config invalid (NACK)
    
    Applying --> Subscribed: ACK sent
    
    Subscribed --> WaitingForConfig: Config change detected
    
    Subscribed --> Reconnecting: Connection lost
    Reconnecting --> Connecting: Retry
    
    Subscribed --> [*]: Envoy shutdown
```

## Subscription in Code

### Creating a Subscription

```cpp
// From source/common/config/xds_manager_impl.cc
absl::StatusOr<SubscriptionPtr> XdsManagerImpl::subscribeToSingletonResource(
    absl::string_view resource_name,
    absl::string_view type_url,  // e.g., "Listener"
    Stats::Scope& scope,
    SubscriptionCallbacks& callbacks) {
  
  // Create subscription via factory
  return subscription_factory_->subscriptionFromConfigSource(
      ads_config_source_,  // Connection to Istiod
      type_url,            // What resource type
      scope,
      callbacks,           // Called when config arrives
      resource_decoder_,
      options);
}
```

### Receiving Updates

```cpp
// Subscription callback when Istiod pushes new config
void onConfigUpdate(
    const std::vector<DecodedResourceRef>& resources,
    const std::string& version_info) {
  
  // Process each resource from Istiod
  for (const auto& resource : resources) {
    // Validate
    if (!validateResource(resource)) {
      // Send NACK to Istiod
      sendNack(version_info);
      return;
    }
    
    // Apply configuration
    applyResource(resource);
  }
  
  // Send ACK to Istiod
  sendAck(version_info);
}
```

## Example: Service Deployment Flow

### Scenario: Deploy new microservice in Istio

```mermaid
sequenceDiagram
    participant K8s as Kubernetes
    participant Istiod as Istiod
    participant Sub as Envoy Subscription
    participant Envoy as Envoy Sidecar
    
    Note over K8s: Deploy new service<br/>"reviews-v2"
    
    K8s->>Istiod: Watch event: New Service
    Istiod->>Istiod: Generate new cluster config
    
    Note over Istiod: Push update to all sidecars
    
    Istiod->>Sub: DiscoveryResponse (CDS)<br/>New cluster: reviews-v2
    Sub->>Envoy: Add cluster
    
    K8s->>Istiod: Watch event: Pods ready
    Istiod->>Istiod: Generate endpoint config
    
    Istiod->>Sub: DiscoveryResponse (EDS)<br/>Endpoints: 10.244.0.10:8080
    Sub->>Envoy: Add endpoints
    
    Note over Envoy: Now ready to route<br/>to reviews-v2
```

## Debugging Subscriptions

### Check Subscription Status

```bash
# Check if subscriptions are connected
curl http://localhost:15000/stats | grep control_plane.connected_state

# Check subscription errors
curl http://localhost:15000/stats | grep update_failure

# Check last update time
curl http://localhost:15000/stats | grep update_success
```

### View Current Config from Istiod

```bash
# Dump all config received from Istiod
curl http://localhost:15000/config_dump

# Check specific resource type
curl http://localhost:15000/config_dump?resource=listeners
curl http://localhost:15000/config_dump?resource=clusters
```

### Enable Debug Logging

```bash
# Enable xDS subscription debugging
curl -X POST "http://localhost:15000/logging?level=debug&paths=config:trace"

# Check logs
kubectl logs -f pod-name -c istio-proxy | grep xds
```

## Common Subscription Issues

### 1. Initial Fetch Timeout

**Symptom**: Envoy fails to start
```
Initial fetch timeout
```

**Cause**: Can't reach Istiod or Istiod too slow

**Fix**:
```yaml
ads_config:
  initial_fetch_timeout: 30s  # Increase timeout
```

### 2. NACK (Negative Acknowledgment)

**Symptom**: Config not applying
```
update_rejected: 5
```

**Cause**: Invalid config from Istiod

**Debug**:
```bash
# Check what was rejected
curl localhost:15000/config_dump | jq '.configs[] | select(.["@type"] | contains("Listener")) | .error_state'
```

### 3. Subscription Disconnect

**Symptom**: No updates received
```
control_plane.connected_state: 0
```

**Cause**: Network issues, Istiod restart

**Check**:
```bash
# Verify connectivity to Istiod
kubectl exec -it pod-name -c istio-proxy -- curl -v istiod.istio-system:15010
```

## Summary

**Subscriptions in Envoy:**

1. **Definition**: Active connection to receive config updates from control plane
2. **With Istiod**: Yes! Subscriptions connect Envoy sidecars to Istiod
3. **Protocol**: gRPC streaming (bidirectional)
4. **Types**: ADS (all-in-one) or individual xDS
5. **Resources**: LDS, CDS, RDS, EDS, SDS
6. **Lifecycle**: Connect → Subscribe → Receive → Apply → Update

**Key Takeaway**: Without subscriptions, Envoy would only have static config. Subscriptions enable **dynamic service mesh** capabilities - services can scale, routes can change, and Envoy automatically adapts!
