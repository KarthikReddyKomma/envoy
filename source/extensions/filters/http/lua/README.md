# Lua Filter (`envoy.filters.http.lua`)

Runs user-supplied Lua scripts on the request and response path. Each request
gets its own coroutine that can mutate headers/body/trailers, make HTTP calls
to other clusters, record metadata, or short-circuit with a response.

Proto: `envoy.extensions.filters.http.lua.v3.Lua`.

## Script entry points

- `envoy_on_request(handle)` — runs in the decode path.
- `envoy_on_response(handle)` — runs in the encode path.

`FilterConfig` stores one or more named code bundles (`default_lua_code_setup_`
or named setups). `FilterConfigPerRoute` picks a named bundle, supplies inline
code, or disables the filter for that route
(`lua_filter.h:487–501`).

## Coroutine states

`StreamHandleWrapper` (`lua_filter.h:148–430`) is the coroutine's view of the
stream. It can yield in the states: `Running`, `WaitForBodyChunk`,
`WaitForBody`, `WaitForTrailers`, `HttpCall`, `Responded`. The filter resumes
the coroutine whenever the awaited event arrives; yields are what give the
filter its streaming capability.

## Lua API surface (selected)

- **Stream inspection** — `headers()`, `body()`, `bodyChunks()`, `trailers()`,
  `metadata()`, `streamInfo()`, `connection()`.
- **Mutation** — header / body mutators, `setUpstreamOverrideHost()`,
  `clearRouteCache()`.
- **Outbound** — `httpCall(cluster, headers, body, timeout, async)`:
  async or sync request to another configured cluster.
- **Short-circuit** — `respond(headers, body)`.
- **Utilities** — crypto (public key import, signature verification),
  `base64Escape()`, timestamps, stats counters.

## Lifecycle & memory

A fresh coroutine is created per stream and reset on stream destruction to
break circular references. `runtimeBytesUsed()` / `runtimeGC()` expose the Lua
runtime for tuning.

## Configuration

- `default_source_code` — the default script (backwards compatible with
  `inline_code`).
- `source_codes` — named scripts; routes pick by name.
- `per_lua_code_setup[*].stats_prefix` — separate stats per bundle.

## Files

- `lua_filter.{h,cc}` — filter and stream wrapper.
- `wrappers.{h,cc}`, `stream_wrapper.{h,cc}` — Lua bindings.
- `config.{h,cc}` — factory.
