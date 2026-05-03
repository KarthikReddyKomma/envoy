# Dynatrace resource detector (`envoy.tracers.opentelemetry.resource_detectors.dynatrace_resource_detector`)

Adds OpenTelemetry resource attributes read from the Dynatrace OneAgent
metadata enrichment files. When OneAgent is monitoring the host / process,
it exposes a small set of virtual files whose contents are a
`key=value\n...` payload; this detector parses those files into
`ResourceAttributes`.

Proto: `envoy.extensions.tracers.opentelemetry.resource_detectors.v3.DynatraceResourceDetectorConfig`.

## Files
- `config.h/cc` - `DynatraceResourceDetectorFactory` registered in the
  `envoy.tracers.opentelemetry.resource_detectors` category. Builds a
  `DynatraceResourceDetector` with the default
  `DynatraceMetadataFileReaderImpl`.
- `dynatrace_resource_detector.h/cc` - `DynatraceResourceDetector::detect`
  iterates the hardcoded `dynatraceMetadataFiles()` list, reads each
  through the file reader, and populates `Resource.attributes_` from the
  `key=value` lines.
- `dynatrace_metadata_file_reader.h/cc` -
  `DynatraceMetadataFileReader` interface + the default filesystem-backed
  implementation.

## Tracer role
Not a tracer. Runs once per OTel driver construction; feeds attributes
into the shared `Resource`, which is attached to every
`ExportTraceServiceRequest`.

## Flow
- `detect()` loops over `dynatraceMetadataFiles()`.
- Each file is read via `DynatraceMetadataFileReader::readEnrichmentFile`.
- `addAttributes` splits on `\n`, skips blank lines, and splits each line
  on `=` into exactly two parts; anything else is ignored silently.
- If every configured file comes back empty or throws, a `warn` log is
  emitted pointing the operator at the Dynatrace deployment status.

## Key decision points
- `dynatrace_resource_detector.cc:14-19` - malformed lines (not exactly
  `k=v`) are dropped without error.
- `dynatrace_resource_detector.cc:49-54` - the "all files failed" warning.
- `dynatrace_metadata_file_reader.h:20` - the reader intentionally relies
  on OneAgent's virtual-file mechanism; absence of OneAgent produces empty
  reads, which is the expected "not deployed" signal.

## Configuration
None; the detector has no tunables beyond selecting it in the OTel config.

## Stats / errors
No stats. Per-file read failures log at `debug`; global failure logs at
`warn`. No exceptions are propagated to the driver.
