# Original Source Listener Filter (`envoy.filters.listener.original_src`)

Captures the downstream remote address when the connection is accepted and attaches socket options that make the upstream connection bind to that address (IP-transparent, typically via `IP_TRANSPARENT` and `SO_MARK`). The filter itself does not open an upstream connection; it stores options on the downstream `ConnectionSocket` via `addOptions`, and those options are later applied to the upstream socket by the connection pool as part of the option-hash-based pool selection.

Proto: `envoy.extensions.filters.listener.original_src.v3.OriginalSrc`.

## Files
- `original_src.h` / `original_src.cc` — `OriginalSrcFilter` (the listener filter). `onData` and `maxReadBytes` are no-ops (`original_src.h:26`-`original_src.h:30`).
- `config.h` / `config.cc` — `Config`: holds `use_port_` (from proto `bind_port`) and `mark_` (SO_MARK value) (`config.h:9`, `config.cc:10`).
- `original_src_config_factory.h` / `original_src_config_factory.cc` — `OriginalSrcConfigFactory`: validates the proto, constructs `Config`, and installs the filter via `filter_manager.addAcceptFilter`; registers under `envoy.filters.listener.original_src` plus deprecated alias `envoy.listener.original_src` (`original_src_config_factory.cc:35`).

## Lifecycle
- `onAccept(cb)` — reads `socket.connectionInfoProvider().remoteAddress()` (`original_src.cc:18`), asserts it is non-null. If the address is not IP (e.g. Unix domain), returns `Continue` without acting (`original_src.cc:25`). Otherwise builds the socket options via `Filters::Common::OriginalSrc::buildOriginalSrcOptions(address, mark)` and attaches them with `socket.addOptions(...)` (`original_src.cc:29`-`original_src.cc:31`). Always returns `Network::FilterStatus::Continue`.
- `onData()` — returns `Continue` (unused).
- `maxReadBytes()` — returns 0.

## Decision / logic
- The actual option set (including whether to bind port or just address, and the `SO_MARK` value) is produced by `Filters::Common::OriginalSrc::buildOriginalSrcOptions` (see `source/extensions/filters/common/original_src/`). This filter merely feeds the downstream remote address and the configured mark into that helper.
- Because options are attached, not immediately applied, they become part of the upstream socket pool key so that different downstream sources are not multiplexed onto the same upstream connection.

## Configuration
- `bind_port` (bool) — when set, the options also force the upstream to bind to the downstream source port. Stored as `use_port_` (`config.cc:11`).
- `mark` (uint32) — value passed to `SO_MARK` on the upstream socket (`config.cc:11`).

## Stats
None emitted directly by this filter.
