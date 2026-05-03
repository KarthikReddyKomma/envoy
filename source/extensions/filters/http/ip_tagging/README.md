# IP Tagging (`envoy.filters.http.ip_tagging`)

A decoder-only filter that matches the downstream remote address against a set of CIDR ranges using a Level-Compressed Trie and writes the matching tag names into a header (default `x-envoy-ip-tags`). Per-tag hit counters and overall hit/miss/total counters are emitted. Can be scoped to internal-only, external-only, or both request types, and is gated by a runtime feature flag.

Proto: `envoy.extensions.filters.http.ip_tagging.v3.IPTagging`.

## Files
- `config.h/cc` — `IpTaggingFilterFactory` (`ExceptionFreeFactoryBase`) that constructs the shared `IpTaggingFilterConfig` and registers under both `envoy.filters.http.ip_tagging` and the legacy name `envoy.ip_tagging` (`config.cc:31-32`).
- `ip_tagging_filter.h/cc` — `IpTaggingFilterConfig` (trie + stat-name set + runtime ref), `FilterRequestType` enum, and `IpTaggingFilter` (`Http::StreamDecoderFilter`).

## Lifecycle
- `IpTaggingFilter` extends `Http::StreamDecoderFilter` (`ip_tagging_filter.h:100-120`); it is added via `addStreamDecoderFilter` (`config.cc:24`). Response-path callbacks do not exist.
- `decodeHeaders` (`ip_tagging_filter.cc:87-114`): determines if the request is internal by inspecting the `x-envoy-internal` header, then short-circuits based on `request_type` and runtime gate. On a match it queries the trie and calls `applyTags`.
- `decodeData` / `decodeTrailers` are pass-throughs (`ip_tagging_filter.cc:116-122`).
- `setDecoderFilterCallbacks` caches the callbacks for stream-info access (`ip_tagging_filter.cc:124-126`).
- `onDestroy` is a no-op.

## Decision / logic
- Short-circuit (`ip_tagging_filter.cc:92-96`): returns `Continue` immediately if (a) internal request with `request_type == EXTERNAL`, (b) external request with `request_type == INTERNAL`, or (c) runtime feature `ip_tagging.http_filter_enabled` is not satisfied (default 100%).
- Lookup (`ip_tagging_filter.cc:98-99`): `config_->trie().getData(downstreamAddressProvider().remoteAddress())`. Trie is built in `IpTaggingFilterConfig::IpTaggingFilterConfig` (`ip_tagging_filter.cc:45-74`) from `ip_tags[].ip_list[]` entries; invalid CIDRs fail config creation (`ip_tagging_filter.cc:62-67`).
- Stats path (`ip_tagging_filter.cc:102-112`): each matched tag charges `<tag>.hit` (unknown tags fall through to `unknown_tag.hit` via `getBuiltin` default, `ip_tagging_filter.h:54-55`); empty match increments `no_hit`; every processed request increments `total`.
- Header application (`applyTags`, `ip_tagging_filter.cc:128-166`):
  - If no tags matched: if `ip_tag_header` is set and action is `SANITIZE`, remove the configured header; if the remove changed the map call `clearRouteCache` so a stale route decision is invalidated (`ip_tagging_filter.cc:137-142`).
  - If tags matched and `ip_tag_header` is unset: `appendEnvoyIpTags` joins with `,` (backwards-compatible default `x-envoy-ip-tags`).
  - If tags matched and `ip_tag_header` is set: switch on action — `SANITIZE` uses `setCopy` (overwrite), `APPEND_IF_EXISTS_OR_ADD` uses `appendCopy`.
  - In both matched-cases the route cache is cleared so the new header can affect route matching (`ip_tagging_filter.cc:165`).
- Config requires at least one `ip_tag` entry (`ip_tagging_filter.cc:45-49`).

## Configuration
- `request_type` — enum `BOTH` / `INTERNAL` / `EXTERNAL` translated by `requestTypeEnum` (`ip_tagging_filter.h:64-76`).
- `ip_tags[]` — list of `{ip_tag_name, ip_list[]}`; compiled into a single `Network::LcTrie::LcTrie<std::string>` mapping CIDR -> tag name.
- `ip_tag_header` — optional `{header, action}`; action is `SANITIZE` (default) or `APPEND_IF_EXISTS_OR_ADD`.

## Stats
Stat-name set `IpTagging`, prefixed by `<stat_prefix>ip_tagging.` and emitted via `IpTaggingFilterConfig::incCounter` (`ip_tagging_filter.cc:76-79`):
- `<prefix>ip_tagging.<tag>.hit` — one per configured tag, pre-registered in the set.
- `<prefix>ip_tagging.unknown_tag.hit` — fallback for unregistered tag names.
- `<prefix>ip_tagging.no_hit` — request did not match any tag.
- `<prefix>ip_tagging.total` — every request the filter evaluates.
