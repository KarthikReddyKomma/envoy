# ZooKeeper Proxy (`envoy.filters.network.zookeeper_proxy`)

Sniffing L7 inspector for the ZooKeeper wire protocol. It decodes both directions of the TCP stream (requests and responses), emits per-opcode counters/histograms, publishes per-request dynamic metadata (opname, path, zxid, error, etc.), and measures response latency with optional fast/slow buckets against a per-opcode latency budget. The filter never modifies bytes: `onData`/`onWrite` simply forward the buffers and return `Continue`.

Proto: `envoy.extensions.filters.network.zookeeper_proxy.v3.ZooKeeperProxy`.

## Files
- `config.h/cc` — `ZooKeeperConfigFactory` registered as `envoy.filters.network.zookeeper_proxy` (`config.h`). Validates the per-opcode latency overrides (rejects duplicates) and builds the shared `ZooKeeperFilterConfig` (`config.cc:21`).
- `filter.h/cc` — `ZooKeeperFilterConfig` (`filter.h:251`) holding stats, stat-name set, op-code map, and latency-threshold lookup; `ZooKeeperFilter` (`filter.h:356`) implementing `Network::Filter` and `DecoderCallbacks`.
- `decoder.h/cc` — `DecoderImpl` (`decoder.h:134`), a stateful frame decoder that calls back into `DecoderCallbacks` on each parsed request/response.
- `utils.h/cc` — ZooKeeper wire-format helpers (varints, strings, etc.) used by the decoder.

## Lifecycle
- `onNewConnection()` — returns `Continue` (`filter.cc:210`). No per-connection state beyond `decoder_`.
- `initializeReadFilterCallbacks(cb)` — stores the callbacks (`filter.cc:196`). `decoder_` is constructed in the ctor (`filter.cc:193`) with a `createDecoder` that wires the filter as the `DecoderCallbacks` sink.
- `onData(data, end_stream)` — clears this-request dynamic metadata and delegates to `decoder_->onData(data)` (`filter.cc:200`).
- `onWrite(data, end_stream)` — clears metadata and delegates to `decoder_->onWrite(data)` (`filter.cc:205`).
- No `onEvent` hook is implemented; the filter has no connection-level timers.

## Decoder state machine
`DecoderImpl` keeps a per-direction ring-buffered view of the TCP stream and decodes full ZooKeeper frames. Each frame starts with a 32-bit big-endian length; the decoder defers parsing until the full frame is buffered. Requests are keyed by `xid`/`opcode` and stored in a pending table so responses can be correlated for latency. The decoder enforces `max_packet_bytes` (default 1 MiB, `config.cc:29`); oversized frames cause `onDecodeError` and connection close via the decoder's error path.

For each decoded frame the decoder invokes the matching `DecoderCallbacks::on*` hook:
- `onConnect`, `onPing`, `onAuthRequest`, `onGetDataRequest`, `onCreateRequest`, `onSetRequest`, `onGetChildrenRequest`, `onDeleteRequest`, `onExistsRequest`, `onGetAclRequest`, `onSetAclRequest`, `onSyncRequest`, `onCheckRequest`, `onMultiRequest`, `onReconfigRequest`, `onSetWatchesRequest`, `onSetWatches2Request`, `onAddWatchRequest`, `onCheckWatchesRequest`, `onRemoveWatchesRequest`, `onGetEphemeralsRequest`, `onGetAllChildrenNumberRequest`, `onCloseRequest` (declarations `filter.h:372`-`:398`).
- Byte accounting: `onRequestBytes` / `onResponseBytes` (`filter.cc:271`, `:286`) bump the global `request_bytes`/`response_bytes` counters plus per-opcode `*_rq_bytes`/`*_resp_bytes` when the matching `enable_per_opcode_*_bytes_metrics` flag is set.
- Response correlation: `onConnectResponse` / `onResponse` / `onWatchEvent` (`filter.h:400`-`:405`) fire the `*_resp` counter, record latency, and classify fast/slow.
- Decode errors: `onDecodeError` (`filter.cc:256`) bumps `decoder_error_` and, when `enable_per_opcode_decoder_error_metrics_` is set, the opcode-specific error counter (special-cased for `Connect` as `connect_decoder_error_`).

