# A2A Filter (`envoy.filters.http.a2a`)

HTTP decoder filter that validates traffic against the Agent-to-Agent (A2A) protocol. It accepts GET requests (used for agent discovery) and POST requests whose body is a JSON-RPC 2.0 envelope. When configured in `REJECT` traffic mode, non-A2A traffic is answered with `400 Bad Request`; in `PASS_THROUGH` mode it is forwarded. On valid requests it can emit selected JSON-RPC fields (method, id, params) into dynamic metadata for downstream consumers.

Proto: `envoy.extensions.filters.http.a2a.v3.A2a`.

## Files
- `a2a_filter.h/cc` — filter class, config wrapper, stats struct.
- `a2a_json_parser.h/cc` — streaming JSON-RPC parser `A2aJsonParser` built on top of `Json::JsonRpcFieldExtractor`; extracts configured fields with early-stop.
- `config.h/cc` — `A2aFilterConfigFactory`, registered as `envoy.filters.http.a2a`.

## Lifecycle
The filter extends `Http::PassThroughFilter` (`a2a_filter.h:66`) so only decode-side methods are overridden.

- `decodeHeaders(headers, end_stream)` — classifies the request (`a2a_filter.cc:63`).
  - Valid A2A GET + `end_stream` -> mark `is_a2a_request_=true`, `Continue` (`a2a_filter.cc:68`).
  - Valid A2A POST with body -> `StopIteration` and raises the decoder buffer limit to `max_request_body_size` (default 8192) so the JSON body is buffered for inspection (`a2a_filter.cc:74-88`).
  - Anything else in `REJECT` mode -> `sendLocalReply(400, "Only A2A traffic is allowed")` and bump `requests_rejected` (`a2a_filter.cc:91-97`).
- `decodeData(data, end_stream)` — lazily constructs `A2aJsonParser` and feeds slices through it (`a2a_filter.cc:103`).
  - Skipped when `is_a2a_request_` is false or parsing already finished (`a2a_filter.cc:104-111`).
  - Iterates `data.getRawSlices()`; clamps length to `max_request_body_size - bytes_parsed_` (`a2a_filter.cc:122-128`).
  - On `isAllFieldsCollected()` early-exits via `completeParsing()` (`a2a_filter.cc:134-137`).
  - On parser error returns `400 not a valid JSON` and bumps `invalid_json` (`a2a_filter.cc:139-144`).
  - On `end_stream` or size-limit-hit it calls `parser_->finishParse()`; body-too-large and incomplete-JSON branch to `body_too_large` vs `invalid_json` stats (`a2a_filter.cc:152-165`).
  - Otherwise returns `StopIterationAndWatermark` to keep buffering (`a2a_filter.cc:169`).

## Decision / logic
- Content-Type gate for POST allows `application/json` and `application/a2a+json`, using a prefix check that also accepts `;` or whitespace delimiters (`a2a_filter.cc:44-55`).
- GET with a body has no A2A semantics; it is explicitly rejected in `REJECT` mode and passed through otherwise (`a2a_filter.cc:65-72`).
- `shouldRejectRequest()` maps to `TrafficMode::REJECT` (`a2a_filter.cc:57-59`).
- `completeParsing()` calls `parser_->isValidA2aRequest()` (JSON-RPC 2.0 compliance) and, if still invalid in `REJECT` mode, sends `400 request must be a valid JSON-RPC 2.0 message for A2A` (`a2a_filter.cc:179-190`).
- On success, if the parser collected any fields, the filter writes them to stream dynamic metadata under its filter config name (`a2a_filter.cc:192-200`).

## Configuration
- `traffic_mode` (enum): `REJECT` or `PASS_THROUGH` — controls behavior when the request is not A2A.
- `max_request_body_size` (uint32, default 8192): soft body cap enforced via `setDecoderBufferLimit` and tracked by `bytes_parsed_`; `0` disables the cap.
- `parser_config_` is `A2aParserConfig::createDefault()` (`a2a_filter.cc:26`) — selects which JSON-RPC fields to extract (see `a2a_json_parser.h:55`).
- No per-route config is defined.

## Stats
Prefix `<stats_prefix>a2a.` (`a2a_filter.cc:15-18`):

- `requests_rejected` — non-A2A request rejected in `REJECT` mode.
- `invalid_json` — parser rejected the body (invalid or incomplete JSON/JSON-RPC).
- `body_too_large` — parsing exhausted `max_request_body_size` before all required fields were seen.

## Factory
`A2aFilterConfigFactory::createFilterFactoryFromProtoTyped` builds a shared `A2aFilterConfig` against the server root scope and registers one `StreamFilter` per stream via `addStreamFilter` (`config.cc:12-23`). Registration via `REGISTER_FACTORY` under `envoy.filters.http.a2a` (`config.cc:28`).
