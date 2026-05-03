# Mongo Proxy (`envoy.filters.network.mongo_proxy`)

A read+write sniffing filter that parses the MongoDB wire protocol on both directions of a TCP connection. It does not terminate or route; it observes OP_* messages to produce fine-grained stats (per-collection, per-callsite, per-command), optional JSON access logs, optional fault injection (fixed delay), and dynamic metadata. It also integrates with drain to let in-flight queries finish before the connection is closed.

Proto: `envoy.extensions.filters.network.mongo_proxy.v3.MongoProxy`.

## Files
- `config.h` / `config.cc` — `MongoProxyFilterConfigFactory`. Builds shared `MongoStats` with a command allow-list (defaults `{delete, insert, update}`, `config.cc:37-42`), optional `AccessLog`, and an optional `FaultDelayConfig`; installs a `ProdProxyFilter` via `filter_manager.addFilter(...)` (`config.cc:45-52`).
- `proxy.h` / `proxy.cc` — `AccessLog` (JSON lines with timestamp, message, upstream host), `ProxyFilter` (combined `Network::Filter` + `DecoderCallbacks` + `ConnectionCallbacks`), and `ProdProxyFilter` which overrides `createDecoder()` to return the real `DecoderImpl`.
- `codec.h` / `codec_impl.h/.cc` — protocol objects (`Message` hierarchy with `OpCode`, `GetMore`/`Insert`/`KillCursors`/`Query`/`Reply`/`Command`/`CommandReply`), `Decoder` interface, `DecoderCallbacks` sink, and `Encoder`.
- `bson.h` / `bson_impl.h/.cc` — BSON document/element parsing and printing used by `codec_impl`.
- `utility.h` / `utility.cc` — `QueryMessageInfo` extracts `$query`, `$explain`, `$hint`, `$comment` (callsite), `$maxTimeMS`, the target collection/command, and classifies the `QueryType` (`PrimaryKey`, `MultiGet`, `ScatterGet`, ...).
- `mongo_stats.h` / `mongo_stats.cc` — `MongoStats` interns StatNames for collection/callsite/cmd/query element trees used to build dynamic stat paths without re-tokenization.

## Lifecycle
- `onNewConnection()` (`proxy.h:119`) — unchanged `Continue`.
- `onData()` (`proxy.cc:378-383`) — appends to `read_buffer_`, runs `doDecode(read_buffer_)`. Returns `StopIteration` if a fault `delay_timer_` is active (requests must be held until the timer fires), else `Continue`.
- `onWrite()` (`proxy.cc:385-389`) — appends to `write_buffer_`, runs `doDecode(write_buffer_)`. Always returns `Continue`; the write path observes replies but never throttles.
- `doDecode()` (`proxy.cc:318-346`) — kill-switch checks:
  - `sniffing_` guards against re-entering after a decode error.
  - Runtime `mongo.proxy_enabled` (default 100%) can disable decoding globally; on disable the buffer is drained to prevent memory growth.
  - Lazily constructs `decoder_ = createDecoder(*this)`.
  - Wraps `decoder_->onData(buffer)` in `TRY_NEEDS_AUDIT`; on `EnvoyException` increments `decoding_error` and sets `sniffing_ = false` so we fail open for the remainder of the connection.
- `onEvent()` (`proxy.cc:355-376`) — on close cancels `delay_timer_`/`drain_close_timer_`. Increments `cx_destroy_{remote,local}_with_active_rq` when closing while `active_query_list_` is non-empty.

