# CSRF Filter (`envoy.filters.http.csrf`)

Rejects cross-origin state-changing requests (POST / PUT / DELETE / PATCH) by
comparing the request's source origin against the target origin.

Proto: `envoy.extensions.filters.http.csrf.v3.CsrfPolicy`.

## Lifecycle

Decode-only. `decodeHeaders()` (`csrf_filter.cc:98–126`):

1. Skip if method is not state-changing.
2. Skip if filter not enabled via `filter_enabled` runtime percentage.
3. Compute **source origin** from `Origin`, falling back to `Referer`
   (`csrf_filter.cc:53–63`).
4. Compute **target origin** by reconstructing `scheme://host` from the
   request headers (`csrf_filter.cc:65–76`).
5. `isValid()` (`csrf_filter.cc:138–151`) — allow if source == target or if
   source matches any `additional_origins` `StringMatcher`.
6. On invalid:
   - Enforced mode → 403 + `response_detail = "csrf_origin_mismatch"`.
   - Shadow mode only (enforced off, shadow on) → count the discrepancy and
     continue (lines 119–121).

## Configuration (`CsrfPolicy`)

- `filter_enabled` — `RuntimeFractionalPercent`; controls enforcement.
- `shadow_enabled` — `RuntimeFractionalPercent`; observation mode.
- `additional_origins` — extra `StringMatcher` entries treated as same-origin
  for this route / virtual host.

Policy resolution at `csrf_filter.cc:128–136` merges route and virtual-host
configs, route takes precedence.

## Stats

- `missing_source_origin` — neither `Origin` nor `Referer` present.
- `request_invalid` — origin mismatch, enforced.
- `request_valid` — request allowed.
- Shadow counterparts for shadow-only hits.

## Files

- `csrf_filter.{h,cc}` — filter implementation.
- `config.{h,cc}` — factory.
