# QUIC Server Preferred Address

Implements the QUIC transport parameter `preferred_address`, which tells a
connecting client to migrate to a different server address after the
handshake completes. Two factories share the same runtime
`ServerPreferredAddressConfig`:

- `quic.server_preferred_address.fixed` - addresses supplied directly in
  the proto.
- `quic.server_preferred_address.datasource` - addresses sourced from
  `DataSource`s (file/inline/env), useful when the control plane doesn't
  know the externally-reachable address but the deployment context does.

Protos:
- `envoy.extensions.quic.server_preferred_address.v3.FixedServerPreferredAddressConfig`
- `envoy.extensions.quic.server_preferred_address.v3.DataSourceServerPreferredAddressConfig`

## Files
- `server_preferred_address.h/cc` - `ServerPreferredAddressConfig` (the
  runtime strategy), holding per-family SPA and optional DNAT addresses.
- `fixed_server_preferred_address_config.h/cc` - factory for the fixed
  (proto-supplied) variant.
- `datasource_server_preferred_address_config.h/cc` - factory for the
  `DataSource`-driven variant.

## Interface
- Runtime base: `Quic::EnvoyQuicServerPreferredAddressConfig`.
- Factory base: `Quic::EnvoyQuicServerPreferredAddressConfigFactory`.
- Extension names: `quic.server_preferred_address.fixed` and
  `quic.server_preferred_address.datasource`.

## Logic
- `ServerPreferredAddressConfig::getServerPreferredAddresses` takes the
  listener's local address, reuses its port when the SPA's port is 0,
  and fills the QUICHE `Addresses` struct with SPA and DNAT pairs for v4
  and v6 (`server_preferred_address.cc:17`).
- The fixed factory parses either `ipv4_address` / `ipv6_address`
  strings or the structured `ipv4_config` / `ipv6_config` messages.
  Validation enforces: `dnat_address` requires `address`; `address`
  port must be 0 unless a `dnat_address` is supplied; the address family
  matches the slot (`fixed_server_preferred_address_config.cc:62`).
- The datasource factory reads each address from a `DataSource`,
  trims whitespace, parses it as a QUIC IP, and verifies family
  (`datasource_server_preferred_address_config.cc:16`). Ports come from a
  separate `DataSource` and are allowed only when a `dnat_address` is
  present.

## Key decision points
- `fixed_server_preferred_address_config.cc:79` - enforces the
  "port must be zero" rule unless DNAT is configured; otherwise the
  listener's own port is inherited from `local_address`.
- `datasource_server_preferred_address_config.cc:65` - uses a `uint32_t`
  for `absl::SimpleAtoi` then bounds-checks against `UINT16_MAX`
  because `SimpleAtoi` does not support `uint16_t` directly.
- Both factories throw `ProtoValidationException` on any mismatch so the
  listener fails to load rather than silently advertising a bad address.

## Configuration
- Fixed: `ipv4_address` / `ipv6_address` (simple string form) or
  `ipv4_config` / `ipv6_config` (full `AddressFamilyConfig` with
  `address` and optional `dnat_address`).
- DataSource: `ipv4_config` / `ipv6_config` with `address`
  (`DataSource`), optional `port` (`DataSource`), optional
  `dnat_address`.

## Stats / errors
- None. Validation errors surface as `ProtoValidationException` at
  listener config time.
