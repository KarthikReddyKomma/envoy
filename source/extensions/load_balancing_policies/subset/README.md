# Subset (`envoy.load_balancing_policies.subset`)

Metadata-aware subset load balancer. It partitions the cluster's hosts into
named subsets based on configured `subset_selectors` (sets of metadata keys)
and, for each request, picks the subset whose keys match
`context->metadataMatchCriteria()`. It then delegates host selection within
that subset to a child LB policy (round_robin, least_request, maglev, etc.).
Use this for service-mesh routing to a specific version / region / shard of a
cluster without splitting the cluster in two.

Proto: `envoy.extensions.load_balancing_policies.subset.v3.Subset`.

## Files
- `config.h/cc` — `SubsetLbFactory` registered as
  `envoy.load_balancing_policies.subset`. `loadConfig` / `loadLegacy` build a
  `SubsetLoadBalancerConfig` that embeds the child LB's factory and config.
  `create` wraps a simple thread-aware LB around an internal `LbFactory` that
  constructs `SubsetLoadBalancer` per worker.
- `subset_lb.h/cc` — `SubsetLoadBalancer`, the nested `HostSubsetImpl` /
  `PrioritySubsetImpl` (a `PrioritySetImpl` that tracks only the matching
  hosts), subset selector map, and the per-selector fallback policy handling.
- `subset_lb_config.h/cc` — `SubsetSelector` and `SubsetLoadBalancerConfig`:
  parsing of fallback policy, default subset metadata, selector keys,
  locality-weight flags, and resolving the child LB factory from the proto.

## Load balancer class
`SubsetLoadBalancer` implements `Upstream::LoadBalancer` directly — it does
not extend `EdfLoadBalancerBase` or `ZoneAwareLoadBalancerBase`. Each subset
owns its own `PrioritySubsetImpl`, and inside that subset a child LB
(constructed via the child factory's worker factory) does the actual picking.

## Algorithm
Subset maintenance: on every host-set update, `SubsetLoadBalancer::update`
(and the `HostSubsetImpl`/`PrioritySubsetImpl` machinery in `subset_lb.cc`)
evaluates each configured selector against every host's metadata and
rebuilds the corresponding subset's `PrioritySet`. Each subset creates a
child LB instance the first time it is populated.

Per request, `chooseHost(context)` (`subset_lb.cc:184-198`):
1. If the metadata fallback policy is `FALLBACK_LIST`, look up
   `metadata_fallback_list` on the request's metadata match criteria. If
   present, iterate the fallback list, trying `chooseHostIteration` with each
   metadata override until one returns a host
   (`subset_lb.cc:200-216`).
2. Otherwise go straight to `chooseHostIteration(context)`
   (`subset_lb.cc:251-308`):
   - Optionally strip redundant keys (`allow_redundant_keys_`).
   - Call `tryChooseHostFromContext`, which looks up the subset matching the
     current metadata criteria via `findSubset` and delegates to that
     subset's child `lb_subset_->chooseHost(context)`
     (`subset_lb.cc:373-382`).
   - If the selector declares a per-selector fallback (`ANY_ENDPOINT`,
     `DEFAULT_SUBSET`, `KEYS_SUBSET`, `NOT_DEFINED`),
     `chooseHostForSelectorFallbackPolicy` handles it
     (`subset_lb.cc:335-355`).
   - Otherwise fall back to the cluster-level `fallback_subset_` and finally
     `panic_mode_subset_` (`subset_lb.cc:285-304`).

`SubsetSelector::singleHostPerSubset()` enables the optimization where every
subset has exactly one host (effectively sharded by metadata).

## Key decision points
- Factory create/load: `config.cc:50-101`.
- Per-worker LB creation: `config.cc:15-37` (`LbFactory::create`).
- Metadata fallback-list handling: `subset_lb.cc:184-216`.
- Core subset lookup and fallback cascade: `subset_lb.cc:251-308`.
- Per-selector fallback policy: `subset_lb.cc:335-385`
  (`chooseHostForSelectorFallbackPolicy`, `findSubset`).
- Locality-weight behavior for subsets: `HostSubsetImpl::determineLocalityWeights`
  in `subset_lb.h/cc` (controls `locality_weight_aware` / `scale_locality_weight`).

## Configuration
Proto fields consumed in `SubsetLoadBalancerConfig` (see `subset_lb_config.h`):
- `fallback_policy` — behavior when the request's metadata matches no subset
  (`ANY_ENDPOINT`, `DEFAULT_SUBSET`, `NO_FALLBACK`).
- `default_subset` — the metadata match used for `DEFAULT_SUBSET`.
- `subset_selectors[]` — list of selectors; each carries `keys`,
  `fallback_policy`, optional `fallback_keys_subset`, and
  `single_host_per_subset`.
- `metadata_fallback_policy` — whether to iterate a per-request
  `metadata_fallback_list`.
- `locality_weight_aware` / `scale_locality_weight` — propagate the parent
  cluster's locality weights into subsets.
- `list_as_any` — treat list-valued metadata as a set of allowed values.
- `allow_redundant_keys` — strip keys in the request criteria that aren't
  part of any selector.
- `subset_lb_policy` — the child LB policy applied inside each subset
  (resolved to a `TypedLoadBalancerFactory` by
  `SubsetLoadBalancerConfig`).

## Stats
Scope: inherits from the cluster. `ClusterLbStats` gains the subset-specific
counters (`lb_subsets_active`, `lb_subsets_created`, `lb_subsets_removed`,
`lb_subsets_selected`, `lb_subsets_fallback`, `lb_subsets_fallback_panic`,
`lb_subsets_single_host_per_subset_duplicate`). Of those, the ones set here
include `lb_subsets_fallback_` (`subset_lb.cc:291`) and
`lb_subsets_fallback_panic_` (`subset_lb.cc:298`); the rest are updated when
subsets are created/destroyed in the host-update paths.
