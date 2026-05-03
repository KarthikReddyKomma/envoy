# CDN-Loop Filter (`envoy.filters.http.cdn_loop`)

Implements the `CDN-Loop` request header defined by RFC 8586. For each request the filter counts how many times the configured `cdn_id` already appears in the incoming header; if the count exceeds the configured ceiling the request is rejected with 502 to break the loop. Otherwise the filter appends its own `cdn_id` so the next hop can detect it.

Proto: `envoy.extensions.filters.http.cdn_loop.v3.CdnLoopConfig`.

## Files
- `filter.h/cc` — `CdnLoopFilter`, a decoder-only filter.
- `parser.h/cc` — recursive-descent parser for the CDN-Loop grammar (quoted-strings, cdn-info list, lax IPv6, etc.).
- `utils.h/cc` — `countCdnLoopOccurrences(header, cdn_id)` built on top of the parser.
- `config.h/cc` — `CdnLoopFilterFactory` plus `cdn_id` validation at config time.

## Lifecycle
`CdnLoopFilter` extends `Http::PassThroughDecoderFilter` (`filter.h:15`). Only `decodeHeaders` is overridden; every other callback uses pass-through defaults. The filter never touches response headers/body.

Overridden method:
- `decodeHeaders(RequestHeaderMap&, bool)` (`filter.cc:28`): does the count-and-append. Returns either `StopIteration` with a local reply, or `Continue`.

## Decision / logic
Inside `decodeHeaders`:
1. Resolve the request's `CDN-Loop` entry via a registered inline header handle (`filter.cc:18-19`, `filter.cc:31`) — makes header lookup O(1).
2. If the header is present, call `countCdnLoopOccurrences(header_value, cdn_id_)` (`filter.cc:33`, defined in `utils.cc`).
   - Parse failure ⇒ `sendLocalReply(BadRequest, "Invalid CDN-Loop header in request.", ..., "invalid_cdn_loop_header")` and return `StopIteration` (`filter.cc:36-38`).
   - `count > max_allowed_occurrences_` ⇒ `sendLocalReply(BadGateway, "The server has detected a loop between CDNs.", ..., "cdn_loop_detected")` and return `StopIteration` (`filter.cc:39-42`).
3. Append the local `cdn_id` to `CDN-Loop` via `appendCopy` (`filter.cc:46`) and return `Continue`. Appending runs even when the header was previously absent — that's how the filter advertises itself downstream.

Parsing rules (see `parser.h` comments):
- `cdn-id = (plausible-ipv6 / token) [ ":" *DIGIT ]` — lax IPv6 (`parser.h:210`) and RFC 7230 `token` (`parser.h:195`).
- `cdn-info = cdn-id *( OWS ";" OWS parameter )` — parameter values may be quoted strings.
- `CDN-Loop = #cdn-info` with empty-element tolerance per RFC 7230 §7.

## Configuration
`CdnLoopFilterFactory::createFilterFactoryFromProtoTyped` (`config.cc:24`) validates `cdn_id` by parsing it through `Parser::parseCdnId` and requiring that the parse consumes the entire input (`config.cc:27-32`). On failure it returns `absl::InvalidArgumentError` with the message returned by the parser. No per-route config.

Proto fields consumed:
- `cdn_id` — the CDN identifier to count and inject. Required to be a valid `cdn-id` token/host per RFC.
- `max_allowed_occurrences` — ceiling before the loop is declared. A typical value of `0` makes any prior occurrence trigger the loop-detected response (note: this applies to the count seen *before* this filter appends its own id, so `0` means "we've already been here once").

## Stats
None. This filter does not declare its own counters or histograms.

## Factory
`CdnLoopFilterFactory` extends `Common::ExceptionFreeFactoryBase` (`config.h:17`). The callback does `callbacks.addStreamDecoderFilter(std::make_shared<CdnLoopFilter>(config.cdn_id(), config.max_allowed_occurrences()))` (`config.cc:33-36`). Registered via `REGISTER_FACTORY(CdnLoopFilterFactory, NamedHttpFilterConfigFactory)` at `config.cc:39`.
