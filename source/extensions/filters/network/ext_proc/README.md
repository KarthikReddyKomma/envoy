# Network External Processor (`envoy.filters.network.ext_proc`)

A combined `Network::ReadFilter` + `WriteFilter` that streams raw TCP bytes in both directions over a bidirectional gRPC stream to an external processor. The processor can inspect, mutate, inject, drop or reset bytes (per direction and per end-of-stream) as well as command the connection to `CONTINUE`, close with FIN (`CLOSE`), or reset (`CLOSE_RST`).

Proto: `envoy.extensions.filters.network.ext_proc.v3.NetworkExternalProcessor`.

## Files
- `config.h/cc` — `NetworkExtProcConfigFactory::createFilterFactoryFromProtoTyped`. Validates that `grpc_service` is set and that at least one direction is not `SKIP` (`config.cc:20-34`), builds an immutable `Config`, constructs the client via `createExternalProcessorClient(...)`, and adds the filter as *both* a read and write filter via `filter_manager.addFilter` (`config.cc:38-54`).
- `ext_proc.h/cc` — `Config`, per-connection `NetworkExtProcFilter`, `MessageTimeoutManager` (independent read/write timers), and the filter-state `NetworkExtProcLoggingInfo` aggregator.
- `client_impl.h/cc` — gRPC stream wrapper (`ExternalProcessorClient` / `ExternalProcessorStream`) and `createExternalProcessorClient` factory (`config.cc:49`).

## Lifecycle
- `initializeReadFilterCallbacks()` stashes read callbacks, registers `downstream_callbacks_` on the connection, lazily constructs the `MessageTimeoutManager`, and calls `initializeLoggingInfo()` which publishes a `NetworkExtProcLoggingInfo` into `filterState()` under `envoy.filters.network.ext_proc` (connection lifespan, mutable) (`ext_proc.cc:124-134`, `101-122`).
- `initializeWriteFilterCallbacks()` just stores `write_callbacks_` (`ext_proc.cc:136-139`).
- `onNewConnection()` — logs and returns `Continue`; no gRPC work here (`ext_proc.cc:141-144`).
- `onData(data, end_stream)` — read path:
  1. If `processing_mode.process_read == SKIP`, return `Continue` (`ext_proc.cc:150-153`).
  2. `openStream()` (lazy gRPC stream open, see below); on `Error` dispatch through `handleStreamError()`, on `IgnoreError` continue (`ext_proc.cc:155-159`).
  3. `sendRequest(..., is_read=true)`. Returns `StopIteration` — downstream bytes resume only via `injectReadDataToFilterChain` in `onReceiveMessage` (`ext_proc.cc:160-162`).
- `onWrite(data, end_stream)` — mirror of `onData` but for the response path, guarded by `processing_mode.process_write` (`ext_proc.cc:164-181`).
- `onDownstreamEvent(event)` (via `DownstreamCallbacks`) — closes the gRPC stream on local/remote close (`ext_proc.cc:183-188`).
- Destructor closes the stream (`ext_proc.cc:99`).

## Stream state machine
`StreamOpenState openStream()` (`ext_proc.cc:232-266`):
- `processing_complete_` → `IgnoreError` (no-op, let traffic flow).
- Existing `stream_` → `Ok`.
- Otherwise starts a new gRPC stream with a `ParentContext` tied to the connection stream-info and `setBufferBodyForRetry(grpc_service.has_retry_policy())`; bumps `streams_started_` on success, `stream_open_failures_` + returns `Error` on failure.

`handleStreamError()` (`ext_proc.cc:190-206`): marks processing complete, closes the stream, then forks on `failure_mode_allow`: either increments `failure_mode_allowed_` and returns `Continue`, or calls `closeConnection("ext_proc_stream_error", FlushWrite)` and returns `StopIteration`.

