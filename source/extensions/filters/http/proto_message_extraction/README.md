# Proto Message Extraction (`envoy.filters.http.proto_message_extraction`)

A bi-directional HTTP filter that extracts specified fields from gRPC request and response payloads and publishes them as dynamic metadata under the filter's namespace. It reframes gRPC-over-HTTP/2 DATA frames via the `grpc_field_extraction` `MessageConverter`, runs the configured per-method `Extractor` over the decoded `Message`, and records up to two metadata entries (`first` and `last` for streaming calls). Non-matching methods and non-gRPC traffic pass through untouched; the filter never rewrites the payload, only observes it.

Proto: `envoy.extensions.filters.http.proto_message_extraction.v3.ProtoMessageExtractionConfig`.

## Files
- `config.h/cc` — `FilterFactoryCreator` building a shared `FilterConfig` (parses descriptors, builds per-method `Extractor`s) and installing `Filter` per stream; `REGISTER_FACTORY` at `config.cc:46`.
- `filter.h/cc` — `Filter` extending `PassThroughFilter` with handcrafted decode/encode data loops.
- `filter_config.h/cc` — `FilterConfig` that owns the `DescriptorPool`, `TypeHelper`, and `proto_path -> Extractor` map (`filter_config.h:25-46`).
- `extractor.h`, `extractor_impl.h/cc` — `Extractor` interface and factory plus the proto-field-extraction-backed implementation.
- `extraction_util/` — shared helpers used by the extractor.

## Lifecycle
Installed as a full stream filter (`config.cc:30`, `config.cc:42`: `callbacks.addStreamFilter(...)`).

Overridden callbacks:
- `decodeHeaders` (`filter.cc:109-161`) — gRPC detection, path-to-proto-path, lookup `Extractor`, prime request converter.
- `decodeData` (`filter.cc:163-176`) — delegates to `handleDecodeData`.
- `encodeHeaders` (`filter.cc:252-278`) — gRPC response detection, prime response converter.
- `encodeData` (`filter.cc:280-293`) — delegates to `handleEncodeData`.

No header/trailer writes, no local replies outside error paths.

## Decision / logic
`decodeHeaders`:
- `filter.cc:114-122`: non-gRPC -> `Continue`, filter is effectively disabled for the stream (`extractor_` stays null).
- `filter.cc:126-137`: `grpcPathToProtoPath(:path)` converts `/pkg.Svc/Method` to `pkg.Svc.Method`; malformed paths `rejectRequest` with `BAD_REQUEST` rc_detail.
- `filter.cc:139-145`: `filter_config_.findExtractor(proto_path)`; if no extractor is configured for this method, pass through (the filter intentionally no-ops).
- `filter.cc:148-152`: cache the `Extractor*` and `ClearResult()` for reuse across calls.
- `filter.cc:154-158`: construct `request_msg_converter_` with `decoder_callbacks_->bufferLimit()`.

`handleDecodeData` (`filter.cc:178-250`):
- `filter.cc:183-191`: `accumulateMessages` error -> `rejectRequest` with `REQUEST_BUFFER_CONVERSION_FAIL`.
- `filter.cc:193-197`: incomplete frame -> `StopIterationNoBuffer`.
- `filter.cc:200-217`: iterate accumulated `StreamMessage`s; skip the terminal empty one (asserts on `isFinalMessage`).
- `filter.cc:218-221`: set `request_extraction_done_ = true` and call `extractor_->processRequest(message)`.
- `filter.cc:223-228`: pull `GetResult()`; if `request_data` is empty, return `StopIterationNoBuffer` (wait for more data). Otherwise `handleRequestExtractionResult` writes metadata.
- `filter.cc:230-236`: `convertBackToBuffer` is expected to succeed (`RELEASE_ASSERT`) since payload isn't modified, and the bytes are moved back into `data`.
- `filter.cc:241-248`: if we never produced a message, reject with `REQUEST_OUT_OF_DATA`.

`encodeHeaders` / `handleEncodeData` mirror the decode path with `response_*` names (`filter.cc:252-363`). On success each branch recycles the buffer and eventually returns `Continue`. Failures call `rejectResponse` (`filter.cc:96-107`) which additionally sets `CoreResponseFlag::UnauthorizedExternalService`.

`handleRequestExtractionResult` / `handleResponseExtractionResult` (`filter.cc:365-436`):
- Build a `Protobuf::Struct` with top-level keys `"requests"` / `"responses"`, each containing `"first"` and optionally `"last"` (for streaming RPCs `result.size() == 2`).
- For responses, additionally emit `numResponseItems` when the extractor reports a count (`filter.cc:417-420`).
- Write via `streamInfo().setDynamicMetadata(kFilterName, dest_metadata)` (`filter.cc:395`, `filter.cc:434`).

`rejectRequest` / `rejectResponse` use `sendLocalReply` with the mapped HTTP status and an rc_detail of the form `proto_message_extraction_<statusCode>{<errorType>}`.

## Configuration
`ProtoMessageExtractionConfig`:
- Descriptor source (inline or file) loaded by `FilterConfig::initDescriptorPool` (`filter_config.h:34`).
- Per-method extraction directives turned into `Extractor` instances by `ExtractorFactoryImpl` and indexed in `proto_path_to_extractor_` (`filter_config.h:41`).

No per-route config: the factory only implements `createFilterFactoryFromProtoTyped` (and the with-server-context variant). Both build the same `FilterConfig` using `ExtractorFactoryImpl` and `context.api()` (`config.cc:23-44`).

## Stats
None. Extraction outcomes surface as dynamic metadata (`kFilterName`), not counters. Debug logs at `trace`/`debug` levels describe the processed messages.

## Factory
`FilterFactoryCreator` (`config.h:18`):
- `FactoryBase<ProtoMessageExtractionConfig>` registered under `kFilterName = "envoy.filters.http.proto_message_extraction"` (`filter.h:24`).
- `createFilterFactoryFromProtoTyped` and `createFilterFactoryFromProtoWithServerContextTyped` both instantiate `FilterConfig(proto_config, ExtractorFactoryImpl, api)` and add a `Filter` per stream (`config.cc:23-44`).
- `REGISTER_FACTORY(FilterFactoryCreator, NamedHttpFilterConfigFactory)` at `config.cc:46`.
