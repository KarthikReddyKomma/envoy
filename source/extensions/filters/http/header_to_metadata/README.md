# Header to Metadata (`envoy.filters.http.header_to_metadata`)

A stream filter that extracts values from HTTP request / response headers or request cookies and writes them to the stream's dynamic metadata. Rules support type coercion (string / number / protobuf Value), optional base64 decoding, optional regex rewrite of the value, and separate `on_header_present` / `on_header_missing` actions. Per-route configs can override the top-level config.

Proto: `envoy.extensions.filters.http.header_to_metadata.v3.Config`.

## Files
- `config.h/cc` — `HeaderToMetadataConfig` factory (`ExceptionFreeFactoryBase`) that builds the shared `Config` from proto and registers `envoy.filters.http.header_to_metadata` (`config.cc:44`). Produces the per-route config (`config.cc:31-39`).
- `header_to_metadata_filter.h/cc` — value selectors (`HeaderValueSelector`, `CookieValueSelector`), `Rule`, `Config` (holds request / response `Rule` vectors, optional stats), and `HeaderToMetadataFilter` (`Http::StreamFilter`).

## Lifecycle
- `HeaderToMetadataFilter` implements the full `Http::StreamFilter` interface but only `decodeHeaders` / `encodeHeaders` do work; other callbacks return `Continue` (`header_to_metadata_filter.h:208-231`).
- `decodeHeaders` (`header_to_metadata_filter.cc:215-224`): resolves the effective config (global vs. per-route) and, if request rules exist, invokes `writeHeaderToMetadata` with `HeaderDirection::Request`.
- `encodeHeaders` (`header_to_metadata_filter.cc:231-239`): mirrors decode for response rules with `HeaderDirection::Response`.
- `setDecoderFilterCallbacks` / `setEncoderFilterCallbacks` stash the callback pointers used for metadata writes.

## Decision / logic
- `getConfig` (`header_to_metadata_filter.cc:368-381`): caches `effective_config_`; first checks `Http::Utility::resolveMostSpecificPerFilterConfig<Config>` on the decoder callbacks, falls back to the filter's global `Config`. The same cached pointer is reused for both decode and encode paths.
- `Rule::Rule` validates (`header_to_metadata_filter.cc:51-109`): exactly one of `header` / `cookie`; at least one of `on_header_present` / `on_header_missing`; mutual exclusivity of `value` and `regex_value_rewrite`; `remove` is illegal for cookie rules; `on_header_missing` cannot have an empty value. Regex rewrites are compiled via `Matcher::RegexReplace::create`.
- `HeaderValueSelector::extract` (`header_to_metadata_filter.cc:22-33`) collects the header via `Http::HeaderUtility::getAllOfHeaderAsString`; if `remove` is set it strips the header after capture. `CookieValueSelector::extract` (`header_to_metadata_filter.cc:36-42`) pulls the named cookie value.
- `writeHeaderToMetadata` (`header_to_metadata_filter.cc:335-365`): iterates rules, charges `RulesProcessed`, and dispatches to `applyKeyValue` via `on_header_present` when extraction succeeded or `on_header_missing` otherwise (also charging `HeaderNotFound`). Accumulated namespaces are flushed as dynamic metadata via `callbacks.streamInfo().setDynamicMetadata` (`header_to_metadata_filter.cc:361-364`).
- `applyKeyValue` (`header_to_metadata_filter.cc:309-333`): if the rule's `KeyValuePair.value` is set that literal is used; otherwise, when a `regex_value_rewrite` is present it is applied to the extracted value (empty result from non-empty input charges `RegexSubstitutionFailed`). Empty final values are skipped.
- `addMetadata` (`header_to_metadata_filter.cc:246-302`): rejects values >= `MAX_HEADER_VALUE_LEN` (8KiB, `header_to_metadata_filter.h:127`) with `HeaderValueTooLong`; base64-decodes when `encode == BASE64` and charges `Base64DecodeFailed` on empty result; converts based on `ValueType` — `STRING` set directly, `NUMBER` via `absl::SimpleAtod` on trimmed input, `PROTOBUF_VALUE` via `Protobuf::Value::ParseFromString`. On success charges `MetadataAdded`.
- Namespace resolution: `decideNamespace` (`header_to_metadata_filter.cc:304-306`) defaults empty namespaces to `HttpFilterNames::get().HeaderToMetadata`.

## Configuration
- `request_rules` / `response_rules` — repeated `Rule`; `request_set_` / `response_set_` are derived in `Config::configToVector` (`header_to_metadata_filter.cc:149-164`) and gate whether the filter does anything.
- `stat_prefix` — when non-empty the `Config` ctor emplaces stats (`header_to_metadata_filter.cc:134-136`); when empty the filter is stat-less (opt-in).
- Per route: same `Config` message; requires at least one of `request_rules` / `response_rules` (`header_to_metadata_filter.cc:142-146`).

## Stats
Emitted under `http_filter_name.<stat_prefix>` (`header_to_metadata_filter.cc:166-170`). Counters (`header_to_metadata_filter.h:24-33`):
- `request_rules_processed`, `response_rules_processed`
- `request_metadata_added`, `response_metadata_added`
- `request_header_not_found`, `response_header_not_found`
- `base64_decode_failed`
- `header_value_too_long`
- `regex_substitution_failed`
