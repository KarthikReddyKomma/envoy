# Round Robin (`envoy.load_balancing_policies.round_robin`)

Simple round-robin balancing. When all hosts share the same weight, it uses
a per-`HostsSource` incrementing index, giving O(1) picks. When weights
differ, it falls back to the shared EDF scheduler so that hosts are visited
in weight-proportional frequency. Use this as the default workhorse LB for
clusters where you don't need affinity, adaptive P2C, or consistent hashing.

Proto: `envoy.extensions.load_balancing_policies.round_robin.v3.RoundRobin`.

## Files
- `config.h/cc` — `Factory` (`envoy.load_balancing_policies.round_robin`) via
  `Common::FactoryBase<RoundRobinLbProto, RoundRobinCreator>`.
  `RoundRobinCreator::operator()` builds the per-worker
  `RoundRobinLoadBalancer`. `TypedRoundRobinLbConfig` (declared in
  `round_robin_lb.h`, defined in `round_robin_lb.cc`) converts legacy
  `CommonLbConfig` / `RoundRobinLbConfig` into the new proto.
- `round_robin_lb.h/cc` — `RoundRobinLoadBalancer` plus the
  unweighted `rr_indexes_` map.

## Load balancer class
`RoundRobinLoadBalancer` extends `EdfLoadBalancerBase`
(`../common/load_balancer_impl.h`). The weighted path comes "for free" from
the base; the subclass supplies the unweighted fast path, the host-weight
extractor, and the RR-index bookkeeping.

## Algorithm
`chooseHostOnce` is inherited from `EdfLoadBalancerBase`
(`../common/load_balancer_impl.cc:1145`):
- If a priority has hosts whose original weights differ,
  `scheduler.edf_` is present and `pickAndAdd` is used with
  `hostWeight(host)`.
- Otherwise, the unweighted path calls `unweightedHostPick(hosts, source)` in
  `round_robin_lb.h:69-79`, which returns
  `hosts[rr_indexes_[source]++ % hosts.size()]`. `rr_indexes_` is keyed by
  `HostsSource` so each priority / locality / health class has its own
  independent cursor.

`refreshHostSource(source)` inserts a new cursor seeded from `seed_` the
first time a `HostsSource` appears, so load balancers across a fleet don't
stay in lock-step (`round_robin_lb.h:44-52`). It also resets
`peekahead_index_` since peek order would otherwise be stale.

`unweightedHostPeek` returns the host at `rr_indexes_[source] +
peekahead_index_++`, allowing speculative preconnects to read ahead without
advancing the real cursor (`round_robin_lb.h:60-67`).

`hostWeight(host)` returns `host.weight()` but applies slow-start when
applicable (`round_robin_lb.h:53-58`).

## Key decision points
- Factory create: `config.cc:10-24`.
- Unweighted pick path: `round_robin_lb.h:69-79`.
- Unweighted peek path: `round_robin_lb.h:60-67`.
- Per-source cursor init with anti-sync seed: `round_robin_lb.h:44-52`.
- Slow-start integration in `hostWeight`: `round_robin_lb.h:53-58`.
- Legacy proto translation: `round_robin_lb.cc:9-13`.

## Configuration
Proto fields (see `round_robin_lb.h:30-41` and `../common/load_balancer_impl.h`):
- `slow_start_config` — optional slow-start window, aggression, min weight.
- `locality_lb_config` — locality weighted / zone-aware routing; converted
  from legacy `CommonLbConfig` via `LoadBalancerConfigHelper`.

There are no RR-specific algorithmic knobs.

## Stats
Uses `ClusterLbStats` from the cluster. No RR-specific stats struct.
