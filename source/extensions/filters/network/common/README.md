# Network Filter Common Library

Infrastructure library shared by network (L4) filters. Contains no filter of its own; instead provides the factory base template used by every `NamedNetworkFilterConfigFactory`, plus the Redis client/codec/stats subsystem that `redis_proxy`, `redis` cluster discovery, and health checking all build on.

Scope: `source/extensions/filters/network/common/`
Proto: none (pure library; users include its headers).

## Files

### Factory plumbing
- `factory_base.h` — CRTP bases:
  - `FactoryCommon<ConfigProto, ProtocolOptionsProto>` (factory_base.h:19): implements
    `createEmptyConfigProto`, `createEmptyProtocolOptionsProto`, `name`, `isTerminalFilterByProto` — the boilerplate every `NamedNetworkFilterConfigFactory` needs. Stores the registered name and a terminal flag (factory_base.h:47).
  - `FactoryBase<...>` (factory_base.h:72): adds `createFilterFactoryFromProto` and forwards to a subclass-supplied `createFilterFactoryFromProtoTyped` (factory_base.h:87). Used by most filters (echo, connection_limit, rbac, tcp_proxy, ...).
  - `ExceptionFreeFactoryBase<...>` (factory_base.h:93): same as `FactoryBase` but the typed hook returns `absl::StatusOr<Network::FilterFactoryCb>` instead of throwing. Used where config parsing may fail gracefully (direct_response, dynamic_modules, http_connection_manager).
  - Default `createProtocolOptionsTyped` returns `InvalidArgumentError` (factory_base.h:59); subclasses override when the filter participates in per-cluster protocol options.

### Redis subsystem (`common/redis/`)
Shared between the Redis proxy network filter, Redis health checker, and Redis cluster discovery.

- `codec.h` / `codec_impl.cc` — RESP protocol:
  - `RespType` enum covers `Null`, `SimpleString`, `BulkString`, `Integer`, `Error`, `Array`, plus the internal zero-copy `CompositeArray` (codec.h:25) used to rewrite `MGET`/`MSET` without cloning elements.
  - `RespValue` (codec.h:31) is a tagged-union variant holding any RESP value.
  - `Decoder` / `Encoder` (codec.h:171, codec.h:201) stream-decode and encode bytes through a `Buffer::Instance`. Parse errors throw `ProtocolError` (codec.h:218).
- `client.h` / `client_impl.{h,cc}` — single-connection upstream client:
  - `Client` interface (client.h:74) extends `Event::DeferredDeletable`: `makeRequest`, `initialize(auth_username, auth_password)`, `active`, `close`, `sendAwsIamAuth`.
  - `ClientCallbacks` (client.h:35): `onResponse`, `onFailure`, `onRedirection` (for MOVED/ASK).
  - `ReadPolicy` enum (client.h:117): `Primary`, `PreferPrimary`, `Replica`, `PreferReplica`, `Any`, `LocalZoneAffinity`, `LocalZoneAffinityReplicasAndPrimary`.
  - `Config` (client.h:132) exposes pool tunables: `opTimeout`, `maxBufferSizeBeforeFlush`, `bufferFlushTimeoutInMs`, `enableRedirection`, `enableHashtagging`, `enableCommandStats`, `connectionRateLimitEnabled`, etc. `ConfigImpl` parses `ConnPoolSettings` (client_impl.cc:28).
  - `ClientImpl` (client_impl.h:74) pipelines requests on one TCP connection; implements `DecoderCallbacks` and `Network::ConnectionCallbacks`. See lifecycle below.
  - `ClientFactoryImpl` (client_impl.h:184) creates a `ClientImpl`, wires an `UpstreamReadFilter` into the upstream connection, optionally runs `AUTH`/`READONLY`/AWS IAM auth.
  - Transaction support: `Transaction` struct (client.h:252) bundles the primary client plus mirror clients, tracks `current_client_idx_`, and closes them on scope exit. `MultiRequest` / `EmptyArray` are pre-built constant RESP values.
- `aws_iam_authenticator_impl.{h,cc}` — `AwsIamAuthenticatorImpl` (aws_iam_authenticator_impl.h:49) wraps `Extensions::Common::Aws::Signer` to produce an ElastiCache IAM auth token. `AwsIamAuthenticatorFactory::initAwsIamAuthenticator` (aws_iam_authenticator_impl.h:73) builds one from the proto. Default service is `elasticache`, token expiration 60s (aws_iam_authenticator_impl.h:18).
- `redis_command_stats.{h,cc}` — `RedisCommandStats` (redis_command_stats.h:18) creates per-command counters and latency histograms under `<scope>.upstream_commands.<cmd>.{total,success,failure,latency}`. `getCommandFromRequest` interns the lowercased command token. Only emitted when `enable_command_stats` is true.
- `supported_commands.h` / `supported_commands.cc` — `SupportedCommands` (supported_commands.h:21) exposes sets used by the proxy router:
  - `simpleCommands` — key-hashed to one shard (supported_commands.h:25).
  - `multiKeyCommands` — `del`, `mget`, `mset`, `touch`, `unlink`, `msetnx` — fanned out and reassembled (supported_commands.h:53).
  - `evalCommands`, `objectCommands` — special hashing (supported_commands.h:61, 69).
  - `hashMultipleSumResultCommands` — responses summed across shards (supported_commands.h:76).
  - `ClusterScopeCommands`, `randomShardCommands` — broadcast or random-shard routing (supported_commands.h:84, 92).
  - `transactionCommands` — `multi`/`exec`/`discard`/`watch`/`unwatch` (supported_commands.h:111).
  - `writeCommands` + `isReadCommand` (supported_commands.h:169) drive read/write replica split.
  - `commandSubcommandValidationMap` (supported_commands.h:101) whitelists subcommands (e.g., `cluster` -> `info|slots|keyslot|nodes`).
