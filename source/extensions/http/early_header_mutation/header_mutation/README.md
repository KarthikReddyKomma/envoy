# Early Header Mutation

An `EarlyHeaderMutation` plugin consumed by the HTTP Connection Manager
(HCM). The HCM invokes any configured early header mutation extensions very
early in request processing - before the router and before most filters -
so they can mutate request headers that later routing / filter decisions
depend on. This extension runs a standard `HeaderMutations` evaluator over
each request.

Proto: `envoy.extensions.http.early_header_mutation.header_mutation.v3.HeaderMutation`.

## Files
- `header_mutation.h/cc` - `HeaderMutation` (the extension itself).
- `config.h/cc` - `EarlyHeaderMutationFactory` that the HCM consults.

## Interface
- Implements `Envoy::Http::EarlyHeaderMutation` (see
  `envoy/http/early_header_mutation.h`). The HCM iterates configured
  early-header-mutation extensions and calls `mutate` on each.
- Factory implements `Envoy::Http::EarlyHeaderMutationFactory`, registered
  via `REGISTER_FACTORY`.

## Logic
- Constructor builds a shared `Envoy::Http::HeaderMutations` from the
  proto's `mutations` field via `HeaderMutations::create`; errors throw.
  This reuses the common header mutation engine used by router headers,
  allowing `%HOST%`/substitution formatter style syntax.
- `mutate(headers, stream_info)` delegates to
  `mutations_->evaluateHeaders(headers, {&headers}, stream_info)`. Only
  request headers are passed as context (the early path has no response
  yet). Always returns `true` (the HCM uses the return value only to halt
  on hard errors, which this extension never raises).

## Key decision points
- `header_mutation.cc:15` - `HeaderMutations::create` is invoked at ctor
  time so configuration errors (bad formatter, bad regex, disallowed
  header) fail fast at config load.
- `header_mutation.cc:21` - passes request headers as both the mutation
  target and the formatter context; response/trailer context is null.

## Configuration
`HeaderMutation.mutations` is a list of
`envoy.config.common.mutation_rules.v3.HeaderMutation` entries (add,
append, remove, with optional match rules). No other knobs.

## Stats / errors
No dedicated stats. Config-time errors (rejected by the mutation rules
engine) propagate as exceptions from the factory.
