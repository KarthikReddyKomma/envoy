# Admin Interface and Operations

This directory contains documentation for Envoy's administrative interface and operational procedures.

## Contents

### [01-admin-interface.md](01-admin-interface.md)
Complete reference for Envoy's admin HTTP interface including all endpoints, parameters, and use cases.

**Topics**:
- Admin interface architecture
- All admin endpoints (stats, config, clusters, listeners, etc.)
- Configuration and security
- Common operational patterns

### [02-runtime-configuration.md](02-runtime-configuration.md)
Guide to Envoy's runtime configuration system for dynamic behavior changes without restarts.

**Topics**:
- Runtime layer architecture
- Configuration methods (static, disk, RTDS, admin)
- Common runtime keys and flags
- Feature flag patterns
- Best practices

### [03-operational-procedures.md](03-operational-procedures.md)
Standard operating procedures for managing Envoy in production.

**Topics**:
- Graceful shutdown and draining
- Hot restart for zero-downtime updates
- Health check management
- Log management and rotation
- Statistics collection
- Configuration management
- Monitoring and alerting
- Troubleshooting

## Quick Reference

### Most Common Operations

**Check server status**:
```bash
curl http://localhost:9901/server_info
```

**View all stats**:
```bash
curl http://localhost:9901/stats
```

**Dump current config**:
```bash
curl http://localhost:9901/config_dump
```

**Check cluster health**:
```bash
curl http://localhost:9901/clusters
```

**Graceful shutdown**:
```bash
curl -X POST "http://localhost:9901/drain_listeners?graceful=true"
```

**Adjust logging**:
```bash
curl -X POST "http://localhost:9901/logging?level=debug"
```

**Modify runtime value**:
```bash
curl -X POST "http://localhost:9901/runtime_modify?key=value"
```

## Admin Port

Default: `9901` (configurable in bootstrap config)

**Security**: Always bind to localhost or internal network only in production.

## Related Documentation

- [Envoy Architecture Reference](../ENVOY_ARCHITECTURE_REFERENCE.md)
- [Source Common](../source-common/) - Implementation details
- [Observability](../../source/docs/observability/) - Stats, logging, tracing
