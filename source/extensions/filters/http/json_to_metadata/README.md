# JSON to Metadata (`envoy.filters.http.json_to_metadata`)

A stream filter that buffers the request and/or response body, parses it as JSON, walks a configured selector path, and writes the resulting values into the stream's dynamic metadata. Supports `on_present`, `on_missing`, and `on_error` actions per rule, with value coercion into string / number / protobuf-Value. Content-Type allow-lists and regex matchers control which payloads are inspected. Supports per-route overrides.

Proto: `envoy.extensions.filters.http.json_to_metadata.v3.JsonToMetadata`.

## Files
- `config.h/cc` — `JsonToMetadataConfig` (`ExceptionFreeFactoryBase`) that builds the shared `FilterConfig` and registers `envoy.filters.http.json_to_metadata` (`config.cc:37`). Also yields per-route configs (`config.cc:27-32`).
- `filter.h/cc` — `Rule`, `FilterConfig` (stats, rule vectors, content-type allow-list + regex), and `Filter` (`Http::PassThroughFilter`). Contains the JSON value converters used by `handleOnPresent`.

## Lifecycle
- `Filter` extends `Http::PassThroughFilter` (`filter.h:100-111`). Added with `addStreamFilter` (`config.cc:23`) so it participates in both decode and encode.
- `decodeHeaders` (`filter.cc:418-437`): returns `Continue` if no request rules; if the `Content-Type` is not allowed, it triggers `handleAllOnError` and increments `mismatched_content_type`; if `end_stream` is already set, it runs `handleAllOnMissing` and increments `no_body`; otherwise returns `StopIteration` to start buffering.
- `decodeData` (`filter.cc:460-484`): buffers via `addDecodedData`; on `end_stream`, if the buffer is empty/absent increments `no_body` and calls `handleAllOnMissing`, else calls `processRequestBody`; intermediate chunks return `StopIterationAndBuffer`.
- `decodeTrailers` (`filter.cc:512-521`): if processing isn't already done, runs `processRequestBody` then `Continue`.
- `encodeHeaders` / `encodeData` / `encodeTrailers` (`filter.cc:439-458`, `486-510`, `523-532`) mirror the decode-side logic against response rules and response content-type settings; the `should_clear_route_cache` argument to `processBody` is `true` for the request side and `false` for the response side (`filter.cc:406-416`).
- `request_processing_finished_` / `response_processing_finished_` guard against double-processing when both data and trailers arrive.

## Decision / logic
- Effective config: `getConfig` (`filter.cc:534-548`) caches `effective_config_`, preferring `resolveMostSpecificPerFilterConfig<FilterConfig>` on the decoder callbacks.
- Content-Type gate: `requestContentTypeAllowed` / `responseContentTypeAllowed` (`filter.cc:161-179`) accept an empty CT only if `allow_empty_content_type`; otherwise the literal set plus the compiled regex matcher. Default allow-set is `{application/json}` when none is configured (`filter.cc:64-75`).
- Body processing: `processBody` (`filter.cc:342-404`)
  - Missing / empty body -> `handleAllOnMissing` + `no_body`.
  - JSON parse error -> `handleAllOnError` + `invalid_json_body` (`filter.cc:353-359`).
  - Valid JSON but not an object (bare string / number) -> `handleAllOnMissing` plus `success` (`filter.cc:365-373`).
  - Otherwise, per rule, walk keys with `getObject` up to the last key; on any missing node call `handleOnMissing` (`filter.cc:380-388`); on the leaf call `handleOnPresent`, which falls back to `handleOnMissing` on failure (`filter.cc:393-398`). Increments `success` once per body.
- `handleOnPresent` (`filter.cc:286-340`): if the rule has a literal `value` use it; else fetch via `getValue`, then switch on `ValueType`:
  - `PROTOBUF_VALUE` — `JsonValueToProtobufValueConverter` (string values above `MAX_PAYLOAD_VALUE_LEN` = 8KiB are rejected, `filter.cc:54-57`).
  - `NUMBER` — `JsonValueToDoubleConverter` uses `absl::SimpleAtod` for string inputs.
  - `STRING` — `JsonValueToStringConverter`; oversize strings return `InvalidArgumentError`, empty strings skip without adding metadata.
- `addMetadata` (`filter.cc:208-231`) honours `preserve_existing_metadata_value`: if a value for the same `{namespace, key}` is already in dynamic metadata the new value is dropped.
- `finalizeDynamicMetadata` (`filter.cc:233-247`): pushes accumulated `StructMap` entries via `setDynamicMetadata` and, only on the request side, clears the route cache so metadata-matched routing sees the new values. It also marks the `processing_finished_flag`.
- Namespace default: `decideNamespace` (`filter.cc:204-206`) falls back to `HttpFilterNames::get().JsonToMetadata` when empty.
- `Rule` ctor validates that one of `on_present`/`on_missing` is set and that `on_missing`/`on_error` have non-empty values (`filter.cc:91-109`). Only `key` selectors are supported (array index selectors are ignored).

## Configuration
- `request_rules` / `response_rules` — each carries `rules[]`, `allow_content_types[]`, `allow_empty_content_type`, optional `allow_content_types_regex`.
- Rules use `selectors[].key` to address JSON fields; each rule has optional `on_present`, `on_missing`, `on_error`, each a `KeyValuePair` with `metadata_namespace`, `key`, `type`, `value`, `preserve_existing_metadata_value`.
- Per route: same `JsonToMetadata` message; per-route configs require at least one rule side (`filter.cc:146-150`).

## Stats
Two counter groups, emitted directly on the scope (`filter.cc:124-127`):
- `json_to_metadata.rq.{success, mismatched_content_type, no_body, invalid_json_body}`
- `json_to_metadata.resp.{success, mismatched_content_type, no_body, invalid_json_body}`
