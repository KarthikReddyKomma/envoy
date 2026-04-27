# Access Log Filters

## Introduction

Access log filters determine which requests should be logged. They are applied before the log entry is formatted and written, making them efficient for reducing log volume and cost.

**Location**: `source/extensions/access_loggers/filters/`

## Why Use Filters?

- **Reduce Cost**: Log only what matters (errors, slow requests, etc.)
- **Improve Performance**: Skip formatting and I/O for filtered requests
- **Focus on Important Events**: Errors, anomalies, or sampled traffic
- **Comply with Privacy**: Exclude health checks, internal traffic

## Filter Types

### 1. CEL Filter (Common Expression Language)

**Location**: `filters/cel/`

Most powerful filter using Google's Common Expression Language.

#### Features
- Rich expression syntax
- Access to all request/response data
- Logical operators (&&, ||, !)
- Comparison operators (==, !=, <, >, <=, >=)
- String operations
- Type safe

#### Configuration

```yaml
filter:
  extension_filter:
    name: envoy.access_loggers.extension_filters.cel
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter
      expression: "response.code >= 400"
```

#### Examples

**Log only errors (4xx and 5xx)**:
```yaml
expression: "response.code >= 400"
```

**Log slow requests (>1 second)**:
```yaml
expression: "request.duration > duration('1s')"
```

**Complex conditions**:
```yaml
expression: |
  (response.code >= 500) ||
  (request.duration > duration('1s')) ||
  (response.flags.contains('UH'))
```

**Exclude health checks**:
```yaml
expression: "!request.path.startsWith('/healthz')"
```

**Log only specific methods**:
```yaml
expression: "request.method in ['POST', 'PUT', 'DELETE']"
```

**Combine multiple conditions**:
```yaml
expression: |
  response.code >= 400 &&
  !request.path.startsWith('/internal') &&
  request.headers['x-user-type'] == 'external'
```

#### Available Fields

```
request.path           # Request path
request.method         # HTTP method
request.headers        # Request headers (map)
request.duration       # Request duration
request.size           # Request size in bytes
request.protocol       # HTTP protocol version

response.code          # Response status code
response.code_details  # Response code details
response.headers       # Response headers (map)
response.trailers      # Response trailers (map)
response.flags         # Response flags (string)
response.size          # Response size in bytes

connection.id          # Connection ID
connection.remote_address  # Client address
connection.local_address   # Envoy address

upstream.cluster       # Upstream cluster name
upstream.host          # Upstream host address
```

#### Performance

- CEL expressions are compiled once at startup
- Evaluation is fast (~1-5μs per filter check)
- More complex expressions take longer

---

### 2. Built-in Filters

Envoy provides many built-in filters without CEL:

#### Status Code Filter

```yaml
filter:
  status_code_filter:
    comparison:
      op: GE  # Greater than or equal
      value:
        default_value: 400
        runtime_key: access_log.min_status_code  # Optional runtime override
```

**Operators**: EQ, GE, LE

**Use Cases**:
- Log only errors: `op: GE, value: 400`
- Log only 5xx: `op: GE, value: 500`
- Log only success: `op: LT, value: 400`

#### Duration Filter

```yaml
filter:
  duration_filter:
    comparison:
      op: GE
      value:
        default_value: 1000  # milliseconds
        runtime_key: access_log.min_duration
```

**Use Cases**:
- Log slow requests: `op: GE, value: 1000` (>1s)
- Log very fast requests: `op: LE, value: 10` (<10ms)

#### Not Health Check Filter

```yaml
filter:
  not_health_check_filter: {}
```

Excludes requests marked as health checks.

#### Header Filter

```yaml
filter:
  header_filter:
    header:
      name: "x-log-this"
      string_match:
        exact: "true"
```

**Use Cases**:
- Log only requests with specific header
- Exclude requests with specific header (use not_filter wrapper)

#### Response Flag Filter

```yaml
filter:
  response_flag_filter:
    flags:
      - UH  # No healthy upstream
      - UF  # Upstream connection failure
      - DC  # Downstream connection termination
```

**Common Flags**:
- `UH` - No healthy upstream
- `UF` - Upstream connection failure  
- `UO` - Upstream overflow (circuit breaker)
- `UR` - Upstream retry limit exceeded
- `DC` - Downstream connection termination
- `LH` - Local service failed health check
- `UT` - Upstream request timeout
- `LR` - Connection local reset
- `RL` - Rate limited
- `DI` - Delay injection
- `FI` - Fault injection

#### Runtime Filter (Sampling)

```yaml
filter:
  runtime_filter:
    runtime_key: access_log.sampling_rate
    percent_sampled:
      numerator: 10  # 10%
      denominator: HUNDRED  # Default
```

**Use Cases**:
- Sample 1%: `numerator: 1`
- Sample 10%: `numerator: 10`
- Sample 0.1%: `numerator: 1, denominator: THOUSAND`

