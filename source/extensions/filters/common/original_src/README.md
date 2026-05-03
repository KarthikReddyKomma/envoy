# Original Source (shared filter infrastructure)

Shared helpers that let a filter force an upstream connection to use the downstream client's IP (and optionally port) as its source address via Linux `IP_TRANSPARENT`. This is the engine behind both the HTTP and listener `original_src` filters; those filters just extract the downstream address and hand it to this library to produce the right bundle of `Network::Socket::Option`s.

## Files
- `original_src_socket_option.h/cc` — `OriginalSrcSocketOption`, a `Network::Socket::Option` that rewrites a socket's local address at pre-bind time (`original_src_socket_option.h:16`).
- `socket_option_factory.h/cc` — free function `buildOriginalSrcOptions(source, mark)` that assembles the full option list (`socket_option_factory.h:12`, `socket_option_factory.cc:14`).
- `BUILD` — two `envoy_cc_library` targets: `original_src_socket_option_lib` and `socket_option_factory_lib`.

## Public interface
- `OriginalSrcSocketOption(Network::Address::InstanceConstSharedPtr src_address)` — asserts IP address (`original_src_socket_option.cc:13`).
- `bool setOption(Network::Socket&, SocketState)` — only acts on `STATE_PREBIND`, overwriting `connectionInfoProvider().setLocalAddress(src_address_)` (`original_src_socket_option.cc:20`).
- `void hashKey(std::vector<uint8_t>&)` — appends v4 (`uint32_t`) or v6 (`absl::uint128`) raw bytes so the connection pool keys by source IP (`original_src_socket_option.cc:38`).
- `absl::optional<Details> getOptionDetails(...)` — always `nullopt`, i.e. not reported in admin output (`original_src_socket_option.cc:55`).
- `bool isSupported() const override { return true; }` (`original_src_socket_option.h:39`).
- `Network::Socket::OptionsSharedPtr buildOriginalSrcOptions(source, uint32_t mark)` — factory used by the actual filters (`socket_option_factory.h:12`).

## Implementation logic
`buildOriginalSrcOptions` (`socket_option_factory.cc:14`):
1. Strips the port with `Network::Utility::getAddressWithPort(*source, 0)` so only the IP is carried (`socket_option_factory.cc:16`).
2. Pushes an `OriginalSrcSocketOption` (sets the local address at pre-bind) (`socket_option_factory.cc:22`).
3. If `mark != 0`, appends `SocketOptionFactory::buildSocketMarkOptions(mark)` so the kernel tags packets with `SO_MARK` for policy routing (`socket_option_factory.cc:26`).
4. Always appends `buildIpTransparentOptions()` (`IP_TRANSPARENT` / `IPV6_TRANSPARENT` — required by the kernel to bind a non-local address) (`socket_option_factory.cc:30`).
5. Always appends `buildBindAddressNoPort()` so an ephemeral port is still picked by the kernel (`IP_BIND_ADDRESS_NO_PORT`) (`socket_option_factory.cc:34`).

The returned `OptionsSharedPtr` is later merged into the upstream `ConnectionSocket`'s options by the caller; only `STATE_PREBIND` matters because `setLocalAddress` primes the `ConnectionInfoProvider` before `bind(2)` is invoked.

## Consumers
- `source/extensions/filters/listener/original_src/original_src.cc` — listener filter for TCP; applies these options to the accepted downstream socket so any upstream connection inherits them.
- `source/extensions/filters/http/original_src/original_src.cc` — HTTP filter; attaches options to the downstream `StreamInfo` upstream socket options list at decode time.

Both variants depend only on `buildOriginalSrcOptions` and feed back the downstream `remoteAddress()` as `source`.

## Stats / errors / failure modes
No stats are owned here; the consuming filters count failures. Failure modes:
- Constructor `ASSERT` trips if a non-IP address is passed (`original_src_socket_option.cc:17`).
- `setOption` returns `true` unconditionally but the kernel will reject the later `bind` if `CAP_NET_ADMIN` is absent or `IP_TRANSPARENT` is unsupported; that error is surfaced by the owning socket, not by this library.
- `hashKey` skips non-IPv4/IPv6 silently; call sites guarantee IP addresses, guarded by the ctor assert (`original_src_socket_option.cc:44`).
