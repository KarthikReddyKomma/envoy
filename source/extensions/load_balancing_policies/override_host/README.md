# Override Host (`envoy.load_balancing_policies.override_host`)

A delegating LB used by dynamic-forwarding / endpoint-picker flows. For each
request it tries to pick a host listed in a request header or in dynamic
metadata (populated by an upstream "endpoint picker" extension, e.g. a
`LbTrafficExtension`). If no override is present, or no listed host can be
found in the cluster, it defers to a configured fallback LB policy. Use this
when an external component (a traffic extension) decides the concrete endpoint
per request and Envoy should honor that decision while still having a safe
fallback.

Proto: `envoy.extensions.load_balancing_policies.override_host.v3.OverrideHost`.

## Files
- `config.h/cc` — `OverrideHostLoadBalancerFactory` registered as
  `envoy.load_balancing_policies.override_host`. `loadConfig` validates the
  proto, builds an `OverrideHostLbConfig` (including the nested fallback LB's
  factory + config), and `create` instantiates the child fallback LB and wraps
  it in `OverrideHostLoadBalancer`.
- `load_balancer.h/cc` — `OverrideHostLbConfig` (typed config with parsed
  override sources), `OverrideHostLoadBalancer` (thread-aware),
  `OverrideHostLoadBalancer::LoadBalancerImpl` (per-worker picker), and
  `OverrideHostLoadBalancer::LoadBalancerFactoryImpl` (factory used by workers).
- `override_host_filter_state.h` — `OverrideHostFilterState`, a
  `StreamInfo::FilterState::Object` that caches the parsed host list for the
  current request and tracks which entry has been consumed (so per-request
  retries walk the list).

## Load balancer class
`OverrideHostLoadBalancer` implements `Upstream::ThreadAwareLoadBalancer` but
does not extend any of the common bases. Host selection is delegated either to
the request-supplied list or to the child fallback LB (which can itself be any
registered LB policy).

## Algorithm
`chooseHost(context)` -> `chooseHostInternal(context)` in
`load_balancer.cc:153-190`:

1. If there is no context or no request stream info, call the fallback LB's
   `chooseHost` directly.
2. Look up `OverrideHostFilterState` on the request. If absent, build one by
   calling `getSelectedHosts(context)` which reads each configured override
   source in priority order: first any configured request header, then any
   configured dynamic metadata key. The extracted list (e.g. a
   comma-separated endpoints string) is parsed once and cached in the filter
   state (`load_balancer.h:159-174`).
3. If the list is empty, fall back to the child LB.
4. Otherwise call `getEndpoint(override_host_state)` which pops the next host
   string off the list and asks `findHost` to look it up in the cluster's host
   map by address; the first listed host that resolves wins. Subsequent retries
   of the same request continue through the remaining list entries.
5. If nothing in the list resolves, fall back to the child LB.

After the decision, `addSelectedHostKey` optionally writes the selected
endpoint into dynamic metadata under `selected_host_key` so that downstream
filters or telemetry can observe which host was used
(`load_balancer.cc:199-229`).

`peekAnotherHost` simply forwards to the child LB's peek.

## Key decision points
- Config load and fallback factory resolution: `config.cc:25-47`.
- Override-source parsing (header or metadata, exclusive):
  `load_balancer.cc:57-82`.
- Per-request dispatch: `load_balancer.cc:153-190`.
- Host-list consumption for retries: `override_host_filter_state.h:16-60`.
- Selected-host-key metadata write: `load_balancer.cc:199-229`.

## Configuration
Proto fields consumed by `OverrideHostLbConfig::make`:
- `fallback_policy` — required. A list of LB extensions; the first registered
  one wins and its config is parsed here.
- `override_host_sources[]` — ordered list of sources. Each must set
  exactly one of `header` (an HTTP header name) or `metadata` (a metadata
  key). Validated in `makeOverrideSources`
  (`load_balancer.cc:67-82`).
- `selected_host_key` — optional metadata key path. When set, the LB writes
  the picked endpoint's `address:port` into this path.

## Stats
None specific to this policy. The fallback LB contributes its own
`ClusterLbStats`.
