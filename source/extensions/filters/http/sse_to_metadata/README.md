# SSE To Metadata (`envoy.filters.http.sse_to_metadata`)

Encoder-side filter that parses Server-Sent Events (SSE) out of HTTP response bodies and writes extracted values into dynamic stream metadata. Extraction is delegated to a pluggable `ContentParser` factory (configured via `TypedExtensionConfig`) so the same SSE-framing logic can drive different value-extraction schemes (JSON path, regex, etc.).

Proto: `envoy.extensions.filters.http.sse_to_metadata.v3.SseToMetadata`.

## Files
- `config.h/cc` — `SseToMetadataConfig` factory derived from `ExceptionFreeFactoryBase`; registers the name `envoy.filters.http.sse_to_metadata` and builds a `FilterConfig` shared instance per listener.
- `filter.h/cc` — `FilterConfig` (owns the parser factory, stats, and `max_event_size`) and `Filter` (the per-stream `PassThroughEncoderFilter`).

## Lifecycle
- Registered at `config.cc:28` (`REGISTER_FACTORY`). `createFilterFactoryFromProtoTyped` (`config.cc:12-23`) builds one `FilterConfig` then returns a cb that installs a new `Filter` via `addStreamEncoderFilter` — the filter only participates in the encode path.
- `FilterConfig` ctor (`filter.cc:33-49`) resolves the `content_parser` `TypedExtensionConfig`, builds a `ContentParser::ParserFactory`, generates stats under `sse_to_metadata.resp.<parser-prefix>`, and stores `max_event_size` (default 8192).
- `Filter` ctor (`filter.cc:51-52`) creates a per-stream `ContentParser::Parser` from the factory.
- `encodeHeaders` (`filter.cc:54-68`) inspects `Content-Type`; if it equals `text/event-stream` (parameter-stripped, case-insensitive via `isSseContentType`, `filter.cc:26-29`) sets `content_type_matched_ = true`, otherwise bumps `mismatched_content_type_` and short-circuits all downstream work.
- `encodeData` (`filter.cc:70-86`) appends to `buffer_`, invokes `processBuffer` to chew complete events, and calls `finalizeRules()` at `end_stream` or when processing has completed early. Always returns `Continue` so data is not held back.
- `encodeTrailers` (`filter.cc:88-94`) calls `finalizeRules()` if the Content-Type matched and finalization hasn't run yet.

## Decision / logic
- Content-type gate: `filter.cc:56` — mismatch increments `mismatched_content_type_` and prevents body parsing.
- Event framing loop: `processBuffer` (`filter.cc:96-133`) linearizes the buffer (`filter.cc:99`), calls `Http::Sse::SseParser::findEventEnd` (`filter.cc:102`). If no complete event and the buffer size exceeds `max_event_size` (`filter.cc:106-114`), the filter logs a warning, bumps `event_too_large_`, and drains the whole buffer to recover.
- Per-event parsing: `processSseEvent` (`filter.cc:135-160`) calls `SseParser::parseEvent`. Missing `data` field bumps `no_data_field_` (`filter.cc:140`). Parser errors bump `parse_error_` (`filter.cc:148`). Successful parses iterate `result.immediate_actions` and call `writeMetadata`. Returning `result.stop_processing` ends the loop.
- Deferred actions: `finalizeRules` (`filter.cc:162-178`) drains `parser_->getAllDeferredActions()` (fallbacks for unmatched rules) and increments both `metadata_added_` and `metadata_from_fallback_` for each success.
- `writeMetadata` (`filter.cc:180-223`): respects `preserve_existing` (bumps `preserved_existing_metadata_` and returns false if key exists, `filter.cc:186-199`); validates that a Protobuf value is set (`filter.cc:202-215`); writes via `encoder_callbacks_->streamInfo().setDynamicMetadata(namespace, struct)` (`filter.cc:219`).

## Configuration
- `response_rules.content_parser` — required `TypedExtensionConfig` selecting the `ContentParser::NamedContentParserConfigFactory`.
- `response_rules.max_event_size` — uint wrapper, default 8192 bytes (`filter.cc:48-49`). `0` disables the cap.
- No per-route override surface; configuration is listener-level only.

## Stats
Prefix `sse_to_metadata.resp.<parser_stats_prefix>.` (`filter.cc:46`). Counters (`filter.h:29-36`):
- `metadata_added` — metadata writes that succeeded.
- `metadata_from_fallback` — subset of above, sourced from deferred `on_error`/`on_missing` actions.
- `mismatched_content_type` — response was not `text/event-stream`.
- `no_data_field` — parsed SSE event lacked a `data:` field.
- `parse_error` — content parser reported an error.
- `preserved_existing_metadata` — `preserve_existing` blocked an overwrite.
- `event_too_large` — buffered bytes exceeded `max_event_size` with no terminator; buffer was discarded.
