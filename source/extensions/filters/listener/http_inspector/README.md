# HTTP Inspector Listener Filter (`envoy.filters.listener.http_inspector`)

Peeks at the first bytes of an accepted connection to detect whether it is HTTP/1.0, HTTP/1.1, or HTTP/2 (via the h2 connection preface), then writes the appropriate ALPN application protocol (`http/1.0`, `http/1.1`, or `h2c`) onto the `ConnectionSocket`. This lets downstream filter-chain matching and protocol selection route the connection to the right listener filter chain without needing the client to negotiate ALPN over TLS. Uses either the Balsa parser or the legacy HTTP/1 parser with no-op callbacks to validate the first request line.

Proto: `envoy.extensions.filters.listener.http_inspector.v3.HttpInspector`.

## Files
- `http_inspector.h` / `http_inspector.cc` — `Filter` (the listener filter), `Config` (holds stats scope and buffer-size constants), and `NoOpParserCallbacks` (a `Http::Http1::ParserCallbacks` stub whose callbacks all return `CallbackResult::Success` and ignore payloads, `http_inspector.h:64`).
- `config.cc` — `HttpInspectorConfigFactory`: creates a shared `Config` and installs the filter with `filter_manager.addAcceptFilter(...)` (`config.cc:22`-`config.cc:27`); registers the filter name `envoy.filters.listener.http_inspector` plus deprecated alias `envoy.listener.http_inspector` (`config.cc:34`, `config.cc:40`).

## Lifecycle
- `onAccept(cb)` — if the socket already has a non-empty detected transport protocol other than `raw_buffer`, the inspector is unnecessary and returns `Continue` (`http_inspector.cc:82`). Otherwise records `cb_` and returns `StopIteration` so the manager will call `onData` when bytes arrive (`http_inspector.cc:89`).
- `onData(buffer)` — calls `parseHttpHeader` on the peeked slice. `ParseState::Done` or `Error` mark the filter finished via `done(success)` and return `Continue`; `ParseState::Continue` doubles `requested_read_bytes_` up to `MAX_INSPECT_SIZE` (64KiB) and returns `StopIteration`. If the buffer already exhausts `MAX_INSPECT_SIZE` without a decision, increments `read_error` and closes the socket (`http_inspector.cc:58`-`http_inspector.cc:72`).
- `maxReadBytes()` — returns `requested_read_bytes_`, which starts at `DEFAULT_INITIAL_BUFFER_SIZE` = 8KiB (`http_inspector.h:112`, `http_inspector.h:57`).

## Decision / logic
- HTTP/2 detection compares the leading bytes against the HTTP/2 preface `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` (`http_inspector.cc:24`, `http_inspector.cc:95`). A full prefix match sets `protocol_ = "HTTP/2"` and returns `Done`.
- HTTP/1.x detection scans for the first `\n`, feeds the request line to the HTTP/1 parser (`Http::Http1::BalsaParser` when `envoy.reloadable_features.http_inspector_use_balsa_parser` is on, else `LegacyHttpParserImpl`) and reads `parser_->isHttp11()` to choose `HTTP/1.1` vs `HTTP/1.0` (`http_inspector.cc:30`-`http_inspector.cc:38`, `http_inspector.cc:126`). Any other parser status is `Error`.
- On success `done(true)` maps the detected HTTP version to the ALPN name (`AlpnNames::get().Http10`, `Http11`, `Http2c`) and calls `cb_->socket().setRequestedApplicationProtocols({protocol})` (`http_inspector.cc:171`). On failure `done(false)` only increments `http_not_found_` and leaves the socket alone.

## Configuration
- Empty proto (`HttpInspector`). Behavior is only adjusted via the `envoy.reloadable_features.http_inspector_use_balsa_parser` runtime flag (`http_inspector.cc:30`).

## Stats
Counters are rooted under the listener's scope with prefix `http_inspector.` (`http_inspector.cc:22`) and declared at `http_inspector.h:25`:
- `http_inspector.read_error` — buffer hit `MAX_INSPECT_SIZE` without a decision.
- `http_inspector.http10_found`, `http_inspector.http11_found`, `http_inspector.http2_found` — HTTP version detected.
- `http_inspector.http_not_found` — parsing finished without identifying HTTP.
