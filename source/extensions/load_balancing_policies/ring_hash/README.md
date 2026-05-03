# Ring Hash (`envoy.load_balancing_policies.ring_hash`)

Ketama-style consistent hashing. Each host is placed on a 64-bit ring at
multiple positions proportional to its weight; each request's hash is mapped
to the next-clockwise host on the ring. Minimal host movement when the
member set changes. Use this when you need session affinity by hash and want
fine-grained control over weight fidelity through the ring size (vs. Maglev's
fixed-size table).

Proto: `envoy.extensions.load_balancing_policies.ring_hash.v3.RingHash`.

## Files
- `config.h/cc` — `Factory` (`envoy.load_balancing_policies.ring_hash`).
  `loadConfig` builds `TypedRingHashLbConfig` including the hash policies via
  `TypedHashLbConfigBase`. `loadLegacy` supports
  `Cluster.RingHashLbConfig` and converts its `hash_function` /
  min/max ring size. `create` instantiates `RingHashLoadBalancer`.
- `ring_hash_lb.h/cc` — `RingHashLoadBalancer` (thread-aware),
  inner `Ring` class (the `HashingLoadBalancer`), `RingEntry`
  `{hash, host}` struct, and the `RingHashLoadBalancerStats` gauges.

## Load balancer class
`RingHashLoadBalancer` extends `ThreadAwareLoadBalancerBase`
(`../common/thread_aware_lb_impl.h`). The published per-worker picker is a
`Ring` object (a sorted `std::vector<RingEntry>`). As with Maglev, when
`hash_balance_factor > 0` the ring is wrapped in
`BoundedLoadHashingLoadBalancer`.

## Algorithm
Pre-computation in `Ring::Ring` (`ring_hash_lb.cc:128-223`):
1. Compute a `scale` that gives the least-weighted host
   `ceil(min_weight * min_ring_size / min_weight)` hashes, clamped to
   `max_ring_size`. This preserves "equal hashes for equal weights" when no
   weights are set.
2. For each `(host, weight)` pair in the normalized host list, generate
   `scale * weight` hashes (using running sums `current_hashes`/`target_hashes`
   so fractional hashes per host are handled stably) by hashing
   `hashKey(host) + "_" + i` with either `xxHash64` or `murmurHash2`.
3. Sort the ring by hash.
4. Record `size`, `min_hashes_per_host`, `max_hashes_per_host` gauges.

Per-request lookup in `Ring::chooseHost(hash, attempt)`
(`ring_hash_lb.cc:78-125`) is the classic ketama binary search for the
next-clockwise point:
- Maintain signed `lowp/highp/midp`.
- At each step, check whether `h <= midval && h > midval1` (i.e. we
  landed in the slot "between" two ring entries).
- Wrap to `midp = 0` when we fall off the end (lookups past the largest hash
  go to the smallest).
- For retries (`attempt > 0`), advance `midp = (midp + attempt) % ring_.size()`
  so retries tend to hit a different host.

The ring is built on the main thread inside
`ThreadAwareLoadBalancerBase::refresh` and published to workers as an
immutable `shared_ptr<const Ring>`.

## Key decision points
- Factory create: `config.cc:10-25`.
- Config validation (min vs. max ring size): `ring_hash_lb.cc:68-71`.
- `createLoadBalancer` wires bounded-load when configured:
  `ring_hash_lb.h:94-106`.
- Ring build loop and running sums: `ring_hash_lb.cc:176-208`.
- Ring lookup binary search: `ring_hash_lb.cc:78-125`.
- Legacy hash-function translation: `ring_hash_lb.cc:31-35`.

## Configuration
From the RingHash proto:
- `minimum_ring_size` — default 1024. Floor on ring size (`ring_hash_lb.h:113`).
- `maximum_ring_size` — default 8 * 1024 * 1024. Cap on ring size
  (`ring_hash_lb.h:114`).
- `hash_function` — `XX_HASH` (default) or `MURMUR_HASH_2`
  (`ring_hash_lb.cc:196-198`).
- `consistent_hashing_lb_config.use_hostname_for_hashing` — hash on hostname
  vs. address.
- `consistent_hashing_lb_config.hash_policy` — request hash policy, via
  `TypedHashLbConfigBase`.
- `consistent_hashing_lb_config.hash_balance_factor` — bounded-load factor.
  0 disables (`ring_hash_lb.cc:62-65`).
- `locality_weighted_lb_config` — enables locality-weighted balancing in the
  thread-aware base.

## Stats
Scope: `ring_hash_lb.` (`ring_hash_lb.cc:52`).

Gauges from `ALL_RING_HASH_LOAD_BALANCER_STATS` in `ring_hash_lb.h:41-50`,
populated in `Ring::Ring`:
- `size` — total entries in the built ring.
- `min_hashes_per_host` — smallest number of hashes given to any host.
- `max_hashes_per_host` — largest number of hashes given to any host.

A low `min_hashes_per_host` relative to `size` indicates poor weight
resolution — raise `minimum_ring_size`.
