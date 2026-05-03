# Thrift To Metadata (`envoy.filters.http.thrift_to_metadata`)

Parses Thrift-over-HTTP request and/or response payloads (bodies as Thrift messages) and writes extracted values to dynamic metadata. Supports extraction of message-level properties (method name, protocol, transport, header flags, sequence id, message type, reply type) as well as field-level payload extraction via a `PayloadExtractor` trie.

Proto: `envoy.extensions.filters.http.thrift_to_metadata.v3.ThriftToMetadata`.

## Files
- `config.h/cc` — `ThriftToMetadataConfig` factory (implements both `FactoryContext` and `ServerFactoryContext` variants).
- `filter.h/cc` — `Rule`, `ThriftDecoderHandler`, `FilterConfig`, and the `Filter` class (a `PassThroughFilter` + `PayloadExtractor::MetadataHandler`).

## Lifecycle
- Registered at `config.cc:42` (`REGISTER_FACTORY`). Both factory entry points (`config.cc:17-37`) build a `FilterConfig` and install the filter via `addStreamFilter`.
- `FilterConfig` ctor (`filter.cc:118-141`): creates the request/response stats groups under prefixes `thrift_to_metadata.rq` / `thrift_to_metadata.resp`, builds request/response `Rule` vectors and their tries, resolves transport/protocol (rejects `TWITTER`), and the allowed content-type set (defaults to `application/x-thrift`).
- `Rule` ctor (`filter.cc:14-31`): enforces that each rule has `on_present` or `on_missing` (throws otherwise); if a `field_selector` is configured the rule is inserted into the trie keyed by `rule_id`, else a synchronous `protobuf_value_extracter_` is built by `getValueExtractorFromField` (`filter.cc:46-116`).
- Decode path (`filter.cc:151-211`):
  - `decodeHeaders` (`filter.cc:151-171`) — bails to `Continue` if no request rules; on mismatched content type bumps `mismatched_content_type_` and marks processing done; on `end_stream` runs `handleAllOnMissing` (bumps `no_body_`); otherwise initializes `rq_trie_handler_` and `rq_handler_` and returns `StopIteration`.
  - `decodeData` (`filter.cc:173-198`) — buffers and calls `processData`. Decoder exceptions bump `invalid_thrift_body_` and invoke `handleAllOnMissing`. While still parsing returns `StopIterationAndBuffer`.
  - `decodeComplete` (`filter.cc:200-211`) — if the stream ended without a full Thrift message, treats all rules as missing and bumps `invalid_thrift_body_`.
- Encode path (`filter.cc:213-277`) mirrors the decode path with response rules/stats/handlers.

## Decision / logic
- Content-type gate (`filter.cc:156, 218`, helper at `filter.cc:143-149`): empty allowed only if `allow_empty_content_type`; otherwise set membership check.
- Decoder error handling (`filter.cc:279-299`): `processData` catches `AppException` (logs as error, does not count) and generic `EnvoyException` (stream-scoped log). Returning false triggers `handleAllOnMissing`.
- `handleThriftMetadata` (`filter.cc:302-319`) — called on each Thrift `messageBegin`. Runs `processMetadata`, which evaluates simple (non field-selector) rules inline (`filter.cc:382-397`), and tracks matched field-selector rule IDs. If any field-selector rule is still pending it returns `FilterStatus::Continue` to keep decoding the payload; otherwise it returns `StopIteration` (avoids buffering the rest of the message).
- `handleOnPresent` (`filter.cc:322-353`) — per trie hit, erases the rule from the pending set, converts the variant into a `Protobuf::Value`, and stages `on_present` via `applyKeyValue`. Empty strings are skipped (`filter.cc:337-339`).
- `handleComplete` (`filter.cc:356-376`) — for any field-selector rule still unmatched, runs `on_missing`; then finalizes per-direction dynamic metadata and bumps `success_`.
- Method-name matching: `Rule::matches` (`filter.cc:33-44`) supports both plain names and service-qualified names (`service:method`).
- Namespace default: `decideNamespace` (`filter.cc:457-459`) falls back to `HttpFilterNames::get().ThriftToMetadata` when the rule leaves `metadata_namespace` empty.

## Configuration
- `request_rules` / `response_rules` — at least one must be non-empty (`filter.cc:133-136`). Each rule picks a message-level `field`, or a nested `field_selector` for payload field extraction, plus `on_present` / `on_missing` key-value pairs (`metadata_namespace`, `key`, `value`).
- `transport` — Thrift transport (`FRAMED`, `UNFRAMED`, `HEADER`, `AUTO`); rejected if `TWITTER`.
- `protocol` — Thrift protocol (`BINARY`, `COMPACT`, `AUTO`).
- `allow_content_types` — defaults to `application/x-thrift` when empty.
- `allow_empty_content_type` — whether empty `Content-Type` passes the gate.
- No per-route config.

## Stats
Two groups, prefixes `thrift_to_metadata.rq.` and `thrift_to_metadata.resp.` (`filter.cc:121-124`). Counters (`filter.h:36-40`):
- `success` — per-direction finalized metadata write.
- `mismatched_content_type` — header gate rejected the message.
- `no_body` — headers-only request/response with rules configured.
- `invalid_thrift_body` — Thrift decoder failed or stream ended mid-message.
