# Maglev (`envoy.load_balancing_policies.maglev`)

Maglev consistent-hashing load balancer. Builds a fixed-size lookup table
(default 65537 slots, must be prime) from the set of upstream hosts; each
request's hash is looked up in the table to pick a host. Minimal disruption
when hosts are added or removed, but stronger host-balance than Ring Hash for
the same memory. Use this when you need session affinity by hash and want a
more compact, O(1)-lookup consistent-hashing alternative to Ring Hash.

Proto: `envoy.extensions.load_balancing_policies.maglev.v3.Maglev`.

## Files
- `config.h/cc` — `Factory` (`envoy.load_balancing_policies.maglev`).
  `loadConfig` constructs a `TypedMaglevLbConfig` (including hash policies via
  `TypedHashLbConfigBase`); `loadLegacy` converts from
  `Cluster.MaglevLbConfig`. `Factory::create` instantiates
  `MaglevLoadBalancer`.
- `maglev_lb.h/cc` — `MaglevLoadBalancer` (thread-aware), the abstract
  `MaglevTable` and its three implementations (`OriginalMaglevTable`,
  `CompactMaglevTable`, `DegenerateMaglevTable`), the anonymous `MaglevFactory`
  that picks the right implementation for the host set, and the
  `MaglevLoadBalancerStats` struct.

## Load balancer class
`MaglevLoadBalancer` extends `ThreadAwareLoadBalancerBase`
(`../common/thread_aware_lb_impl.h`). The actual picker is a
`MaglevTable` deriving from `HashingLoadBalancer`. When
`hash_balance_factor` is non-zero, the table is wrapped in
`BoundedLoadHashingLoadBalancer` so overloaded hosts spill to the next entry.

## Algorithm
1. On host-set change, `MaglevLoadBalancer::createLoadBalancer` is called by
   `ThreadAwareLoadBalancerBase::refresh` with
   `(normalized_host_weights, min, max)`. It asks
   `MaglevFactory::createMaglevTable` to pick the best table representation
   (`maglev_lb.cc:38-53`):
   - 1 host: `DegenerateMaglevTable` (returns that host always).
   - Small host count and supported platform: `CompactMaglevTable` (stores
     hosts once plus a `BitArray` index into them).
   - Otherwise: `OriginalMaglevTable` (a dense
     `vector<HostConstSharedPtr>` of size `table_size`).
2. `MaglevTable::constructMaglevTableInternal` (`maglev_lb.cc:81-129`) sorts
   hosts by hash key for determinism, computes each host's
   `(offset, skip)` as `xxHash64(key)` and `xxHash64(key, 1) % (table_size-1)+1`,
   and then runs the Maglev permutation algorithm (pseudocode listing 1 of the
   paper) weighted by the normalized host weights. Each iteration advances a
   host's `current_permutation_ = offset + n * skip (mod table_size)`, claiming
   the next empty slot and filling it with the host.
3. Per request, the hash-based LB wrapper computes the request hash and calls
   `chooseHost(hash, attempt)`. Each table type implements it as
   `table_[hash % table_size]`. Retries XOR the hash with a pattern derived
   from `attempt` to pick a different slot
   (`maglev_lb.cc:264-296`).

Pre-computation happens on the main thread in
`ThreadAwareLoadBalancerBase::refresh`; the built `MaglevTable` is published
to workers through a shared pointer so lookups are lock-free.

## Key decision points
- Table representation selection: `maglev_lb.cc:11-31` (`shouldUseCompactTable`)
  and `maglev_lb.cc:38-53` (`MaglevFactory::createMaglevTable`).
- Table build (paper's listing 1): `maglev_lb.cc:131-169`
  (`OriginalMaglevTable::constructImplementationInternals`) and
  `maglev_lb.cc:176-228` (`CompactMaglevTable::constructImplementationInternals`).
- Hash lookup and retry skew: `maglev_lb.cc:264-296`.
- `hash_balance_factor` wrap: `maglev_lb.cc:74-78`.
- Constant: `DefaultTableSize = 65537` (`maglev_lb.h:64`).
- Prime-number validation: `maglev_lb.cc:303-306`.

## Configuration
From the Maglev proto (`maglev_lb.cc:298-307`):
- `table_size` — lookup table size. Must be prime. Default 65537.
- `consistent_hashing_lb_config.hash_policy` — the per-route hash policy,
  compiled via `TypedHashLbConfigBase`.
- `consistent_hashing_lb_config.use_hostname_for_hashing` — hash on hostname
  rather than address.
- `consistent_hashing_lb_config.hash_balance_factor` — bounded-load factor. 0
  disables it. Non-zero wraps the table in `BoundedLoadHashingLoadBalancer`.
- `locality_weighted_lb_config` — enable locality-weighted balancing in the
  thread-aware base.

## Stats
Scope: `maglev_lb.` (`maglev_lb.cc:299`).

Gauges defined by `ALL_MAGLEV_LOAD_BALANCER_STATS` in `maglev_lb.h:38-46`:
- `max_entries_per_host` — maximum number of table slots assigned to any host.
- `min_entries_per_host` — minimum across hosts.

They are set in `constructMaglevTableInternal` after the table is built
(`maglev_lb.cc:117-124`).
