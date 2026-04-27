# Container Deployment

This document covers best practices for deploying Envoy in containerized environments, with a focus on Docker and Kubernetes.

## Table of Contents

1. [Docker Deployment](#docker-deployment)
2. [Kubernetes Deployment](#kubernetes-deployment)
3. [Sidecar Pattern](#sidecar-pattern)
4. [Init Containers](#init-containers)
5. [Istio Sidecar Injection](#istio-sidecar-injection)
6. [Lifecycle Management](#lifecycle-management)
7. [Resource Limits](#resource-limits)
8. [Health Checks](#health-checks)
9. [Best Practices](#best-practices)

---

## Docker Deployment

### Dockerfile Best Practices

```dockerfile
# Multi-stage build for minimal image size
FROM envoyproxy/envoy:v1.28-latest as envoy

FROM debian:bookworm-slim

# Install minimal runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy Envoy binary from official image
COPY --from=envoy /usr/local/bin/envoy /usr/local/bin/envoy

# Create non-root user
RUN groupadd -r envoy && \
    useradd -r -g envoy -s /sbin/nologin -c "Envoy user" envoy && \
    mkdir -p /etc/envoy /var/log/envoy && \
    chown -R envoy:envoy /etc/envoy /var/log/envoy

# Copy configuration
COPY --chown=envoy:envoy envoy.yaml /etc/envoy/envoy.yaml

# Expose ports
EXPOSE 10000 9901

# Switch to non-root user
USER envoy

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:9901/ready || exit 1

# Set entrypoint
ENTRYPOINT ["/usr/local/bin/envoy"]
CMD ["-c", "/etc/envoy/envoy.yaml", "--log-level", "info"]
```

### Optimized Production Dockerfile

```dockerfile
# production.Dockerfile
FROM envoyproxy/envoy:v1.28-latest as envoy

# Use distroless for minimal attack surface
FROM gcr.io/distroless/base-debian11

# Copy only what's needed
COPY --from=envoy /usr/local/bin/envoy /usr/local/bin/envoy
COPY --from=envoy /etc/ssl/certs /etc/ssl/certs

# Configuration (can be overridden with volume mount)
COPY envoy.yaml /etc/envoy/envoy.yaml

# Run as non-root
USER 1000:1000

EXPOSE 10000 9901

ENTRYPOINT ["/usr/local/bin/envoy"]
CMD ["-c", "/etc/envoy/envoy.yaml"]
```

### Docker Compose Example

```yaml
# docker-compose.yaml
version: '3.8'

services:
  envoy:
    image: mycompany/envoy:latest
    build:
      context: .
      dockerfile: Dockerfile
    
    ports:
      - "80:10000"
      - "9901:9901"
    
    volumes:
      # Configuration
      - ./envoy.yaml:/etc/envoy/envoy.yaml:ro
      # TLS certificates
      - ./certs:/etc/envoy/certs:ro
      # Logs (optional, prefer stdout)
      - ./logs:/var/log/envoy
    
    environment:
      - ENVOY_LOG_LEVEL=info
      - ENVOY_UID=0  # Run as root if binding to privileged ports
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9901/ready"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s
    
    # Restart policy
    restart: unless-stopped
    
    networks:
      - envoy-network
  
  # Backend service
  backend:
    image: mycompany/backend:latest
    networks:
      - envoy-network

networks:
  envoy-network:
    driver: bridge
```

### Running with Docker

```bash
# Build image
docker build -t mycompany/envoy:latest .

# Run with configuration override
docker run -d \
  --name envoy \
  -p 80:10000 \
  -p 9901:9901 \
  -v $(pwd)/envoy.yaml:/etc/envoy/envoy.yaml:ro \
  -v $(pwd)/certs:/etc/envoy/certs:ro \
  mycompany/envoy:latest

# View logs
docker logs -f envoy

# Check health
docker exec envoy curl -s http://localhost:9901/ready

# Perform hot restart
docker exec envoy /usr/local/bin/envoy \
  -c /etc/envoy/envoy-new.yaml \
  --restart-epoch 1 \
  --drain-time-s 600

# Stop gracefully
docker stop --time 30 envoy
```

---

## Kubernetes Deployment

### Standalone Deployment

```yaml
# envoy-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
  namespace: default
data:
  envoy.yaml: |
    admin:
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9901
    
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 10000
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              codec_type: AUTO
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: backend_cluster
              http_filters:
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      
      clusters:
      - name: backend_cluster
        type: STRICT_DNS
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: backend_cluster
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: backend.default.svc.cluster.local
                    port_value: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy
  namespace: default
  labels:
    app: envoy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: envoy
  
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  
  template:
    metadata:
      labels:
        app: envoy
        version: v1
      annotations:
        # Force pod restart on config change
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    
    spec:
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Init container (if needed for setup)
      initContainers:
      - name: init-config
        image: busybox:1.35
        command: ['sh', '-c', 'echo "Validating configuration..."; exit 0']
      
      containers:
      - name: envoy
        image: envoyproxy/envoy:v1.28-latest
        imagePullPolicy: IfNotPresent
        
        command:
        - /usr/local/bin/envoy
        args:
        - -c
        - /etc/envoy/envoy.yaml
        - --log-level
        - info
        - --component-log-level
        - upstream:debug,connection:trace
        
        ports:
        - name: http
          containerPort: 10000
          protocol: TCP
        - name: admin
          containerPort: 9901
          protocol: TCP
        
        # Environment variables
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        # Resource limits
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 2000m
            memory: 512Mi
        
        # Volume mounts
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
          readOnly: true
        - name: tls-certs
          mountPath: /etc/envoy/certs
          readOnly: true
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /server_info
            port: 9901
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /ready
            port: 9901
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Startup probe (for slow-starting containers)
        startupProbe:
          httpGet:
            path: /ready
            port: 9901
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 30  # 30 * 5 = 150s max startup time
        
        # Lifecycle hooks
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Graceful shutdown: drain connections before killing
                curl -X POST http://localhost:9901/drain_listeners
                sleep 15
      
      # Volumes
      volumes:
      - name: envoy-config
        configMap:
          name: envoy-config
      - name: tls-certs
        secret:
          secretName: envoy-tls-certs
      
      # DNS policy
      dnsPolicy: ClusterFirst
      
      # Termination grace period
      terminationGracePeriodSeconds: 30
      
      # Affinity (spread across nodes)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - envoy
              topologyKey: kubernetes.io/hostname
---
apiVersion: v1
kind: Service
metadata:
  name: envoy
  namespace: default
  labels:
    app: envoy
spec:
  type: LoadBalancer
  selector:
    app: envoy
  ports:
  - name: http
    port: 80
    targetPort: 10000
    protocol: TCP
  - name: admin
    port: 9901
    targetPort: 9901
    protocol: TCP
  
  # Session affinity (if needed)
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: envoy
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: envoy
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max
---
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: envoy
  namespace: default
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: envoy
```

---

## Sidecar Pattern

### Kubernetes Sidecar Deployment

```yaml
# app-with-envoy-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      # Shared volume for communication
      volumes:
      - name: shared-data
        emptyDir: {}
      - name: envoy-config
        configMap:
          name: envoy-sidecar-config
      
      containers:
      # Application container
      - name: app
        image: mycompany/my-app:latest
        ports:
        - containerPort: 8080
          name: http
        
        env:
        # App sends requests to localhost:15001 (Envoy)
        - name: HTTP_PROXY
          value: "http://127.0.0.1:15001"
        - name: HTTPS_PROXY
          value: "http://127.0.0.1:15001"
        
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
      
      # Envoy sidecar
      - name: envoy
        image: envoyproxy/envoy:v1.28-latest
        
        command:
        - /usr/local/bin/envoy
        args:
        - -c
        - /etc/envoy/envoy.yaml
        - --log-level
        - info
        - --service-cluster
        - my-app
        - --service-node
        - $(POD_NAME)
        
        ports:
        - name: http-envoy
          containerPort: 15001
          protocol: TCP
        - name: admin
          containerPort: 15000
          protocol: TCP
        
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
          readOnly: true
        - name: shared-data
          mountPath: /var/lib/envoy
        
        livenessProbe:
          httpGet:
            path: /server_info
            port: 15000
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 15000
          initialDelaySeconds: 5
          periodSeconds: 5
        
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                curl -X POST http://localhost:15000/drain_listeners
                sleep 10
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-sidecar-config
  namespace: default
data:
  envoy.yaml: |
    admin:
      address:
        socket_address:
          address: 127.0.0.1
          port_value: 15000
    
    static_resources:
      listeners:
      # Inbound listener (receives traffic from other services)
      - name: inbound_listener
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 15001
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: inbound
              codec_type: AUTO
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: local_app
              http_filters:
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      
      # Outbound listener (intercepts traffic from app to external services)
      - name: outbound_listener
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 15002
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: outbound
              codec_type: AUTO
              route_config:
                name: outbound_route
                virtual_hosts:
                - name: external
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: passthrough
              http_filters:
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      
      clusters:
      # Local application
      - name: local_app
        type: STATIC
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: local_app
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 127.0.0.1
                    port_value: 8080
      
      # Passthrough for outbound traffic
      - name: passthrough
        type: ORIGINAL_DST
        lb_policy: CLUSTER_PROVIDED
```

---

## Init Containers

Init containers prepare the environment before Envoy starts.

### Traffic Interception with iptables

```yaml
# envoy-with-init.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-with-proxy
spec:
  template:
    spec:
      # Init container to configure iptables
      initContainers:
      - name: istio-init
        image: istio/proxyv2:1.19.0
        imagePullPolicy: IfNotPresent
        
        # Must run as root to configure iptables
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
          runAsUser: 0
          runAsNonRoot: false
        
        command:
        - istio-iptables
        args:
        - "-p"
        - "15001"  # Envoy inbound port
        - "-z"
        - "15006"  # Envoy inbound capture port
        - "-u"
        - "1337"   # UID of proxy user
        - "-m"
        - "REDIRECT"  # Mode
        - "-i"
        - "*"      # Capture all inbound
        - "-x"
        - ""       # Exclude IPs
        - "-b"
        - "*"      # Capture all inbound ports
        - "-d"
        - "15090,15021,15020"  # Exclude outbound ports (health, metrics)
        
        env:
        - name: DNS_CAPTURE
          value: "false"
      
      containers:
      - name: app
        image: mycompany/my-app:latest
        # App configuration...
      
      - name: envoy
        image: envoyproxy/envoy:v1.28-latest
        securityContext:
          runAsUser: 1337
        # Envoy configuration...
```

### Configuration Validation Init Container

```yaml
initContainers:
- name: validate-config
  image: envoyproxy/envoy:v1.28-latest
  command:
  - /usr/local/bin/envoy
  args:
  - --mode
  - validate
  - -c
  - /etc/envoy/envoy.yaml
  volumeMounts:
  - name: envoy-config
    mountPath: /etc/envoy
    readOnly: true
```

### Certificate Fetching Init Container

```yaml
initContainers:
- name: fetch-certs
  image: mycompany/cert-fetcher:latest
  command:
  - /bin/sh
  - -c
  - |
    # Fetch certificates from vault or secret manager
    /usr/local/bin/fetch-certs \
      --output /etc/envoy/certs \
      --cert-name my-app-cert
  
  env:
  - name: VAULT_ADDR
    value: "https://vault.internal.example.com"
  - name: VAULT_TOKEN
    valueFrom:
      secretKeyRef:
        name: vault-token
        key: token
  
  volumeMounts:
  - name: tls-certs
    mountPath: /etc/envoy/certs
```

---

## Istio Sidecar Injection

### Automatic Injection

```yaml
# Enable automatic sidecar injection for namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
---
# Application deployment (sidecar auto-injected)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1
      annotations:
        # Control sidecar injection
        sidecar.istio.io/inject: "true"
        
        # Sidecar resource customization
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyCPULimit: "2000m"
        sidecar.istio.io/proxyMemory: "128Mi"
        sidecar.istio.io/proxyMemoryLimit: "512Mi"
        
        # Custom proxy config
        proxy.istio.io/config: |
          concurrency: 2
          tracing:
            zipkin:
              address: zipkin.istio-system:9411
        
        # Traffic interception modes
        sidecar.istio.io/interceptionMode: "REDIRECT"  # or TPROXY
        
        # Include/exclude ports
        traffic.sidecar.istio.io/includeInboundPorts: "8080,8443"
        traffic.sidecar.istio.io/excludeOutboundPorts: "3306,6379"
        
        # Status port for health checks
        status.sidecar.istio.io/port: "15021"
    
    spec:
      containers:
      - name: my-app
        image: mycompany/my-app:latest
        ports:
        - containerPort: 8080
          name: http
```

### Manual Injection

```bash
# Inject sidecar into existing deployment YAML
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# Inject with custom configuration
istioctl kube-inject \
  -f deployment.yaml \
  --injectConfigFile inject-config.yaml \
  --meshConfigFile mesh-config.yaml \
  | kubectl apply -f -
```

### Custom Sidecar Resource

```yaml
# Custom sidecar configuration
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: my-app-sidecar
  namespace: production
spec:
  workloadSelector:
    labels:
      app: my-app
  
  # Ingress configuration
  ingress:
  - port:
      number: 8080
      protocol: HTTP
      name: http
    defaultEndpoint: 127.0.0.1:8080
  
  # Egress configuration (limit what services can be reached)
  egress:
  - hosts:
    - "production/*"
    - "istio-system/*"
    - "*/api.external.com"
  
  # Outbound traffic policy
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY  # Only allow registered services
```

---

## Lifecycle Management

### Graceful Shutdown with preStop Hook

```yaml
containers:
- name: envoy
  image: envoyproxy/envoy:v1.28-latest
  
  lifecycle:
    preStop:
      exec:
        command:
        - /bin/sh
        - -c
        - |
          #!/bin/sh
          set -e
          
          # 1. Start draining listeners (stop accepting new connections)
          echo "Draining listeners..."
          curl -X POST http://localhost:9901/drain_listeners
          
          # 2. Wait for existing connections to complete
          echo "Waiting for connections to drain..."
          DRAIN_TIME=15
          
          for i in $(seq 1 $DRAIN_TIME); do
            ACTIVE_CONN=$(curl -s http://localhost:9901/stats/prometheus | \
              grep downstream_cx_active | \
              awk '{print $2}')
            
            if [ "$ACTIVE_CONN" = "0" ]; then
              echo "All connections drained"
              exit 0
            fi
            
            echo "Active connections: $ACTIVE_CONN (waiting...)"
            sleep 1
          done
          
          echo "Drain timeout reached, exiting"
          exit 0
  
  # Give enough time for graceful shutdown
  terminationGracePeriodSeconds: 30
```

### Rolling Update with Hot Restart

```yaml
# Deployment with hot restart support
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy
spec:
  replicas: 3
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # At least 2 replicas always running
      maxSurge: 1        # Maximum 4 replicas during update
  
  template:
    spec:
      containers:
      - name: envoy
        image: envoyproxy/envoy:v1.29-latest  # New version
        
        # Use shared volume for hot restart
        volumeMounts:
        - name: shared-memory
          mountPath: /dev/shm
        - name: socket-dir
          mountPath: /tmp
        
        command:
        - /usr/local/bin/envoy
        args:
        - -c
        - /etc/envoy/envoy.yaml
        - --base-id
        - "0"
        - --restart-epoch
        - "0"  # Will be managed by orchestrator
        
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Wait for Envoy to be ready
                while ! curl -s http://localhost:9901/ready; do
                  sleep 1
                done
                echo "Envoy is ready"
          
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                curl -X POST http://localhost:9901/drain_listeners
                sleep 15
      
      volumes:
      - name: shared-memory
        emptyDir:
          medium: Memory
          sizeLimit: 128Mi
      - name: socket-dir
        emptyDir: {}
```

---

## Resource Limits

### CPU and Memory Limits

```yaml
containers:
- name: envoy
  image: envoyproxy/envoy:v1.28-latest
  
  resources:
    requests:
      # Minimum guaranteed resources
      cpu: 200m       # 0.2 CPU cores
      memory: 256Mi   # 256 MiB RAM
    
    limits:
      # Maximum allowed resources
      cpu: 2000m      # 2 CPU cores
      memory: 512Mi   # 512 MiB RAM
```

### Resource Calculation Guidelines

**CPU:**
- Base: 100m per worker thread
- Add: 50m per 1000 req/s
- Edge proxy: 500m - 2000m
- Sidecar: 100m - 500m

**Memory:**
- Base: 128Mi minimum
- Add: 1Mi per active connection
- Add: Memory for buffers (connection_buffer_limit)
- Edge proxy: 512Mi - 2Gi
- Sidecar: 128Mi - 512Mi

### Quality of Service Classes

```yaml
# Guaranteed QoS (requests == limits)
resources:
  requests:
    cpu: 1000m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 512Mi

# Burstable QoS (requests < limits)
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 2000m
    memory: 1Gi

# BestEffort QoS (no requests/limits)
# NOT RECOMMENDED for production
```

---

## Health Checks

### Liveness Probe

Determines if container should be restarted.

```yaml
livenessProbe:
  httpGet:
    path: /server_info
    port: 9901
    scheme: HTTP
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3
```

### Readiness Probe

Determines if pod should receive traffic.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 9901
    scheme: HTTP
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3
```

### Startup Probe

For slow-starting containers (protects liveness probe).

```yaml
startupProbe:
  httpGet:
    path: /ready
    port: 9901
  initialDelaySeconds: 0
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 30  # 30 * 5s = 150s max startup time
```

### Custom Health Check Script

```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - |
      # Check if Envoy admin is responsive
      if ! curl -sf http://localhost:9901/server_info > /dev/null; then
        exit 1
      fi
      
      # Check if any clusters are healthy
      HEALTHY=$(curl -s http://localhost:9901/clusters | \
        grep "health_flags::healthy" | wc -l)
      
      if [ "$HEALTHY" -eq 0 ]; then
        echo "No healthy clusters"
        exit 1
      fi
      
      exit 0
  initialDelaySeconds: 30
  periodSeconds: 10
```

---

## Best Practices

### 1. Use Minimal Base Images

```dockerfile
# Good: Distroless base
FROM gcr.io/distroless/base-debian11

# Better: Distroless static (if no dynamic libs needed)
FROM gcr.io/distroless/static-debian11
```

### 2. Run as Non-Root

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  
  # Drop all capabilities
  capabilities:
    drop:
    - ALL
  
  # Read-only root filesystem
  readOnlyRootFilesystem: true
```

### 3. Set Appropriate Resource Limits

- Always set both requests and limits
- Use Guaranteed QoS for critical workloads
- Monitor actual usage and adjust

### 4. Configure Graceful Shutdown

- Use preStop hooks to drain connections
- Set terminationGracePeriodSeconds ≥ drain time + 10s
- Test shutdown behavior under load

### 5. Use Health Checks

- Configure all three probe types
- Set realistic timeouts and thresholds
- Use startup probe for slow-starting containers

### 6. Enable Logging

```yaml
# Log to stdout (captured by Kubernetes)
args:
- --log-level
- info
- --log-format
- '[%Y-%m-%d %T.%e][%t][%l][%n] %v'

# Don't write logs to files in containers
```

### 7. Use ConfigMaps for Configuration

```yaml
# Separate config from image
volumes:
- name: envoy-config
  configMap:
    name: envoy-config
```

### 8. Implement Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: envoy
spec:
  minAvailable: 2  # Always keep 2 pods running
  selector:
    matchLabels:
      app: envoy
```

### 9. Use Anti-Affinity Rules

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: envoy
        topologyKey: kubernetes.io/hostname
```

### 10. Monitor and Alert

```yaml
# Export metrics
containers:
- name: envoy
  ports:
  - name: metrics
    containerPort: 9901

---
# ServiceMonitor for Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: envoy
spec:
  selector:
    matchLabels:
      app: envoy
  endpoints:
  - port: admin
    path: /stats/prometheus
    interval: 30s
```

---

## Troubleshooting

### Pod Not Starting

```bash
# Check pod status
kubectl get pod -n production

# Check pod events
kubectl describe pod <pod-name> -n production

# Check logs
kubectl logs <pod-name> -c envoy -n production

# Check previous container logs (if crashed)
kubectl logs <pod-name> -c envoy -n production --previous
```

### Configuration Errors

```bash
# Validate configuration locally
docker run --rm -v $(pwd)/envoy.yaml:/etc/envoy/envoy.yaml \
  envoyproxy/envoy:v1.28-latest \
  envoy --mode validate -c /etc/envoy/envoy.yaml

# Exec into pod to debug
kubectl exec -it <pod-name> -c envoy -n production -- /bin/sh
```

### Health Check Failures

```bash
# Check health endpoints manually
kubectl exec <pod-name> -c envoy -- curl -v http://localhost:9901/ready
kubectl exec <pod-name> -c envoy -- curl -v http://localhost:9901/server_info
```

---

## Related Documentation

- [Deployment Patterns](01-deployment-patterns.md) - Common deployment scenarios
- [Hot Restart](02-hot-restart.md) - Zero-downtime updates
- [Production Hardening](04-production-hardening.md) - Security and reliability
