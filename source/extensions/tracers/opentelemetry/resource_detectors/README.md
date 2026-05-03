# OpenTelemetry resource detectors

Extension category: `envoy.tracers.opentelemetry.resource_detectors`.

Resource detectors produce the OTLP `Resource` attached to every exported
span batch. The OpenTelemetry tracer driver feeds the list of
`TypedExtensionConfig` entries from
`OpenTelemetryConfig.resource_detectors` into `ResourceProviderImpl::
getResource`, which instantiates each detector, calls `detect()`, and
merges the result into an initial resource seeded with `service.name`,
`telemetry.sdk.language`, `telemetry.sdk.name`, and
`telemetry.sdk.version` (see `resource_provider.cc:17-30`).

Proto: the extension category uses a `TypedExtensionConfig` pointing at any
config proto registered under
`envoy.extensions.tracers.opentelemetry.resource_detectors.v3.*`.

## Files
- `resource_detector.h` - `Resource` struct (`schema_url_` +
  `ResourceAttributes` hash map), the `ResourceDetector` base with a single
  `detect()` method, and the `ResourceDetectorFactory` typed factory used
  for extension registration (category
  `envoy.tracers.opentelemetry.resource_detectors`).
- `resource_provider.h/cc` - `ResourceProviderImpl::getResource` iterates
  the configured detectors, merges their attributes, and resolves
  `schema_url` per the OTel spec (empty wins, equal wins, conflict falls
  back to the first value with a log warning).
- `dynatrace/`, `environment/`, `static/` - concrete detectors, each with
  its own README.

## Tracer role
Not a tracer on its own. The provider is called once when the driver is
built (`opentelemetry_tracer_impl.cc:79`) and the resulting
`ResourceConstSharedPtr` is stored on the `Tracer` and emitted on every
export batch.

## Flow
- Driver construction -> `ResourceProviderImpl::getResource(resource_
  detectors, ctx, service_name)`.
- `createInitialResource` seeds the static attributes.
- For each `TypedExtensionConfig`: look up factory via
  `Envoy::Config::Utility::getFactory<ResourceDetectorFactory>`, call
  `createResourceDetector`, then `detector->detect()`.
- `mergeResource` applies every detected attribute with `insert_or_assign`
  (later detectors overwrite earlier keys) and resolves `schema_url`.

## Key decision points
- `resource_provider.cc:43-58` - `resolveSchemaUrl` rules for conflicting
  schema URLs (keep the old one, log at info).
- `resource_provider.cc:79-82` - `mergeResource` overwrite semantics.
- `resource_provider.cc:97` / `resource_provider.cc:104` - unknown factory
  or null detector both throw `EnvoyException`, aborting listener config
  load.

## Configuration
Driven entirely by `OpenTelemetryConfig.resource_detectors`, a repeated
`TypedExtensionConfig`. Order matters (later entries overwrite earlier
ones).

## Stats / errors
No stats. Failures at registration time surface as `EnvoyException` and
abort startup.