- `fault.{h,cc}` / `fault_impl.{h,cc}` — `Fault` (fault.h:22) and `FaultManager` (fault.h:35) inject `Delay` or `Error` faults on selected commands with runtime-controlled probability.
- `utility.{h,cc}` — small helpers: `AuthRequest` (utility.h:14), `ReadOnlyRequest` (utility.h:22), `AskingRequest` (utility.h:28), `GetRequest`/`SetRequest`, and `makeError` (utility.h:20).

## ClientImpl lifecycle

`ClientFactoryImpl::create` (client_impl.cc:347) builds the client, opens the upstream connection, then calls `initialize(auth_username, auth_password)` unless an AWS IAM authenticator was provided.

- `ClientImpl::create` (client_impl.cc:68): creates the connection through the host, installs `UpstreamReadFilter` (client_impl.h:125) so decoded bytes are fed back via `ClientImpl::onData`, registers itself as `ConnectionCallbacks`, calls `connect()` and `noDelay(true)`.
- `initialize` (client_impl.cc:327): sends `AUTH` (username+password or password-only) and — when `readPolicy() != Primary` — sends `READONLY`. Both responses are consumed by `null_pool_callbacks`.
- `sendAwsIamAuth` (client_impl.cc:86): enables `queue_enabled_`, then either calls `add_auth` immediately or registers a credentials-pending callback; `add_auth` resolves the token via the signer and calls `makeRequestImmediate` with the generated `AUTH`.
- `makeRequest` (client_impl.cc:129): encodes into `encoder_buffer_`, appends a `PendingRequest`, increments per-command stats if enabled (client_impl.cc:135), and either flushes when the buffer exceeds `maxBufferSizeBeforeFlush` (client_impl.cc:151), or arms `flush_timer_` for `bufferFlushTimeoutInMs`. When pipeline depth is 1 and the connection is up, `connect_or_op_timer_` is (re)armed for `opTimeout`.
- `onData` (client_impl.cc:207): delegates to the RESP `Decoder`; `ProtocolError` closes the connection and bumps `upstream_cx_protocol_error_` + `rq_error_`.
- `onRespValue` (client_impl.cc:257): pops the head `PendingRequest`, routes the response. If `enableRedirection()` is on and the error starts with `MOVED` or `ASK`, calls `onRedirection` with the parsed `host:port`; `CLUSTERDOWN` becomes `onFailure`; everything else becomes `onResponse`. Cancelled requests drop the value and bump `upstream_rq_cancelled_`.
- `onEvent` (client_impl.cc:223): on `RemoteClose`/`LocalClose`, records the destroy, fails every pending request, disables `connect_or_op_timer_`. On `Connected`, sets `connected_=true` and arms the op timer.
- `onConnectOrOpTimeout` (client_impl.cc:194): bumps `upstream_rq_timeout_` or `upstream_cx_connect_timeout_` and closes with `NoFlush`; reports `LocalOriginTimeout` to the outlier detector (unless `disableOutlierEvents`).
- `PendingRequest::cancel` (client_impl.cc:320): sets `canceled_=true`; the response is later dropped in `onRespValue` — the pipeline is not blown away.

## Stats produced via this library

Reported on the upstream cluster and host (via `cluster().trafficStats()` and `host().stats()`), not in a filter-specific scope:
- `upstream_cx_total`, `upstream_cx_active`, `upstream_cx_connect_fail`, `upstream_cx_connect_timeout`, `upstream_cx_protocol_error` — incremented in `ClientImpl`/its ctor/dtor (client_impl.cc:105-118, 200-253).
- `upstream_rq_total`, `upstream_rq_active`, `upstream_rq_timeout`, `upstream_rq_cancelled`, `rq_error` — incremented around `PendingRequest` lifetime (client_impl.cc:305-318, 197-275).
- Per-command metrics (when enabled) at `<prefix>.upstream_commands.<cmd>.{total,success,failure,latency}` + aggregate `upstream_rq_time` (redis_command_stats.h).

## Who uses this

- `FactoryBase` / `ExceptionFreeFactoryBase`: every in-tree network filter factory.
- `Common::Redis::Client`: `filters/network/redis_proxy`, `health_checkers/redis`, `clusters/redis`.
- `Common::Redis::Fault`: `filters/network/redis_proxy` command splitter.
- `Common::Redis::SupportedCommands`: `filters/network/redis_proxy` router and command splitter.
