# Redis Proxy (`envoy.filters.network.redis_proxy`)

A terminating Redis L7-over-L4 proxy. The filter decodes inline RESP commands from a downstream Redis client, dispatches each command through a pluggable command splitter (AUTH/QUIT/HELLO/PING handled locally; single-key commands hashed to an upstream shard; multi-key commands fragmented; cluster-aware commands fanned out), aggregates the upstream RESP responses, and writes them back in-order. Supports multi-cluster prefix routing, request mirroring, transactions (MULTI/EXEC), ASK/MOVED redirection with DNS caching, downstream AUTH (static or external gRPC), AWS IAM upstream auth, and pluggable fault injection.

Proto: `envoy.extensions.filters.network.redis_proxy.v3.RedisProxy` (plus per-cluster `RedisProtocolOptions`).

## Files
- `config.h` / `config.cc` — `RedisProxyFilterConfigFactory` (factory) and `ProtocolOptionsConfigImpl` (per-upstream-cluster AUTH credentials and AWS IAM, `config.h:50-155`). Wires together connection pools, router, command splitter, decoder/encoder factories, optional external auth gRPC client, then installs a `ProxyFilter` as a read filter (`config.cc:35-123`).
- `proxy_filter.h` / `proxy_filter.cc` — `ProxyFilterConfig` (stats, drain decision, DNS cache for redirects, downstream AUTH credentials) and `ProxyFilter` implementing `Network::ReadFilter`, RESP `DecoderCallbacks`, `ConnectionCallbacks`, and external-auth `AuthenticateCallback`.
- `command_splitter.h` — public interfaces `SplitRequest`, `SplitCallbacks`, `Instance` (`command_splitter.h:21-98`).
- `command_splitter_impl.h` / `command_splitter_impl.cc` — `InstanceImpl` builds a radix tree of `CommandHandler`s and exposes `makeRequest`. Concrete handlers: `SingleServerRequest`, `SimpleRequest`, `EvalRequest`, `TransactionRequest`, `MGETRequest`, `MSETRequest`, `SplitKeysSumResultRequest`, `FragmentedRequest`, plus `makeSingleServerRequest`/`makeFragmentedRequest` helpers that also fan out to `MirrorPolicy` upstreams (`command_splitter_impl.cc:32-80`).
- `router.h` / `router_impl.h` / `router_impl.cc` — `Router`, `Route`, and `MirrorPolicy` interfaces; `PrefixRoutes` owns a radix tree (`common/radix_tree.h`) of `Prefix` routes, each backed by a read/write connection pool and mirror policies (`router_impl.h:28-60`).
- `conn_pool.h` / `conn_pool_impl.h` / `conn_pool_impl.cc` — `ConnPool::Instance` hashes `hash_key` to an upstream host, manages per-host `Client`s, handles pipelining, MOVED/ASK redirects (re-dispatching via DNS cache), host removal, and ClusterSlots refreshes via the shared `ClusterRefreshManager`.
- `cluster_response_handler.h/.cc` — per-shard fan-out helpers used by `CLUSTER`-family commands.
- `info_command_handler.h/.cc` — local handler for `INFO` responses.
- `external_auth.h/.cc` — `GrpcExternalAuthClient` implementing `envoy.service.redis_auth.v3` to authenticate a downstream client via an external service with optional expiration.

## Lifecycle
- `onNewConnection()` — returns `Continue` unchanged (`proxy_filter.h:101`). Accounting happens in the constructor.
- Constructor (`proxy_filter.cc:75-91`) — bumps `downstream_cx_total`/`downstream_cx_active`, primes `connection_allowed_` based on whether static AUTH or external auth is configured, seeds `transaction_`, and adopts the optional `auth_client_`.
- `initializeReadFilterCallbacks()` (`proxy_filter.cc:98-106`) — registers connection stats so the socket's byte counters flow into `downstream_cx_{rx,tx}_bytes_{total,buffered}`.
- `onData()` (`proxy_filter.cc:310-325`) — `decoder_->decode(data)` turns buffered bytes into `RespValue`s. On `ProtocolError`, increments `downstream_cx_protocol_error`, encodes an `-downstream protocol error\r\n` reply, and closes `NoFlush`.
- `onRespValue()` (`proxy_filter.cc:108-121`) — one call per fully-decoded inbound command. Pushes a `PendingRequest` FIFO slot. If external auth is pending, the value is parked on the slot and released in `onAuthenticateExternal` (`proxy_filter.cc:204-208`); otherwise `processRespValue` hands it to the splitter.
- `processRespValue()` (`proxy_filter.cc:123-132`) — `splitter_.makeRequest(...)` returns a `SplitRequestPtr` that may be null when the command has already been completed synchronously (AUTH/QUIT/invalid); otherwise it is retained for `cancel()`.
- `onResponse()` (`proxy_filter.cc:274-308`) — ordered response flush: stores the reply on the matching `PendingRequest`, then drains the head of `pending_requests_` while each front slot has `pending_response_` set, encoding each into `encoder_buffer_` and writing once. Triggers `FlushWrite` close on `connection_quit_` (after a `QUIT` response is sent) or when the drain decision + `redis.drain_close_enabled` runtime fire.
- `onEvent()` (`proxy_filter.cc:134-153`) — on close, `cancel()`s each outstanding split request, closes the transaction, and cancels any in-flight external auth call.

