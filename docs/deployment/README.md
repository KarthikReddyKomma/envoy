# Deployment and Operations Documentation

This directory contains comprehensive guides for deploying and operating Envoy in production environments.

## Contents

### 1. [Deployment Patterns](01-deployment-patterns.md)
Common deployment patterns for Envoy including:
- Edge proxy (internet-facing load balancer)
- Service mesh sidecar (Istio, Consul Connect)
- API gateway (rate limiting, authentication, routing)
- Internal load balancer
- Ingress controller (Kubernetes)
- Example configurations for each pattern

### 2. [Hot Restart](02-hot-restart.md)
Deep dive into Envoy's zero-downtime hot restart mechanism:
- Architecture (shared memory, socket passing, RPC protocol)
- Implementation details from the source code
- Step-by-step operational procedures
- Limitations and troubleshooting
- Systemd integration examples

### 3. [Container Deployment](03-container-deployment.md)
Best practices for deploying Envoy in containerized environments:
- Docker image optimization
- Kubernetes deployment manifests
- Init containers and sidecar patterns
- Istio sidecar injection
- Lifecycle management (preStop hooks)
- Resource limits and requests
- Health checks and readiness probes

### 4. [Production Hardening](04-production-hardening.md)
Production readiness checklist and best practices:
- Security hardening (least privilege, TLS, secrets)
- Resource limits and performance tuning
- Monitoring and observability setup
- Alert configuration and SLOs
- Disaster recovery procedures
- SRE runbooks and troubleshooting guides

## Related Documentation

- [Admin Interface](../admin-operations/01-admin-interface.md) - Admin API endpoints and usage
- [Runtime Configuration](../admin-operations/02-runtime-configuration.md) - Dynamic configuration updates
- [Operational Procedures](../admin-operations/03-operational-procedures.md) - Day-2 operations

## Quick Start

For a quick deployment example, see the [edge proxy pattern](01-deployment-patterns.md#edge-proxy) or [Kubernetes sidecar](03-container-deployment.md#kubernetes-sidecar-deployment).

## Prerequisites

Before deploying Envoy in production, ensure you understand:
- Basic Envoy concepts (listeners, routes, clusters)
- Your deployment pattern requirements
- Observability and monitoring requirements
- Security and compliance requirements

## Contributing

When adding new deployment patterns or operational procedures:
1. Include practical, tested examples
2. Provide YAML/JSON configuration snippets
3. Document known limitations and gotchas
4. Include troubleshooting steps
5. Reference source code where relevant
