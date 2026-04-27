# Index of Uncommitted Documentation

This index catalogs **all Markdown documents that are currently uncommitted** (created or modified in recent work). Use it to find which document covers a given topic and what content each document or series contains.

**Note:** Paths are relative to the repository root. Documents under `docs/` are linked from this file; documents under `source/` and `tools/` are listed with full path for reference.

---

## 1. Topic → Document Quick Reference

Find a topic below to see which document(s) cover it.

| Topic | Document(s) |
|-------|-------------|
| **API (Envoy proto)** | [api-envoy/01](api-envoy/01-API-Overview-and-Structure.md) – [05](api-envoy/05-Common-Types-and-Data.md) |
| **Bootstrap, runtime, xDS, Istio** | [core-runtime-xds-istio/01](core-runtime-xds-istio/01-Core-Runtime-and-Bootstrapping.md) – [04](core-runtime-xds-istio/04-EnvoyFilter-Sidecar-vs-Gateway.md) |
| **Filter chain (creation, factory, matching)** | [filter-chain/01](filter-chain/01-Overview-and-Architecture.md) – [10](filter-chain/10-Sequence-Diagrams-and-Flow-Charts.md), [filter-chain/README](filter-chain/README.md) |
| **Request flow (listener → HCM → router → pools)** | [request-flow/01](request-flow/01-overview.md) – [12](request-flow/12-response-flow.md) |
| **Flows (connection, HTTP, LB, xDS, retry)** | [flows/01](flows/01_connection_lifecycle.md) – [05](flows/05_retry_circuit_breaking.md), [flows/README](flows/README.md) |
| **HTTP filters (router, ext_authz, JWT, etc.)** | [http_filters/01](http_filters/01_router.md) – [10](http_filters/10_health_check.md), [http_filters/README](http_filters/README.md) |
| **Istio–Envoy config (xDS, Istiod, EnvoyFilter)** | [istio-envoy-config/01](istio-envoy-config/01-Architecture-Overview.md) – [06](istio-envoy-config/06-Complete-End-to-End-Example.md), [istio-envoy-config/README](istio-envoy-config/README.md) |
| **Access logs (architecture, formatters, gRPC, I/O)** | [access-logs/01](access-logs/01-architecture-lifecycle.md) – [03](access-logs/03-file-io-flushing.md) |
| **Security (TLS, SDS, ALPN/SNI, RBAC)** | [security/01](security/01_tls_context_implementation.md) – [04](security/04_rbac_internals.md), [security/README](security/README.md) |
| **SSL/TLS (contexts, handshake, SDS, OCSP)** | [ssl-tls/01](ssl-tls/01-architecture-contexts.md) – [03](ssl-tls/03-sds-rotation-ocsp.md) |
| **Tunneling (HTTP CONNECT, TCP over HTTP, internal)** | [tunneling/01](tunneling/01-overview-http-connect.md) – [03](tunneling/03-internal-listeners.md) |
| **Source-common (HTTP, codecs, network, router, upstream)** | [source-common/01](source-common/01-overview.md) – [10](source-common/10-supporting-subsystems.md) |
| **Listener filters (TLS inspector, proxy protocol, etc.)** | [source/extensions/filters/listener/docs/01](source/extensions/filters/listener/docs/01-Overview-and-Architecture.md) – [04](source/extensions/filters/listener/docs/04-Configuration-and-Extension.md) |
| **HTTP layer (connection manager, codec, filter manager)** | [source/common/http/*.md](source/common/http/) (filter_manager, codec_client, conn_manager_impl, etc.) |
| **Listener manager (active TCP, filter chain, LDS)** | [source/common/listener_manager/*.md](source/common/listener_manager/) (active_tcp_listener_and_socket, filter_chain_manager_impl, listener_impl, etc.) |
| **Network (connections, filters, sockets, listeners)** | [source/common/network/*.md](source/common/network/) (OVERVIEW parts, connection_impl, filter_manager_impl, etc.) |
| **Upstream (cluster manager, LB, health, outlier, CDS)** | [source/common/upstream/*.md](source/common/upstream/) (OVERVIEW parts, cluster_manager_impl, load_balancing, etc.) |
| **Observability (access logs, admin, stats, tracing)** | [source/docs/observability/*.md](source/docs/observability/) |
| **Dynamic modules (ABI, HTTP filter, SDKs)** | [source/extensions/dynamic_modules/*.md](source/extensions/dynamic_modules/) |
| **Tools (code quality, API, build, debugging, etc.)** | [tools/docs/01](tools/docs/01-Overview.md) – [09](tools/docs/09-Adding-New-Tools.md) |
| **Admin Interface & Operations** | [admin-operations/01](admin-operations/01-admin-interface.md) – [03](admin-operations/03-operational-procedures.md), [admin-operations/README](admin-operations/README.md) |
| **Extension Development (filters, load balancers, etc.)** | [extension-development/01](extension-development/01-overview.md) – [05](extension-development/05-extension-registration.md), [extension-development/README](extension-development/README.md) |
| **Testing (unit, integration, benchmarks)** | [testing/01](testing/01-testing-overview.md) – [05](testing/05-benchmarking.md), [testing/README](testing/README.md) |
| **Memory & Performance (threading, buffers, optimization)** | [memory-performance/01](memory-performance/01-threading-model.md) – [04](memory-performance/04-performance-optimization.md), [memory-performance/README](memory-performance/README.md) |
| **Debugging & Error Handling** | [debugging/01](debugging/01-error-handling-patterns.md) – [04](debugging/04-common-issues.md), [debugging/README](debugging/README.md) |
| **Deployment & Operations (patterns, hot restart, K8s)** | [deployment/01](deployment/01-deployment-patterns.md) – [04](deployment/04-production-hardening.md), [deployment/README](deployment/README.md) |
| **QUIC & UDP (HTTP/3, UDP proxy, address caching)** | [quic-udp/01](quic-udp/01-quic-overview.md) – [04](quic-udp/04-configuration-examples.md), [quic-udp/README](quic-udp/README.md) |

---

## 2. Content Covered in Each Document / Series

For each **document or document series**, a short summary of what it contains.

---

### docs/api-envoy/ (Envoy API — proto definitions)

| Document | Content covered |
|----------|------------------|
| **01-API-Overview-and-Structure** | Directory layout of `api/envoy/`, versioning (v2/v3), proto counts, how config, service, extensions, and type fit together. |
| **02-Core-Configuration-APIs** | Bootstrap, Cluster (CDS), Listener (LDS), Route (RDS), Endpoint (EDS), Core types (ConfigSource, Address, HealthCheck); configuration flow diagram. |
| **03-xDS-Discovery-Services** | SotW vs Delta, DiscoveryRequest/Response, LDS/CDS/RDS/EDS/SDS/RTDS/ADS, auxiliary services (ExtAuth, ExtProc, RLS, ALS, HDS, CSDS), streaming modes. |
| **04-Extensions-API** | Extension registration, HTTP/network/listener/UDP filters, transport sockets, access loggers, LB policies, clusters, health checkers, tracers, stat sinks, resource monitors. |
| **05-Common-Types-and-Data** | Shared types (matchers, percent, metadata), data events (access log, outlier, health, tap), admin types, watchdog, annotations, supporting config areas. |

---

### docs/core-runtime-xds-istio/

| Document | Content covered |
|----------|------------------|
| **01-Core-Runtime-and-Bootstrapping** | Core runtime, bootstrapping, and how Envoy starts. |
| **02-xDS-and-Dynamic-Configuration** | xDS protocol and dynamic configuration flow. |
| **03-Istio-to-Envoy-Mapping** | Mapping from Istio resources to Envoy config. |
| **04-EnvoyFilter-Sidecar-vs-Gateway** | EnvoyFilter usage in sidecar vs gateway. |

---

### docs/filter-chain/ (Filter chain creation and runtime)

| Document | Content covered |
|----------|------------------|
| **01-Overview-and-Architecture** | Filter chain intro, architecture layers, filter types (network, HTTP, listener), key components. |
| **02-Filter-Chain-Factory-Construction** | Double factory pattern, registration, config → factory mapping, network/HTTP/listener factories. |
| **03-Factory-Context-Hierarchy** | Context inheritance, resource scoping (server, listener, filter chain), init manager, draining. |
| **04-Network-Filter-Chain-Creation** | Network filter chain instantiation, FilterChainManager, filter chain selection, connection flow. |
| **05-HTTP-Filter-Chain-Creation** | HTTP filter chain creation and HCM integration. |
| **06-Filter-Chain-Matching-and-Selection** | How a filter chain is chosen (SNI, ALPN, etc.). |
| **07-Filter-Configuration-and-Registration** | Filter config and registration mechanics. |
| **08-Runtime-Filter-Instantiation** | Runtime filter instantiation and lifecycle. |
| **09-UML-Class-Diagrams** | UML class diagrams for filter chain components. |
| **10-Sequence-Diagrams-and-Flow-Charts** | Sequence diagrams and flowcharts for filter chain flows. |
| **README** | Series overview and per-part topic summary. |

---

### docs/request-flow/ (Request receive path)

| Document | Content covered |
|----------|------------------|
| **01-overview** | End-to-end request path: listener → network → HTTP (HCM, codec, filters, router, pools). |
| **02-listener-socket-accept** | Listen socket and accept path. |
| **03-listener-filters** | Listener filters (e.g. TLS inspector, proxy protocol) before filter chain selection. |
| **04-filter-chain-matching** | Filter chain matching and selection. |
| **05-network-filters** | Network filters and HCM as terminal filter. |
| **06-transport-sockets** | Transport sockets (e.g. TLS) on listener and upstream. |
| **07-http-connection-manager** | HCM role and configuration. |
| **08-http-codec** | HTTP codec (HTTP/1, HTTP/2, HTTP/3). |
| **09-http-filter-manager** | HTTP filter manager and filter iteration. |
| **10-router-filter** | Router filter and upstream routing. |
| **11-connection-pools** | Connection pools and upstream connections. |
| **12-response-flow** | Response path back to client. |

---

### docs/flows/ (UML and flow diagrams)

| Document | Content covered |
|----------|------------------|
| **01_connection_lifecycle** | TCP accept to connection close. |
| **02_http_request_flow** | HTTP request processing flow. |
| **03_cluster_load_balancing** | Cluster and load balancing flow. |
| **04_xds_configuration_flow** | xDS configuration update flow. |
| **05_retry_circuit_breaking** | Retry and circuit breaking behavior. |
| **README** | Overview and index of flow documents. |

---

### docs/http_filters/

| Document | Content covered |
|----------|------------------|
| **01_router** | Router HTTP filter. |
| **02_ext_authz** | External authorization filter. |
| **03_jwt_authn** | JWT authentication filter. |
| **04_ratelimit** | Rate limit filter. |
| **05_rbac** | RBAC filter. |
| **06_cors** | CORS filter. |
| **07_fault** | Fault injection filter. |
| **08_buffer** | Buffer filter. |
| **09_local_ratelimit** | Local rate limit filter. |
| **10_health_check** | Health check filter. |
| **README** | Index of HTTP filter docs. |

---

### docs/istio-envoy-config/

| Document | Content covered |
|----------|------------------|
| **01-Architecture-Overview** | Istio architecture, Envoy as data plane, xDS basics, control→data plane flow. |
| **02-Istio-Configuration-Resources** | VirtualService, DestinationRule, Gateway, ServiceEntry and Envoy mapping. |
| **03-xDS-Protocol-Deep-Dive** | xDS protocol details, ADS, incremental xDS, ACK/NACK. |
| **04-Istiod-Configuration-Processing** | How Istiod produces and pushes config. |
| **05-Envoy-Configuration-Application** | How Envoy applies received config. |
| **06-Complete-End-to-End-Example** | Full example from Istio config to Envoy behavior. |
| **README** | Series overview and diagram index. |

---

### docs/access-logs/

| Document | Content covered |
|----------|------------------|
| **01-architecture-lifecycle** | Access log architecture and lifecycle. |
| **02-formatters-filters-grpc** | Formatters, filters, and gRPC access log. |
| **03-file-io-flushing** | File I/O and flushing behavior. |

---

### docs/security/

| Document | Content covered |
|----------|------------------|
| **01_tls_context_implementation** | TLS context implementation. |
| **02_sds_secret_updates** | SDS secret updates. |
| **03_alpn_sni_handling** | ALPN and SNI handling. |
| **04_rbac_internals** | RBAC internals. |
| **README** | Security docs index. |

---

### docs/ssl-tls/

| Document | Content covered |
|----------|------------------|
| **01-architecture-contexts** | SSL/TLS architecture and contexts. |
| **02-handshake-mtls-validation** | Handshake and mTLS validation. |
| **03-sds-rotation-ocsp** | SDS, rotation, and OCSP. |

---

### docs/tunneling/

| Document | Content covered |
|----------|------------------|
| **01-overview-http-connect** | HTTP CONNECT and tunneling overview. |
| **02-tcp-over-http-tunneling** | TCP over HTTP tunneling. |
| **03-internal-listeners** | Internal listeners. |

---

### docs/source-common/ (High-level source-common guide)

| Document | Content covered |
|----------|------------------|
| **01-overview** | Overview of source/common. |
| **02-http-connection-management** | HTTP connection management. |
| **03-http-codecs-headers-pools** | HTTP codecs, headers, pools. |
| **04-network-connections-sockets** | Network connections and sockets. |
| **05-network-listeners-filters** | Network listeners and filters. |
| **06-router-engine-config** | Router engine and config. |
| **07-router-upstream-retry** | Router, upstream, retry. |
| **08-upstream-cluster-manager** | Upstream cluster manager. |
| **09-upstream-hosts-health** | Upstream hosts and health. |
| **10-supporting-subsystems** | Supporting subsystems. |

---

### source/extensions/filters/listener/docs/ (Listener filters in depth)

| Document | Content covered |
|----------|------------------|
| **01-Overview-and-Architecture** | What listener filters are, where they run, ListenerFilter/ListenerFilterCallbacks/ListenerFilterManager UML, ActiveTcpSocket block diagram. |
| **02-Listener-Filter-Chain-Execution** | Execution model, ActiveTcpSocket lifecycle, sequence diagrams (happy path, TLS peek), ListenerFilterBuffer, timeout/error handling. |
| **03-Built-in-Listener-Filters** | Per-filter: TLS Inspector, Proxy Protocol, Original Dst, Original Src, HTTP Inspector, Local Rate Limit — block diagram, UML, sequence diagram; comparison table. |
| **04-Configuration-and-Extension** | Config flow from listener proto to addAcceptFilter, NamedListenerFilterConfigFactory, matchers, ECDS, dynamic modules. |

---

### source/common/http/ (HTTP layer implementation)

| Document | Content covered |
|----------|------------------|
| **OVERVIEW_PART1_request_pipeline** | Request pipeline overview. |
| **OVERVIEW_PART2_codecs_and_pools** | Codecs and connection pools. |
| **OVERVIEW_PART3_headers_and_utilities** | Headers and utilities. |
| **OVERVIEW_PART4_async_and_advanced** | Async and advanced HTTP behavior. |
| **filter_manager.md** | HTTP filter manager. |
| **codec_client.md** | Codec client. |
| **conn_manager_impl.md** | Connection manager implementation. |
| **conn_manager_utility.md** | Connection manager utilities. |
| **conn_pool_base_and_grid.md** | Connection pool base and grid. |
| **header_map_impl.md** | Header map implementation. |
| **async_client_impl.md** | Async client implementation. |

---

### source/common/listener_manager/

| Document | Content covered |
|----------|------------------|
| **OVERVIEW_PART1_architecture** | Listener manager architecture. |
| **OVERVIEW_PART2_filter_chains** | Filter chains. |
| **OVERVIEW_PART3_active_tcp** | Active TCP listener and sockets. |
| **OVERVIEW_PART4_lds_and_advanced** | LDS and advanced listener behavior. |
| **active_tcp_listener_and_socket.md** | ActiveTcpListener, ActiveTcpSocket, listener filter chain, connection balancing. |
| **filter_chain_manager_impl.md** | Filter chain manager implementation. |
| **listener_impl.md** | Listener implementation. |
| **listener_manager_impl.md** | Listener manager implementation. |
| **connection_handler_impl.md** | Connection handler implementation. |
| **lds_api.md** | LDS API. |

---

### source/common/network/

| Document | Content covered |
|----------|------------------|
| **OVERVIEW_PART1_architecture_and_connections** | Network architecture and connections. |
| **OVERVIEW_PART2_filters_and_listeners** | Filters and listeners. |
| **OVERVIEW_PART3_sockets_and_io** | Sockets and I/O. |
| **OVERVIEW_PART4_addressing_dns_and_utilities** | Addressing, DNS, utilities. |
| **connection_impl.md** | Connection implementation. |
| **filter_manager_impl.md** | Network filter manager implementation. |
| **listeners.md** | Listeners. |
| **socket_and_io_handle.md** | Socket and I/O handle. |
| **address_impl.md** | Address implementation. |
| **happy_eyeballs_connection_impl.md** | Happy eyeballs connection. |
| **transport_socket_options.md** | Transport socket options. |
| **source_common_network_README.md** | Network README. |

---

### source/common/upstream/

| Document | Content covered |
|----------|------------------|
| **OVERVIEW_PART1_architecture_and_cluster_manager** | Upstream architecture and cluster manager. |
| **OVERVIEW_PART2_clusters_hosts_priority_sets** | Clusters, hosts, priority sets. |
| **OVERVIEW_PART3_load_balancing_conn_pools_resources** | Load balancing, connection pools, resources. |
| **OVERVIEW_PART4_health_outlier_cds_advanced** | Health, outlier detection, CDS, advanced. |
| **cluster_manager_impl.md** | Cluster manager implementation. |
| **upstream_impl.md** | Upstream implementation. |
| **load_balancing.md** | Load balancing. |
| **health_checking.md** | Health checking. |
| **outlier_detection_impl.md** | Outlier detection implementation. |
| **cds_and_odcds.md** | CDS and ODCDS. |

---

### source/docs/observability/

| Document | Content covered |
|----------|------------------|
| **OVERVIEW.md** | Observability overview. |
| **README.md** | Observability README. |
| **access_logs.md** | Access logs. |
| **admin_interface.md** | Admin interface. |
| **stats_subsystem.md** | Stats subsystem. |
| **tracing_integration.md** | Tracing integration. |

---

### source/extensions/dynamic_modules/

| Document | Content covered |
|----------|------------------|
| **OVERVIEW_PART1_architecture_and_abi** | Dynamic modules architecture and ABI. |
| **OVERVIEW_PART2_http_filter_and_extensions** | HTTP filter and extensions. |
| **OVERVIEW_PART3_sdks_and_development** | SDKs and development. |
| **OVERVIEW_PART4_callbacks_metrics_advanced** | Callbacks, metrics, advanced topics. |
| **dynamic_modules_core.md** | Core dynamic modules. |
| **http_filter.md** | HTTP filter for dynamic modules. |
| **sdk_overview.md** | SDK overview. |

---

### tools/docs/

| Document | Content covered |
|----------|------------------|
| **01-Overview** | Tools overview. |
| **02-Code-Quality-Tools** | Code quality tools. |
| **03-API-and-Proto-Tools** | API and proto tools. |
| **04-Build-and-CI-Tools** | Build and CI tools. |
| **05-Debugging-and-Profiling-Tools** | Debugging and profiling. |
| **06-Dependency-Management-Tools** | Dependency management. |
| **07-IDE-and-Editor-Tools** | IDE and editor tools. |
| **08-GitHub-Release-and-Misc-Tools** | GitHub, release, misc tools. |
| **09-Adding-New-Tools** | Adding new tools. |

---

### docs/admin-operations/ (Admin Interface & Operations)

| Document | Content covered |
|----------|------------------|
| **01-admin-interface** | Complete admin endpoint reference (/stats, /config_dump, /clusters, /listeners, /runtime, /logging, /drain_listeners, /healthcheck, profiling endpoints, /quitquitquit, /certs, /help); configuration and security best practices. |
| **02-runtime-configuration** | Runtime configuration system; layered architecture (admin, disk, RTDS, static); common runtime keys (circuit breakers, health checks, outlier detection, retry, feature flags); admin interface operations (/runtime, /runtime_modify); gradual feature rollout patterns. |
| **03-operational-procedures** | Graceful shutdown and draining (immediate, graceful, inbound-only modes); hot restart mechanism; health check management; log management and rotation; statistics collection; configuration management; monitoring and alerting; troubleshooting operations. |
| **README** | Overview, quick reference commands, security guidance. |

---

### docs/extension-development/ (Extension Development)

| Document | Content covered |
|----------|------------------|
| **01-overview** | Extension system architecture; extension types (HTTP filters, network filters, listener filters, load balancers, transport sockets, access loggers, clusters, health checkers, tracers, stat sinks, resource monitors); factory pattern; extension lifecycle; configuration system; threading model. |
| **02-http-filter-development** | HTTP filter interfaces (StreamDecoderFilter, StreamEncoderFilter, StreamFilter); complete lifecycle and callbacks; configuration and factory implementation; route-specific config; common patterns (pass-through, buffering, async, header manipulation, body transformation, per-route config); testing patterns (unit and integration). |
| **03-network-filter-development** | Network filter interfaces (ReadFilter, WriteFilter, Filter); lifecycle; implementation examples; common patterns (protocol detection, rate limiting, protocol parsing, TCP proxy, async operations); testing examples. |
| **04-listener-filter-development** | ListenerFilter and QuicListenerFilter interfaces; listener filter lifecycle; common patterns (TLS SNI inspection, original destination recovery, proxy protocol parsing, early rejection, dynamic routing); testing examples. |
| **05-extension-registration** | Extension registry and registration macros; Bazel build system integration; BUILD file templates; protocol buffer setup; static vs dynamic registration; extension metadata; deployment strategies. |
| **README** | Quick start, extension types overview, development workflow, common patterns, best practices. |

---

### docs/testing/ (Testing Infrastructure)

| Document | Content covered |
|----------|------------------|
| **01-testing-overview** | Testing philosophy and principles; test types (unit, integration, benchmark, fuzz); Google Test and Google Mock frameworks; test infrastructure and utilities; time management in tests; custom matchers. |
| **02-unit-testing** | GoogleTest framework basics; test fixtures and setup patterns; mock objects (NiceMock, StrictMock); common mock classes from test/mocks/; testing patterns (protobufs, exceptions, callbacks); real examples from test/common/. |
| **03-integration-testing** | BaseIntegrationTest and HttpIntegrationTest hierarchy; basic integration test patterns; ConfigHelper; FakeUpstream usage; autonomous upstream; common patterns (waiting, stats, timeouts, retries); debugging techniques. |
| **04-filter-testing** | HTTP filter testing patterns; network filter testing; listener filter testing; filter factory testing; integration tests for filters; examples for all filter types. |
| **05-benchmarking** | Google Benchmark framework usage; writing benchmark tests; running benchmarks locally and in CI; benchmark patterns for buffers, headers, stats; best practices and common pitfalls; interpreting results. |
| **README** | Quick start guide, documentation structure, test locations, running commands, key concepts, best practices. |

---

### docs/memory-performance/ (Memory & Performance)

| Document | Content covered |
|----------|------------------|
| **01-threading-model** | Main thread vs worker threads architecture; Thread Local Storage (TLS) implementation; event loop (Dispatcher) design based on libevent; thread-safe design patterns; cross-thread communication via post(); configuration and tuning; debugging threading issues. |
| **02-buffer-management** | Buffer::Instance interface; slice-based architecture (16 KB slices); zero-copy operations (move, fragments, reservations); watermarks for flow control; memory accounting per-stream; performance optimizations; common usage patterns. |
| **03-memory-management** | TCMalloc integration and architecture; memory statistics and monitoring; heap profiling with pprof; AllocatorManager and heap shrinker; memory accounting classes; smart pointer usage patterns; leak detection tools (ASAN, Valgrind); configuration examples. |
| **04-performance-optimization** | Lock-free design patterns (TLS, atomics, RCU); zero-copy techniques; cache-friendly code patterns; efficient allocation strategies; profiling tools (pprof, perf, benchmarks); optimization workflow; best practices and anti-patterns. |
| **README** | Overview, quick start for developers and operators, common use cases, performance metrics, configuration examples, best practices, glossary. |

---

### docs/debugging/ (Debugging & Error Handling)

| Document | Content covered |
|----------|------------------|
| **01-error-handling-patterns** | Status and StatusOr usage patterns; error propagation; complete Response Flags reference (30+ flags); connection close reasons; response code details; common error scenarios; best practices. |
| **02-logging-and-tracing** | Logger system architecture; log levels (trace/debug/info/warn/error/critical); logging macros (ENVOY_LOG, ENVOY_CONN_LOG, ENVOY_STREAM_LOG, etc.); all 60+ logger IDs documented; fine-grain logging; connection and stream tracking; request tracing; rate-limited logging; runtime log configuration; debugging with logs. |
| **03-debugging-tools** | Admin interface for debugging; GDB usage with Envoy; core dump analysis; ASAN (AddressSanitizer); TSAN (ThreadSanitizer); performance profiling with perf and gperftools; network debugging tools (tcpdump, wireshark, strace, lsof); configuration debugging; debugging cheat sheet. |
| **04-common-issues** | Connection failures (DNS, wrong port, pool exhaustion, firewall); TLS/certificate issues (expired certs, validation, SNI, protocol, mTLS); configuration errors; performance problems; memory leaks; crash troubleshooting; timeout issues; health check failures; route matching problems; filter chain issues; troubleshooting checklist. |
| **README** | Quick start with health checks, documentation structure, troubleshooting flowchart, quick reference tables, 4 debugging scenarios, tools installation, FAQ, best practices. |

---

### docs/deployment/ (Deployment & Operations)

| Document | Content covered |
|----------|------------------|
| **01-deployment-patterns** | Edge proxy configuration with TLS, rate limiting, circuit breakers; service mesh sidecar with Istio; API gateway with JWT, external authz, rate limiting; internal load balancer with health checking; ingress controller with Kubernetes; pattern comparison and selection guidance; complete YAML configs for each pattern. |
| **02-hot-restart** | Hot restart architecture with diagrams; implementation details from source/server/hot_restart_impl.h; shared memory structure and RPC protocol; class hierarchy and sequence diagrams; operational procedures; systemd integration with service unit files; epoch calculation scripts; troubleshooting guide; limitations and gotchas. |
| **03-container-deployment** | Docker best practices with multi-stage builds; optimized Dockerfiles (including distroless); Docker Compose examples; complete Kubernetes manifests with HPA and PDB; sidecar pattern implementations; init containers (iptables, validation, cert fetching); Istio sidecar injection (automatic and manual); lifecycle management with preStop hooks; resource limits with calculation guidelines; health checks (liveness, readiness, startup probes). |
| **04-production-hardening** | Security hardening (least privilege, non-root, TLS 1.3); secrets management (External Secrets, SDS); access control (IP whitelisting, JWT, RBAC); rate limiting (local and global); security headers; Kubernetes Network Policies; resource limits and performance tuning; overload manager; monitoring setup (Prometheus, metrics, tracing, access logs); alert rules; AlertManager config; disaster recovery; SRE runbooks for common incidents. |
| **README** | Overview, links to all guides, quick start information. |

---

### docs/quic-udp/ (QUIC & UDP)

| Document | Content covered |
|----------|------------------|
| **01-quic-overview** | QUIC protocol fundamentals and benefits; detailed architecture diagrams; server and client-side QUIC components; connection management and lifecycle; stream multiplexing; integration with HTTP Connection Manager; security and performance characteristics. |
| **02-quic-implementation** | Deep dive into address caching optimization (from io_socket_handle_impl.h:106-107) with LRU cache implementation and performance analysis; QuicServerTransportSocketFactory; QuicFilterManagerConnectionImpl; EnvoyQuicServerSession and EnvoyQuicClientSession; buffer management and watermarks; connection migration implementation; stream management; packet writing and GSO support. |
| **03-udp-listeners** | UDP listener architecture; UdpListenerImpl implementation; session management (sticky and per-packet); UDP session filters; UDP tunneling (CONNECT-UDP and POST methods); DNS resolution over UDP; performance optimizations (GRO/GSO); observability and troubleshooting. |
| **04-configuration-examples** | HTTP/3 listener configurations; QUIC client cluster configurations; UDP proxy examples (DNS, gaming, VoIP); UDP tunneling configurations; performance tuning examples; TLS/mTLS configurations; complete production-ready examples; monitoring and debugging guides. |
| **README** | Overview, documentation structure, architecture highlights with diagrams, key features, use cases. |

---

### Other uncommitted .md (reference)

- **envoy/network/README.md** — Network layer README.
- **source/extensions/bootstrap/reverse_tunnel/.../README.md** — Reverse tunnel downstream socket.
- **source/extensions/filters/http/decompressor/README.md** — Decompressor filter.
- **source/extensions/filters/http/local_ratelimit/README_DESCRIPTOR_IDENTIFICATION.md** — Local rate limit descriptor identification.
- **tools/vscode/README.md** (modified) — VSCode setup.
- **tools/vscode/README_SCRIPT.md** — VSCode script.

---

## 3. Document Locations Summary

| Folder / area | Count (approx.) | Purpose |
|---------------|------------------|---------|
| **docs/api-envoy/** | 5 | Envoy API (proto) reference |
| **docs/core-runtime-xds-istio/** | 4 | Core, runtime, xDS, Istio |
| **docs/filter-chain/** | 11 | Filter chain creation and diagrams |
| **docs/request-flow/** | 12 | Request receive path |
| **docs/flows/** | 6 | Flow/UML diagrams |
| **docs/http_filters/** | 11 | HTTP filter docs |
| **docs/istio-envoy-config/** | 7 | Istio → Envoy config |
| **docs/access-logs/** | 3 | Access log implementation |
| **docs/security/** | 5 | TLS, SDS, ALPN/SNI, RBAC |
| **docs/ssl-tls/** | 3 | SSL/TLS deep dive |
| **docs/tunneling/** | 3 | Tunneling (CONNECT, internal) |
| **docs/source-common/** | 10 | source/common guide |
| **source/.../listener/docs/** | 4 | Listener filters in depth |
| **source/common/http/** | 11 | HTTP implementation |
| **source/common/listener_manager/** | 11 | Listener manager implementation |
| **source/common/network/** | 12 | Network implementation |
| **source/common/upstream/** | 12 | Upstream implementation |
| **source/docs/observability/** | 6 | Observability |
| **source/extensions/dynamic_modules/** | 7 | Dynamic modules |
| **tools/docs/** | 9 | Development/build tools |
| **docs/admin-operations/** | 4 | Admin interface, runtime config, operational procedures |
| **docs/extension-development/** | 6 | Writing HTTP/network/listener filters, registration |
| **docs/testing/** | 6 | Unit tests, integration tests, benchmarking |
| **docs/memory-performance/** | 5 | Threading, buffers, memory management, optimization |
| **docs/debugging/** | 5 | Error handling, logging, debugging tools, common issues |
| **docs/deployment/** | 5 | Deployment patterns, hot restart, containers, hardening |
| **docs/quic-udp/** | 5 | QUIC/HTTP3, UDP listeners, configuration |

Use **Section 1** to find which doc covers a topic; use **Section 2** to see what each document or series contains.