## Command splitter
`Instance::makeRequest` (`command_splitter.h:81-97`) owns the hot path. Responsibilities in `command_splitter_impl.cc`:
- Look up the command by name in a radix tree of handlers built from `SupportedCommands` plus the configured `custom_commands`.
- Apply `FaultManager` checks: possibly inject `error_fault` (immediate `ERR fault injected` response) or a `delay_fault` (timer-gated before dispatch).
- For single-key commands (`SimpleRequest`): call `router_.upstreamPool(key, stream_info)` to resolve a `Route`, then `makeSingleServerRequest` (`command_splitter_impl.cc:32-51`) which also sends to each mirror upstream whose `shouldMirror(command)` passes.
- For multi-key commands (`MGET`/`MSET`/keys-sum): split into N fragments, each routed independently; aggregate fragment responses into a single array reply.
- For `EVAL`/`EVALSHA`: first key is the hash key.
- `MULTI`/`EXEC`/`WATCH`/`DISCARD`: coordinate with `Common::Redis::Client::Transaction` to pin all subsequent commands (and any mirror commands) to the same upstream client until `EXEC`/`DISCARD`.
- `AUTH`/`QUIT`/`HELLO`/`PING` are completed locally via `SplitCallbacks::onAuth/onQuit/onResponse` without contacting an upstream.
- Command timings are captured per-command as `CommandStats` counters `total/success/error/error_fault/delay_fault` and histogram `latency` (ms or us per `latency_in_micros`).

## Connection pool & routing
- `ConnPool::Instance` (`conn_pool.h:51-80`) wraps the cluster manager view of a specific upstream cluster: `makeRequest(hash_key, ...)` picks a host via consistent hashing, creates or reuses a `Common::Redis::Client`, and enqueues the RESP write. Redirect handling consults `dns_cache_` (`proxy_filter.h:71-72`) for ASK/MOVED addresses that may not be in the static cluster set.
- `PrefixRoutes` (router_impl) matches the command's first key against a radix tree of prefix `Route`s; a per-Route `read_command_policy.cluster` can split read vs write traffic; `request_mirror_policy` entries drive `MirrorPolicy` fan-out from `command_splitter_impl`.
- `ClusterRefreshManager` is a shared singleton (`config.cc:42`) that rate-limits CLUSTER SLOTS refreshes triggered by redirects across all pools.

## Downstream authentication
- Static (`proxy_filter.cc:210-263`): `onAuth(password)` checks `downstream_auth_passwords_`; `onAuth(username, password)` additionally matches `downstream_auth_username_` (with `default` synonym per Redis 6 ACL). On success, `connection_allowed_ = true`.
- External (`proxy_filter.cc:176-208`): when `external_auth_provider` is set, `onAuth(...)` dispatches an `authenticateExternal` gRPC call and marks `external_auth_call_status_ = Pending`; subsequent `onRespValue` calls park their values. `onAuthenticateExternal` translates `AuthenticateResponse.status` into `+OK`/`-ERR` replies, sets `connection_allowed_`, records `external_auth_expiration_epoch_`, and resumes parked requests.
- `connectionAllowed()` (`proxy_filter.cc:163-174`) lazily re-denies the connection once `external_auth_expiration_epoch_` is in the past (only when `enable_auth_expiration`).

## Configuration (highlights)
- `stat_prefix` (required) — `redis.<prefix>.` for proxy-level stats.
- `settings` (required) — `ConnPoolSettings`: timeouts, `enable_hashtagging`, `enable_redirection`, `dns_cache_config`, `read_policy`, `max_buffer_size_before_flush`, etc.
- `prefix_routes` — routes + `catch_all_route`; required to have at least one (`config.cc:51-53`).
- `downstream_auth_username` / `downstream_auth_password(s)` — `DataSource`-backed (`proxy_filter.cc:27-56`).
- `external_auth_provider` — gRPC auth; `enable_auth_expiration`.
- `faults` — per-command fault list.
- `custom_commands` — adds non-standard command names to the handler tree.
- `latency_in_micros` — changes `latency` histogram unit.
- Per-cluster `RedisProtocolOptions`: upstream AUTH credentials (username/password by address) and optional `aws_iam` configuration for IAM-authenticated upstreams (`config.h:50-155`, `config.cc:66-84`).

## Stats
Proxy-level, prefix `redis.<stat_prefix>.` (`proxy_filter.h:29-39`):
- Counters: `downstream_cx_total`, `downstream_cx_drain_close`, `downstream_cx_protocol_error`, `downstream_cx_rx_bytes_total`, `downstream_cx_tx_bytes_total`, `downstream_rq_total`.
- Gauges: `downstream_cx_active`, `downstream_cx_rx_bytes_buffered`, `downstream_cx_tx_bytes_buffered`, `downstream_rq_active`.

Additional trees:
- Per command (`command.<name>.`): `total`, `success`, `error`, `error_fault`, `delay_fault`, histogram `latency`.
- Per upstream cluster (`cluster.<name>.redis_cluster.*`): connection pool stats for each host.

## Factory
`RedisProxyFilterConfigFactory` (legacy-registered as `envoy.filters.network.redis_proxy` alias `envoy.redis_proxy`, `config.cc:128`) overrides both `createFilterFactoryFromProtoTyped` and `createProtocolOptionsTyped`. The factory builds the full proxy topology (DNS cache, refresh manager, per-cluster conn pools, router, fault manager, command splitter, optional external auth client) once and then the returned lambda installs a fresh `ProxyFilter` per connection sharing that topology (`config.cc:110-123`).