---

### 3. Composite Filters

Combine multiple filters with logical operators.

#### AND Filter

All conditions must be true:

```yaml
filter:
  and_filter:
    filters:
      - status_code_filter:
          comparison:
            op: GE
            value: 400
      - duration_filter:
          comparison:
            op: GE
            value: 1000
      - not_health_check_filter: {}
```

**Result**: Logs slow errors (>=400 status AND >=1s duration) excluding health checks.

#### OR Filter

At least one condition must be true:

```yaml
filter:
  or_filter:
    filters:
      # Log errors
      - status_code_filter:
          comparison:
            op: GE
            value: 400
      # Log slow requests
      - duration_filter:
          comparison:
            op: GE
            value: 1000
      # Log requests with upstream failures
      - response_flag_filter:
          flags: ["UH", "UF", "UO"]
```

**Result**: Logs errors OR slow requests OR upstream failures.

#### NOT Filter

Inverts the condition:

```yaml
filter:
  not_filter:
    filter:
      header_filter:
        header:
          name: "x-debug"
          string_match:
            exact: "true"
```

**Result**: Logs all requests EXCEPT those with `x-debug: true`.

---

### 4. Process Rate Limit Filter

**Location**: `filters/process_ratelimit/`

Rate limits log output at the Envoy process level (not per request).

```yaml
filter:
  extension_filter:
    name: envoy.access_loggers.extension_filters.process_ratelimit
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.process_ratelimit.v3.ProcessRateLimitConfig
      # Log 1 out of every 100 requests
      sample_rate: 100
```

**How It Works**:
- Global counter shared across all threads
- Logs 1 request per `sample_rate` requests
- Different from runtime filter (which samples randomly)
- Deterministic: every Nth request is logged

**Use Cases**:
- Consistent sampling for cost control
- Predictable log volume
- Testing (sample_rate: 1 logs everything)

**Performance**:
- Very fast (atomic counter increment)
- Minimal overhead

---

## Common Patterns

### Pattern 1: Log All Errors, Sample Success

```yaml
filter:
  or_filter:
    filters:
      # Always log errors
      - status_code_filter:
          comparison:
            op: GE
            value: 400
      # Sample 1% of success
      - and_filter:
          filters:
            - status_code_filter:
                comparison:
                  op: LT
                  value: 400
            - runtime_filter:
                percent_sampled:
                  numerator: 1
```

### Pattern 2: Log Anomalies

```yaml
filter:
  or_filter:
    filters:
      # Errors
      - status_code_filter:
          comparison:
            op: GE
            value: 400
      # Slow requests
      - duration_filter:
          comparison:
            op: GE
            value: 1000
      # Upstream issues
      - response_flag_filter:
          flags: ["UH", "UF", "UO", "UT"]
      # Large requests/responses
      - extension_filter:
          name: envoy.access_loggers.extension_filters.cel
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter
            expression: "request.size > 1000000 || response.size > 1000000"
```

### Pattern 3: Exclude Internal Traffic

```yaml
filter:
  and_filter:
    filters:
      # Exclude health checks
      - not_health_check_filter: {}
      # Exclude internal paths
      - extension_filter:
          name: envoy.access_loggers.extension_filters.cel
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter
            expression: "!request.path.startsWith('/internal')"
      # Exclude internal IP addresses
      - extension_filter:
          name: envoy.access_loggers.extension_filters.cel
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter
            expression: "!connection.remote_address.startsWith('10.')"
```

### Pattern 4: Progressive Sampling

```yaml
access_log:
  # Log all 5xx errors
  - name: envoy.access_loggers.file
    filter:
      status_code_filter:
        comparison:
          op: GE
          value: 500
    typed_config:
      path: /var/log/envoy/errors.log
  
  # Log 10% of 4xx errors
  - name: envoy.access_loggers.file
    filter:
      and_filter:
        filters:
          - status_code_filter:
              comparison:
                op: GE
                value: 400
          - status_code_filter:
              comparison:
                op: LT
                value: 500
          - runtime_filter:
              percent_sampled:
                numerator: 10
    typed_config:
      path: /var/log/envoy/client_errors.log
  
  # Log 1% of success
  - name: envoy.access_loggers.file
    filter:
      and_filter:
        filters:
          - status_code_filter:
              comparison:
                op: LT
                value: 400
          - runtime_filter:
              percent_sampled:
                numerator: 1
    typed_config:
      path: /var/log/envoy/success.log
```

## Filter Performance

### Evaluation Order

Filters are evaluated in the order defined. Optimize by putting cheap filters first:

```yaml
# GOOD: Cheap filter first
filter:
  and_filter:
    filters:
      - not_health_check_filter: {}  # Fast
      - extension_filter:            # Slower (CEL evaluation)
          name: envoy.access_loggers.extension_filters.cel
          typed_config:
            expression: "complex expression here"

# BAD: Expensive filter first
filter:
  and_filter:
    filters:
      - extension_filter:            # Evaluated even for health checks!
          name: envoy.access_loggers.extension_filters.cel
          typed_config:
            expression: "complex expression here"
      - not_health_check_filter: {}
```

