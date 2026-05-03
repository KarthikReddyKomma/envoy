# Original Destination Listener Filter (`envoy.filters.listener.original_dst`)

Restores the connection's pre-redirection destination address onto the `ConnectionSocket`. For IP sockets this reads `SO_ORIGINAL_DST` (Linux iptables redirect) via `Network::Utility::getOriginalDst`. For Envoy-internal listeners it reads the destination from dynamic metadata or filter state that an upstream filter already populated. The restored address is used by listener-filter-chain matching and by the connection balancer to select the actual upstream when `use_original_dst` listener-level routing is desired.

Proto: `envoy.extensions.filters.listener.original_dst.v3.OriginalDst`.

## Files
- `original_dst.h` / `original_dst.cc` — `OriginalDstFilter` (the listener filter) plus `FilterNameValues` singleton exposing the metadata key (`local`) and two filter-state keys used for EnvoyInternal listeners (`original_dst.h:15`-`original_dst.h:23`). `onData` and `maxReadBytes` are no-ops (`original_dst.h:42`-`original_dst.h:46`).
- `config.cc` — `OriginalDstConfigFactory`: enforces Windows preconditions (traffic direction required; compile-time `SO_ORIGINAL_DST` support) before returning the factory callback (`config.cc:32`-`config.cc:40`). Also registers `BaseAddressObjectFactory` handlers for the two filter-state keys (`config.cc:63`-`config.cc:75`).

## Lifecycle
- `onAccept(cb)` — inspects `socket.addressType()` and branches (`original_dst.cc:23`).
  - `Type::Ip`: calls `getOriginalDst(socket)` (overridable, default forwards to `Network::Utility::getOriginalDst`, `original_dst.cc:17`). If the address is valid, calls `socket.connectionInfoProvider().restoreLocalAddress(original_local_address)` (`original_dst.cc:60`). On Windows (`WIN32` + `win32SupportsOriginalDestination()`), for `OUTBOUND` traffic it also queries Windows Filtering Platform redirect records with `SIO_QUERY_WFP_CONNECTION_REDIRECT_RECORDS` and stashes them into `UpstreamSocketOptionsFilterState` (`original_dst.cc:37`-`original_dst.cc:54`).
  - `Type::EnvoyInternal`: first checks dynamic metadata at namespace `envoy.filters.listener.original_dst` key `local`; if present and parsable, calls `restoreLocalAddress` with that address (`original_dst.cc:65`-`original_dst.cc:70`). Otherwise falls back to filter state keyed `envoy.filters.listener.original_dst.local_ip` (`original_dst.cc:75`-`original_dst.cc:79`). Additionally, if filter state `envoy.filters.listener.original_dst.remote_ip` holds an address, calls `setRemoteAddress` on the connection info provider (`original_dst.cc:81`-`original_dst.cc:85`).
- Always returns `Network::FilterStatus::Continue` (`original_dst.cc:92`).
- `onData()` — returns `Continue` (unused).
- `maxReadBytes()` — returns 0.

## Decision / logic
- The listener's `traffic_direction` is threaded through the filter constructor but is only consulted on Windows to decide whether to query redirect records (`original_dst.h:30`, `original_dst.cc:35`).
- On non-Windows platforms, a connection that was not redirected simply gets its current local address back from `getOriginalDst`, so `restoreLocalAddress` is a no-op for the filter-chain matcher.
- EnvoyInternal flow lets an upstream filter (typically on the producing listener) communicate both the original destination and the original source via filter state/dynamic metadata, so the EnvoyInternal-facing listener can recreate them.

## Configuration
- `traffic_direction` is sourced from the listener info, not from the filter proto (`config.cc:43`).
- Proto is otherwise empty.

## Stats
None emitted directly by this filter.