## Latency budget classification (`errorBudgetDecision`)
`ZooKeeperFilterConfig::errorBudgetDecision(opcode, latency)` (`filter.h:298`) returns `Fast`, `Slow`, or `None`:
- If `enable_latency_threshold_metrics_` is disabled, returns `None` — only the global `*_resp` counter is bumped.
- Otherwise compares the observed latency to the per-opcode override (from `latency_threshold_overrides` via `parseLatencyThresholdOverrides`) or falls back to `default_latency_threshold` (default 100 ms, `config.cc:37`). Responses faster than the threshold increment `<opname>_resp_fast`, slower ones `<opname>_resp_slow`.

## Dynamic metadata
For every decoded request, the filter populates dynamic metadata under the key `envoy.filters.network.zookeeper_proxy`:
- `setDynamicMetadata(...)` merges key/value pairs into `streamInfo().dynamicMetadata()` (`filter.cc:228`).
- `clearDynamicMetadata()` wipes the filter's sub-map at the start of `onData`/`onWrite` so each frame publishes a fresh view (`filter.cc:220`).
- Typical keys set by the various `on*Request` handlers: `opname`, `path`, `create_type`, `version`, `watch`, plus `bytes` for byte counters and `zxid`/`error` on responses.

## Decision / logic (selected branches)
- Packet-size guard: `max_packet_bytes` (`config.cc:29`) enforced in the decoder.
- Duplicate-opcode rejection: `ZooKeeperConfigFactory::createFilterFactoryFromProtoTyped` throws `EnvoyException("Duplicated opcode find in config: ...")` when `latency_threshold_overrides` contains the same opcode twice (`config.cc:47`).
- Per-opcode emission gating: three boolean toggles (`enable_per_opcode_request_bytes_metrics`, `_response_bytes_metrics`, `_decoder_error_metrics`) decide whether the opcode-scoped counters are emitted in addition to the global ones (`filter.cc:271`, `:286`, `:259`).
- Readonly connect: `onConnect(readonly)` distinguishes `connect_readonly_rq` vs. `connect_rq` and sets `opname` accordingly (`filter.cc:246`).

## Configuration
- `stat_prefix` — required; used as `<stat_prefix>.zookeeper.` (`config.cc:27`).
- `max_packet_bytes` — frame cap, default 1 MiB (`config.cc:29`).
- `enable_per_opcode_request_bytes_metrics`, `enable_per_opcode_response_bytes_metrics`, `enable_per_opcode_decoder_error_metrics` — toggles for per-opcode byte/error counters.
- `enable_latency_threshold_metrics`, `default_latency_threshold` (default 100 ms) — enable fast/slow classification.
- `latency_threshold_overrides` — list of `{opcode, threshold}` entries, validated unique (`config.cc:44`).

## Stats
Prefix `<stat_prefix>.zookeeper.` (see `ALL_ZOOKEEPER_PROXY_STATS`, `filter.h:30`):
- Global totals: `request_bytes`, `response_bytes`, `decoder_error`, `watch_event`.
- Per-opcode request counters and byte counters: `<op>_rq`, `<op>_rq_bytes` (e.g., `getdata_rq`, `create_rq_bytes`).
- Per-opcode response counters and byte counters: `<op>_resp`, `<op>_resp_bytes`.
- Per-opcode latency buckets (when enabled): `<op>_resp_fast`, `<op>_resp_slow`.
- Per-opcode decoder errors (when enabled): `<op>_decoder_error`.
- Latency histograms are registered dynamically via `stat_name_set_` and keyed by opname (`filter.h:288`, `:281`).

## Factory
`ZooKeeperConfigFactory::createFilterFactoryFromProtoTyped` (`config.cc:21`) validates input, builds `ZooKeeperFilterConfig`, and returns a lambda that calls `filter_manager.addFilter(std::make_shared<ZooKeeperFilter>(filter_config, time_source))` — a combined read+write filter. Registered via `REGISTER_FACTORY` at `config.cc:67`.