### Short-Circuit Evaluation

- `and_filter`: Stops at first false
- `or_filter`: Stops at first true

Use this to optimize:

```yaml
# Evaluates status_code first (fastest)
# If false, skips CEL evaluation
filter:
  and_filter:
    filters:
      - status_code_filter:
          comparison:
            op: GE
            value: 400
      - extension_filter:
          name: envoy.access_loggers.extension_filters.cel
          typed_config:
            expression: "expensive evaluation"
```

### Filter Cost

| Filter Type | Cost | Notes |
|-------------|------|-------|
| `not_health_check_filter` | ~0.1μs | Very fast (simple flag check) |
| `status_code_filter` | ~0.1μs | Very fast (integer comparison) |
| `duration_filter` | ~0.1μs | Very fast (integer comparison) |
| `runtime_filter` | ~0.5μs | Fast (random number generation) |
| `process_ratelimit_filter` | ~0.2μs | Fast (atomic increment) |
| `header_filter` | ~1μs | Fast (hash map lookup) |
| `response_flag_filter` | ~0.5μs | Fast (bitfield check) |
| `extension_filter` (CEL) | ~1-10μs | Depends on expression complexity |

## Best Practices

### 1. Always Use Filters

Never log everything in production:

```yaml
# BAD: No filter
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      path: /var/log/envoy/access.log

# GOOD: Filtered
access_log:
  - name: envoy.access_loggers.file
    filter:
      or_filter:
        filters:
          - status_code_filter:
              comparison:
                op: GE
                value: 400
          - runtime_filter:
              percent_sampled:
                numerator: 1
    typed_config:
      path: /var/log/envoy/access.log
```

### 2. Use Runtime Filters for Dynamic Control

```yaml
filter:
  runtime_filter:
    runtime_key: access_log.sample_rate
    percent_sampled:
      numerator: 10
```

Then adjust at runtime without restart:
```bash
# Increase sampling to 50%
curl -X POST http://envoy-admin:9901/runtime_modify?key=access_log.sample_rate&value=50

# Disable logging
curl -X POST http://envoy-admin:9901/runtime_modify?key=access_log.sample_rate&value=0
```

### 3. Monitor Filter Effectiveness

Check how many requests are being filtered:

```
# Total requests
http.ingress.downstream_rq_total

# Actual logs (depends on logger stats)
access_logs.my_logger.logs_written

# Filter ratio
(http.ingress.downstream_rq_total - access_logs.my_logger.logs_written) / http.ingress.downstream_rq_total
```

### 4. Test Filters Before Deploying

Use a test filter to verify behavior:

```yaml
# Test: log filter decisions to a separate file
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      path: /tmp/filter_test.log
      log_format:
        json_format:
          path: "%REQ(:PATH)%"
          status: "%RESPONSE_CODE%"
          duration: "%DURATION%"
          # Include fields you're filtering on
```

### 5. Use CEL for Complex Logic

When built-in filters aren't enough:

```yaml
filter:
  extension_filter:
    name: envoy.access_loggers.extension_filters.cel
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.filters.cel.v3.ExpressionFilter
      expression: |
        (response.code >= 400 || request.duration > duration('1s')) &&
        !request.path.startsWith('/healthz') &&
        !request.headers['x-debug'].matches('skip-logging')
```

### 6. Different Filters for Different Loggers

```yaml
access_log:
  # Errors to error log (no sampling)
  - name: envoy.access_loggers.file
    filter:
      status_code_filter:
        comparison:
          op: GE
          value: 400
    typed_config:
      path: /var/log/envoy/errors.log
  
  # All requests to central logging (with sampling)
  - name: envoy.access_loggers.open_telemetry
    filter:
      runtime_filter:
        percent_sampled:
          numerator: 5
    typed_config:
      # ... OTLP config ...
```

## Related Documentation

- [ACCESS_LOGGERS_OVERVIEW.md](../ACCESS_LOGGERS_OVERVIEW.md)
- [FILE_ACCESS_LOGGER.md](../file/FILE_ACCESS_LOGGER.md)
- [GRPC_ACCESS_LOGGER.md](../grpc/GRPC_ACCESS_LOGGER.md)
- [OPENTELEMETRY_ACCESS_LOGGER.md](../open_telemetry/OPENTELEMETRY_ACCESS_LOGGER.md)

## Summary

Access log filters are essential for production deployments:

**Key Takeaways**:
- Always use filters to reduce cost and improve performance
- Use CEL for complex conditions
- Combine filters with AND/OR/NOT for sophisticated logic
- Put cheap filters first for efficiency
- Use runtime filters for dynamic control
- Test filters before deploying to production
- Monitor filter effectiveness with stats
