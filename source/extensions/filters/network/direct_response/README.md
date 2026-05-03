# Direct Response Network Filter (`envoy.filters.network.direct_response`)

Terminal L4 filter that writes a fixed payload to the connection and closes it. Useful for health probes, placeholders, or returning a static reply on a TCP port without standing up a backend. The payload is loaded once at configuration time from any `envoy.config.core.v3.DataSource` (inline string, file, or environment variable).

Proto: `envoy.extensions.filters.network.direct_response.v3.Config`.

## Files
- `config.cc` — `DirectResponseConfigFactory` (config.cc:19), a `Common::ExceptionFreeFactoryBase` subclass that reads the payload from the configured DataSource and creates the filter. This factory is marked terminal (config.cc:39-43).
- `filter.h` / `filter.cc` — `DirectResponseFilter`, the `Network::ReadFilter` that writes the payload and closes the connection.

## Factory behavior
`DirectResponseConfigFactory::createFilterFactoryFromProtoTyped` (config.cc:27):
1. Calls `Config::DataSource::read(config.response(), /*allow_empty=*/true, api)` to resolve the payload bytes (config.cc:30). `RETURN_IF_NOT_OK_REF` propagates a bad file / env reference as `absl::Status` without throwing.
2. Captures the resolved string by move into a lambda that adds a fresh `DirectResponseFilter` — constructed with the payload — to each new connection's filter manager (config.cc:34).

Registered with `REGISTER_FACTORY(DirectResponseConfigFactory, NamedNetworkFilterConfigFactory)` (config.cc:49) under name `NetworkFilterNames::get().DirectResponse`. Because `isTerminalFilterByProtoTyped` returns `true` (config.cc:39), this filter must be the last in the chain.

## Filter lifecycle
`DirectResponseFilter` is only a `Network::ReadFilter` (filter.h:16); no write filter, no connection callbacks.

- `initializeReadFilterCallbacks(callbacks)` (filter.h:25): stores `read_callbacks_` and calls `connection().enableHalfClose(true)`. Half-close must be allowed so the downstream write followed by close does not drop buffered bytes before the peer reads them.
- `onNewConnection()` (filter.cc:11):
  1. If `response_` is non-empty, constructs a `Buffer::OwnedImpl(response_)` and calls `connection.write(data, /*end_stream=*/true)` (filter.cc:15-16). The `ASSERT(0 == data.length())` validates the connection drained the buffer fully.
  2. Sets the StreamInfo response code details to `StreamInfo::ResponseCodeDetails::get().DirectResponse` (filter.cc:19) so access logs/stat sinks can attribute the close.
  3. Calls `connection.close(Network::ConnectionCloseType::FlushWrite)` (filter.cc:21) — the payload is drained before FIN is sent.
  4. Returns `StopIteration`. Since the filter is terminal, this prevents any later filter from being invoked and signals the filter manager that the chain is done.
- `onData(...)` (filter.h:21) always returns `Continue` — but in practice the connection is already closing from `onNewConnection`, so no data is expected. The method exists only to satisfy the `Network::ReadFilter` interface.

## Decision points
- Empty payload branch: filter.cc:14 — skip the `write()`; just set response code details and close. Allows `Config { response: {} }` to be a pure TCP-level rejector that sends only FIN.
- Half-close on: filter.h:27 — required so the flushed write is delivered before close.

## Configuration fields
- `response` (`envoy.config.core.v3.DataSource`, required) — static payload. Resolved once by `Config::DataSource::read` at config time (config.cc:30), so runtime file changes are not picked up.

## Stats
The filter emits no counters or gauges of its own. Connection outcomes are attributed via `StreamInfo` response code details (`direct_response`, filter.cc:19) and surface through the listener's access log / downstream connection counters.