## Decoder callbacks (sink)
The `DecoderCallbacks` methods (`proxy.cc:92-289`) are invoked by the decoder per parsed OP_*:
- `decodeQuery` (`proxy.cc:121-181`): increments `op_query`, logs the full message, builds an `ActiveQuery` (incrementing `op_query_active` gauge), applies flag counters (`TailableCursor`, `NoCursorTimeout`, `AwaitData`, `Exhaust`), and either charges command-style stats (`cmd.<name>.total`) or collection-style stats (`collection.<coll>.query.total` plus optional `callsite.<cs>` and the `scatter_get`/`multi_get` variants). Pushes onto `active_query_list_`.
- `decodeReply` (`proxy.cc:205-272`): increments `op_reply`, matches `responseTo` against the FIFO of active queries, emits reply histograms `reply_num_docs`, `reply_size` (bytes), `reply_time_ms` via `chargeReplyStats()` (`proxy.cc:294-316`), erases the `ActiveQuery`, and if the list becomes empty AND the listener is draining AND `mongo.drain_close_enabled` is on (`proxy.cc:254-271`), schedules a zero-timeout timer to call `onDrainClose()` — which issues `FlushWrite` close after the current write has drained.
- `decodeInsert` / `decodeGetMore` / `decodeKillCursors` / `decodeCommand` / `decodeCommandReply` (`proxy.cc:92-119`, `274-288`): call `tryInjectDelay()`, bump the matching counter, and log. `decodeInsert`/`decodeQuery` also write dynamic metadata when `emit_dynamic_metadata` is true (`proxy.cc:77-90`).

## Decision / logic
- Fault injection (`proxy.cc:395-449`):
  - `delayDuration()` pulls percentage from `fault_config_` with runtime override `mongo.fault.fixed_delay.percent` and duration from `mongo.fault.fixed_delay.duration_ms`; returns nullopt if not active.
  - `tryInjectDelay()` starts a dispatcher timer; until it fires, `onData` returns `StopIteration`. On fire, `delayInjectionTimerCallback()` clears the timer and calls `continueReading()`.
- Drain close uses a zero-ms timer as a crude "write complete" hook (comment at `proxy.cc:261-265`).
- Access logging is double-gated: constructor drops `access_log_` if `mongo.connection_logging_enabled` is off at filter creation (`proxy.cc:67-72`); per-message `logMessage` re-checks `mongo.logging_enabled` (`proxy.cc:348-353`).

## Configuration
- `stat_prefix` (required) — produces `mongo.<prefix>.` for both global and per-command/collection stats.
- `access_log` — file path; JSON lines via `AccessLog::logMessage` (`proxy.cc:46-57`).
- `delay` — `FaultDelay` (percentage + duration) for fixed-delay fault injection.
- `emit_dynamic_metadata` — when true, per-message writes `{collection: [operation,...]}` under `envoy.filters.network.mongo_proxy` (`proxy.cc:77-90`, cleared per decode cycle at `proxy.cc:328-334`).
- `commands` — allow-list for built-in command StatNames; defaults to `delete/insert/update`.

## Stats
Prefix `mongo.<stat_prefix>.` (expanded from `ALL_MONGO_PROXY_STATS`, `proxy.h:49-72`):
- Connection: `cx_destroy_local_with_active_rq`, `cx_destroy_remote_with_active_rq`, `cx_drain_close`.
- Decoding: `decoding_error`, `delays_injected`.
- Op counters: `op_command`, `op_command_reply`, `op_get_more`, `op_insert`, `op_kill_cursors`, `op_query`, `op_reply`.
- Query flags: `op_query_await_data`, `op_query_exhaust`, `op_query_multi_get`, `op_query_no_cursor_timeout`, `op_query_no_max_time`, `op_query_scatter_get`, `op_query_tailable_cursor`.
- Reply flags: `op_reply_cursor_not_found`, `op_reply_query_failure`, `op_reply_valid_cursor`.
- Gauge: `op_query_active`.
- Dynamic (via `MongoStats`): `collection.<coll>.query.total`, `.scatter_get`, `.multi_get`; `callsite.<cs>.query.total`; `cmd.<name>.total`; reply histograms `*.reply_num_docs`, `*.reply_size`, `*.reply_time_ms`.

## Factory
`MongoProxyFilterConfigFactory` is legacy-registered as `envoy.filters.network.mongo_proxy` with alias `envoy.mongo_proxy` (`config.cc:58-60`). It uses `addFilter` (not `addReadFilter`) because `ProxyFilter` implements both the read and write halves of `Network::Filter`.
