# Random (`envoy.load_balancing_policies.random`)

Random load balancing: pick a host uniformly at random from the currently
usable host vector. Usually faster and sometimes better than round-robin in the
face of irregular request lengths, because multiple load balancers making
independent random choices still produce even load. Use it when you want
stateless, memory-free balancing and don't care about session affinity.

Proto: `envoy.extensions.load_balancing_policies.random.v3.Random`.

## Files
- `config.h/cc` — `Factory` (`envoy.load_balancing_policies.random`) via
  `Common::FactoryBase<RandomLbProto, RandomCreator>`.
  `TypedRandomLbConfig` wraps the proto and can be built from the legacy
  `CommonLbConfig` by `LoadBalancerConfigHelper::convertLocalityLbConfigTo`.
  `RandomCreator::operator()` builds a per-worker `RandomLoadBalancer`.
- `random_lb.h/cc` — `RandomLoadBalancer` with `chooseHostOnce`,
  `peekAnotherHost`, and the shared `peekOrChoose` helper.

## Load balancer class
`RandomLoadBalancer` extends `ZoneAwareLoadBalancerBase`
(`../common/load_balancer_impl.h`). It does not use EDF: the shared EDF
machinery is skipped, and zone-aware priority/locality selection happens once
per call via `hostSourceToUse`.

## Algorithm
`chooseHostOnce(context)` simply calls `peekOrChoose(context, peek=false)`
(`random_lb.cc:13-15`):

1. Draw a 64-bit random number (with a per-call stash to deduplicate between
   peek and pick: `LoadBalancerBase::random(peek)`).
2. Call `hostSourceToUse(context, random_hash)` to decide which priority /
   locality / health class to pull hosts from. This returns an optional
   `HostsSource` (`nullopt` when `fail_traffic_on_panic` is set and we would
   otherwise route despite panic).
3. Get the host vector via `hostSourceToHosts(*hosts_source)`; if empty,
   return null.
4. Return `hosts_to_use[random_hash % hosts_to_use.size()]`.

`peekAnotherHost` implements pre-connect by reusing the same path with
`peek=true`, bounded by `tooManyPreconnects(stashed_random_.size(), total_healthy_hosts_)`
(`random_lb.cc:6-11`).

## Key decision points
- Factory create: `config.cc:18-31`.
- `chooseHostOnce` entry: `random_lb.cc:13-15`.
- Core pick logic: `random_lb.cc:17-30`.
- Preconnect guard: `random_lb.cc:6-11`.
- Legacy locality config conversion: `config.cc:14-16`.

## Configuration
Proto fields consumed in `RandomLoadBalancer` construction:
- `locality_lb_config` — locality-weighted or zone-aware routing. Converted
  from the legacy `Cluster.CommonLbConfig` when going through `loadLegacy`.

No algorithm knobs of its own — randomness comes from the shared
`Random::RandomGenerator`.

## Stats
Uses `ClusterLbStats` from the cluster. No random-specific stats.
