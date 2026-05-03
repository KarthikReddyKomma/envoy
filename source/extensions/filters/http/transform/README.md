# Transform (`envoy.filters.http.transform`)

Transforms JSON request and/or response bodies and mutates headers using Envoy's substitution-formatter machinery. Bodies are parsed to `Protobuf::Struct`, exposed through a `BodyContextExtension` so format strings can reference request/response body fields via `%REQUEST_BODY(a:b:c)%` / `%RESPONSE_BODY(...)%`, and either fully replaced or merged back with the parsed payload. Works both as a listener filter and as a per-route override.

Proto: `envoy.extensions.filters.http.transform.v3.TransformConfig`.

## Files
- `config.h/cc` — `TransformFactoryConfig` (`ExceptionFreeFactoryBase`) registered as `envoy.filters.http.transform`; also wires `createRouteSpecificFilterConfigTyped`.
- `transform.h/cc` — `BodyContextExtension`, `BodyFormatterProvider`, `BodyFormatterCommandParser`, `Transformation`, `TransformConfig`, `FilterConfig`, and `TransformFilter`.

## Lifecycle
- Registered at `config.cc:35` (`REGISTER_FACTORY`). Listener path creates `FilterConfig` (which extends `TransformConfig` and owns stats) in `createFilterFactoryFromProtoTyped` (`config.cc:12-23`) and installs the filter via `addStreamFilter`. Per-route path creates a bare `TransformConfig` (`config.cc:25-33`).
- `Transformation` ctor (`transform.cc:83-110`): builds a body `Formatter` from `body_format` (wiring in the `BodyFormatterCommandParser` for `REQUEST_BODY` / `RESPONSE_BODY`), captures `content_type`, sets `merge_format_string_` based on `BodyTransformation::MERGE`, and compiles `headers_mutations_` via `Http::HeaderMutations::create`.
- `TransformConfig` ctor (`transform.cc:112-128`): materializes request/response `Transformation`s, stores `clear_route_cache_` / `clear_cluster_cache_`, and rejects configs that set both cache-clear flags.
- Decode path (`transform.cc:151-191`):
  - `decodeHeaders` — skips on headers-only or non-JSON content types (`transform.cc:154-156`); calls `maybeInitializeRouteConfigs` to select `effective_config_`; if no request transform, continue; else sets `decoding_enabled_` and returns `StopIteration`.
  - `decodeData` — returns `Continue` when disabled; on `end_stream` adds the final chunk to the decoding buffer and calls `handleCompleteRequestBody`; otherwise `StopIterationAndBuffer`.
  - `decodeTrailers` — triggers `handleCompleteRequestBody` when enabled.
- Encode path (`transform.cc:193-246`) mirrors the decode path. `onLocalReply` sets `saw_local_reply_` (`transform.h:185-188`) so the encode path skips transformation on local replies.

## Decision / logic
- Route config resolution: `maybeInitializeRouteConfigs` (`transform.cc:130-149`) caches the first resolved `TransformConfig` (per stream) so both directions use the same snapshot. Falls back to the filter-level config when none is found.
- Content-type gating: `transform.cc:154` and `transform.cc:201` — requires `Content-Type` to contain `application/json` substring.
- Local-reply suppression: checked at the start of `encodeHeaders/Data/Trailers` (`transform.cc:195, 219, 236`).
- `handleCompleteBody` (`transform.cc:248-298`):
  - Parses the buffered body via `MessageUtil::loadFromJsonNoThrow`. If it isn't valid JSON, returns an empty result and body/headers are left untouched (`transform.cc:253-257`).
  - Runs the body formatter to produce `new_buffer`. In MERGE mode the formatter output must itself be valid JSON (`transform.cc:269-272`).
  - Applies header mutations after the body formatting but before the final merge (`transform.cc:280-284`).
  - In MERGE mode: `MergeFrom` the newly parsed struct into the original struct, then serialize back to JSON (`transform.cc:287-295`). Non-merge mode replaces the body with the formatter output verbatim.
- `handleCompleteRequestBody` (`transform.cc:300-345`): uses `decodingBuffer()` — if empty the filter no-ops (TODO: support body creation). On `transform_buffer`, strips `Content-Length` and sets `Content-Type` from the transformation if configured, then `modifyDecodingBuffer` swaps in the new payload. On `transform_header`, calls `refreshRouteCluster()` or `clearRouteCache()` depending on which cache-clear flag was set.
- `handleCompleteResponseBody` (`transform.cc:347-380`): symmetric for the encode side but does not touch route/cluster caches.
- `BodyFormatterProvider` (`transform.cc:24-52`) walks the struct via `Config::Metadata::structValue(body, path_)`; strings are returned raw, other kinds are JSON-serialized for string-formatted output, while `formatValue` returns the raw `Protobuf::Value`.

## Configuration
- `request_transformation` / `response_transformation` — each has `body_transformation.body_format` (substitution format string with `content_type`, `action = MERGE|REPLACE`) and `headers_mutations`.
- `clear_route_cache` / `clear_cluster_cache` — mutually exclusive (`transform.cc:124-127`); applied only on the request path after header mutations succeed.
- Per-route: same proto (`TransformConfig`) applied via `createRouteSpecificFilterConfigTyped`. `Http::Utility::resolveMostSpecificPerFilterConfig<TransformConfig>` picks the most specific entry.

## Stats
Prefix `<stats_prefix>.http_transform.` (`transform.h:157`). Counters (`transform.h:40-42`):
- `rq_transformed` — request direction: body and/or headers mutated.
- `rs_transformed` — response direction: body and/or headers mutated.