`sendRequest(data, end_stream, is_read)` (`ext_proc.cc:304-352`):
- Calls `updateCloseCallbackStatus(true, is_read)` which reference-counts `disableClose` to delay connection close while a request is outstanding (`ext_proc.cc:208-230`).
- Records bytes via `NetworkExtProcLoggingInfo::addBytesProcessed`.
- Builds a `ProcessingRequest`, calls `addDynamicMetadata` to copy matching namespaces from `streamInfo().dynamicMetadata()` into `request.metadata` (`ext_proc.cc:532-563`).
- Fills `read_data` or `write_data`, marks the relevant pending flag, records the call start time, and starts the per-direction timer.
- `stream_->send(request, /*end_stream=*/false)`, increments `stream_msgs_sent_`, and drains the local buffer so Envoy does not forward the original bytes (`ext_proc.cc:347-351`).

`onReceiveMessage(res)` (`ext_proc.cc:354-405`):
- Spurious messages (after `processing_complete_`) bump `spurious_msgs_received_` (`ext_proc.cc:355-360`).
- `handleConnectionStatus` is dispatched first (`ext_proc.cc:501-530`): `CONTINUE` logs; `CLOSE` → `closeConnection(..., FlushWrite)` + mark complete; `CLOSE_RST` → `closeConnection(..., AbortReset)` + mark complete.
- Then, depending on which oneof is set, the data is injected into the filter chain with `injectReadDataToFilterChain` or `injectWriteDataToFilterChain` and the per-direction `disableClose` counter is decremented. Missing payload increments `empty_response_received_`.

`onGrpcError` (`ext_proc.cc:407-431`) and `onGrpcClose` (`ext_proc.cc:433-438`) both mark processing complete, close the stream, bump the corresponding counter, and for errors optionally close the connection unless `failure_mode_allow` is set.

`MessageTimeoutManager` (`ext_proc.cc:10-53`) owns independent `read_timer_`/`write_timer_`. Zero `message_timeout` disables the timer (`ext_proc.cc:15-20`). On fire it calls `NetworkExtProcFilter::handleMessageTimeout(is_read)` which increments `message_timeouts_`, clears pending flags, re-enables close callbacks, closes the stream, and either stays open (fail-open) or closes the connection (`ext_proc.cc:268-298`).

`closeStream()` (`ext_proc.cc:459-481`) stops all timers, clears start times/pending flags, closes the underlying stream and bumps `streams_closed_`.

## Configuration (highlights)
Captured by `Config` (`ext_proc.h:95-146`):
- `failure_mode_allow` — fail-open on gRPC errors / timeouts.
- `processing_mode.process_read` / `process_write` — enum, `SKIP` bypasses that direction.
- `grpc_service` — processor endpoint; used to derive `config_with_hash_key_`.
- `metadata_options.forwarding_namespaces.untyped` / `typed` — namespaces forwarded in `ProcessingRequest.metadata`.
- `stat_prefix` — suffix under `network_ext_proc.`.
- `message_timeout` — per-direction timeout, default 200 ms (`ext_proc.h:132, 109`).

No per-route config; state is per-connection.

## Stats
Emitted under `network_ext_proc.<stat_prefix>.` (`ext_proc.h:134-137`). From `ALL_NETWORK_EXT_PROC_FILTER_STATS` (`ext_proc.h:22-39`):
- `streams_started`, `streams_closed`, `streams_grpc_error`, `streams_grpc_close`, `stream_open_failures`.
- `stream_msgs_sent`, `stream_msgs_received`.
- `read_data_sent`, `write_data_sent`, `read_data_injected`, `write_data_injected`.
- `empty_response_received`, `spurious_msgs_received`.
- `connections_closed`, `connections_reset`.
- `failure_mode_allowed`, `message_timeouts`.

`NetworkExtProcLoggingInfo` (`ext_proc.h:49-90`) additionally records per-direction bytes, gRPC call count/errors, and min/max/total latency, along with peer and local addresses, available from filter state for access logs.
