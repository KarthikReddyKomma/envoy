# Static resource detector (`envoy.tracers.opentelemetry.resource_detectors.static_config_resource_detector`)

Adds OpenTelemetry resource attributes from a static `attributes` map in the
bootstrap config. The simplest detector - no environment, no files. Use
when the attributes are known at deploy time.

Proto: `envoy.extensions.tracers.opentelemetry.resource_detectors.v3.StaticConfigResourceDetectorConfig`.

## Files
- `config.h/cc` - `StaticConfigResourceDetectorFactory` registered in the
  `envoy.tracers.opentelemetry.resource_detectors` category. Validates the
  typed config and constructs a `StaticConfigResourceDetector` initialized
  with the proto's `attributes` map.
- `static_config_resource_detector.h/cc` -
  `StaticConfigResourceDetector::detect` copies the stored
  `attributes_` map into a fresh `Resource`, skipping any entry whose
  value is empty (with a `warn` log).

## Tracer role
Not a tracer. Runs once per OTel driver construction; feeds attributes
into the shared `Resource` attached to every
`ExportTraceServiceRequest`.

## Flow
- Config -> factory -> constructor captures `attributes_` from the proto
  into an `absl::flat_hash_map`.
- `detect()` iterates the map, copying non-empty values, returning the
  populated `Resource` (no schema URL).

## Key decision points
- `static_config_resource_detector.cc:23-27` - empty values are dropped
  with a `warn` log rather than propagated as empty strings.

## Configuration
- `attributes` - map<string, string> copied verbatim into resource
  attributes.

## Stats / errors
No stats. Empty-value entries log at `warn`. No exceptions propagated.
