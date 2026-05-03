# gRPC Field Extraction (`envoy.filters.http.grpc_field_extraction`)

Decoder-only filter that parses unary gRPC request bodies with a compiled protobuf descriptor set and extracts specified message fields (by field path) into dynamic metadata under the filter name, for downstream filters (RBAC, ext_authz, logging). Non-gRPC requests and unconfigured gRPC methods pass through untouched.

Proto: `envoy.extensions.filters.http.grpc_field_extraction.v3.GrpcFieldExtractionConfig`.

## Files
- `config.h/cc` — `FilterFactoryCreator` (`FactoryBase`) wiring the `FilterConfig` and `ExtractorFactoryImpl` dependencies. Provides both `createFilterFactoryFromProtoTyped` and `createFilterFactoryFromProtoWithServerContextTyped` so the filter can be built from route-level or server-level factory contexts.
- `filter_config.h/cc` — `FilterConfig`: loads a `Protobuf::DescriptorPool` from the configured descriptor source, builds a `TypeHelper` and a `TypeFinder`, then populates `proto_path_to_extractor_` (one `Extractor` per configured method).
- `filter.h/cc` — `Filter` (`PassThroughDecoderFilter`) implementing the decode-side flow and the per-stream `MessageConverter`.
- `extractor.h`, `extractor_impl.h/cc` — `Extractor` interface and implementation that walks the decoded message to pull out values.
- `message_converter/` — helpers that accumulate gRPC-framed bytes into complete `StreamMessage` units and restore them back to the wire buffer.

## Lifecycle
- `decodeHeaders` (filter.cc:80): if not `isGrpcRequestHeaders` returns `Continue` (pass-through). Converts `:path` `/pkg.svc/method` to `pkg.svc.method` via `grpcPathToProtoPath` (filter.cc:57); malformed paths are rejected with `BadRequest`/`InvalidArgument`. Looks up an `Extractor` for the proto path; if none, `Continue` (method not configured). Otherwise stores `extractor_`, constructs a per-stream `MessageConverter` with a `CordMessageData` factory bound to `decoder_callbacks_->bufferLimit()`, and returns `StopIteration` to buffer the body.
- `decodeData` (filter.cc:120): if no extractor or extraction already done, `Continue`. Otherwise calls `handleDecodeData(data, end_stream)`.
- `handleDecodeData` (filter.cc:134): feeds bytes into `request_msg_converter_->accumulateMessages(...)`. Buffering errors reject with `REQUEST_BUFFER_CONVERSION_FAIL`. If no complete message yet, returns `StopIterationNoBuffer`. For each complete `StreamMessage`, runs `extractor_->processRequest(*message_data->message())` once; on failure rejects with `REQUEST_FIELD_EXTRACTION_FAILED`; on success calls `handleExtractionResult` and then `convertBackToBuffer` to write the original framed bytes into `data` so the upstream request is byte-identical. If the stream ends without producing any message, rejects with `REQUEST_OUT_OF_DATA` / `InvalidArgument`.
- `handleExtractionResult` (filter.cc:211): builds a `Protobuf::Struct`; unset (`KIND_NOT_SET`) fields become empty `ListValue`s, others use the extracted `Value` directly. If any fields were produced, calls `decoder_callbacks_->streamInfo().setDynamicMetadata(kFilterName, dest_metadata)`.

Trailers and encode path use the `PassThroughDecoderFilter` defaults (not overridden).

## Decision / logic
- gRPC gate: non-gRPC requests skip the filter entirely (filter.cc:82).
- Per-method gate: `filter_config_->findExtractor(*proto_path)` controls whether the body is buffered at all (filter.cc:104).
- Extraction runs exactly once per stream (guarded by `extraction_done_`, filter.cc:175) even when multiple frames are buffered.
- Rejection helper `rejectRequest` (filter.cc:71) calls `sendLocalReply` with the mapped HTTP code from `Utility::grpcToHttpStatus`.

## Configuration
- `descriptor_set` / descriptor source on `FilterConfig`: loaded into `descriptor_pool_`.
- `extractions_by_method` (proto): keyed by fully qualified gRPC method; values are the field specs consumed by `ExtractorFactoryImpl`.
- No per-route override is wired; the filter uses only the listener-level config.

## Stats
- No counters or gauges are emitted directly by this filter. Rejection paths set `response_code_details` strings (filter.cc:40-48) of the form `grpc_field_extraction_<CODE>{<TYPE>}` (for access-log consumption):
  - `REQUEST_BUFFER_CONVERSION_FAIL`
  - `BAD_REQUEST`
  - `REQUEST_FIELD_EXTRACTION_FAILED`
  - `REQUEST_OUT_OF_DATA`
- Extracted fields are emitted as dynamic metadata under key `envoy.filters.http.grpc_field_extraction`.

## Factory
- `REGISTER_FACTORY(FilterFactoryCreator, NamedHttpFilterConfigFactory)` (config.cc:47). Name from `kFilterName = "envoy.filters.http.grpc_field_extraction"` (filter.h:20).
