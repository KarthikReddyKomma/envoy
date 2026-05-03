# Environment resource detector (`envoy.tracers.opentelemetry.resource_detectors.environment_resource_detector`)

Reads OpenTelemetry resource attributes from the `OTEL_RESOURCE_ATTRIBUTES`
environment variable as specified by the OTel SDK. The variable is a
comma-separated list of `key=value` pairs; each pair becomes a resource
attribute.

Proto: `envoy.extensions.tracers.opentelemetry.resource_detectors.v3.EnvironmentResourceDetectorConfig`.

## Files
- `config.h/cc` - `EnvironmentResourceDetectorFactory` registered in the
  `envoy.tracers.opentelemetry.resource_detectors` category. Builds an
  `EnvironmentResourceDetector` with the server factory context (needed
  for the `DataSource` API).
- `environment_resource_detector.h/cc` -
  `EnvironmentResourceDetector::detect` reads
  `OTEL_RESOURCE_ATTRIBUTES` through `Config::DataSource::read` with the
  `environment_variable` variant, then splits on `,` and on `=`.

## Tracer role
Not a tracer. Runs once per OTel driver construction; feeds attributes
into the shared `Resource` attached to every
`ExportTraceServiceRequest`.

## Flow
- Build a `DataSource` with `environment_variable =
  OTEL_RESOURCE_ATTRIBUTES`.
- Read it through `Config::DataSource::read(..., true, api)` (the `true`
  allows env var reads).
- Empty / missing -> return an empty `Resource` (the static service.name
  and telemetry.sdk.* attributes are still added upstream by
  `ResourceProviderImpl`).
- Split on `,`, reject entries whose `=`-split is not exactly two parts
  (log `warn`), otherwise write into `resource.attributes_`.

## Key decision points
- `environment_resource_detector.cc:14` -
  `kOtelResourceAttributesEnv = "OTEL_RESOURCE_ATTRIBUTES"`.
- `environment_resource_detector.cc:32-37` - errors from `DataSource::read`
  are caught and logged at `warn`; the detector then returns whatever
  `attributes_` has collected so far (which is typically empty).
- `environment_resource_detector.cc:44-49` - malformed `k=v` entries are
  skipped with a `warn` log instead of failing the listener.

## Configuration
None beyond selecting the detector. All input comes from the environment.

## Stats / errors
No stats. Reads and malformed-entry conditions log at `warn`. No
exceptions are propagated.
