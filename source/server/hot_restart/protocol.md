# Hot Restart — Wire Protocol & Takeover Handshake

This document describes the message protocol and the end-to-end choreography of a hot
restart, message by message.

## 1. The message type

All communication uses a single protobuf, `envoy.HotRestartMessage` (`hot_restart.proto`),
with a `request` / `reply` oneof. Requests the child sends and replies the parent returns:

| Request | Reply payload |
|---------|---------------|
| `ShutdownAdmin` | `original_start_time_unix_seconds`, `enable_reuse_port_default` |
| `PassListenSocket{address, worker_index, network_namespace}` | `fd` (via `SCM_RIGHTS`; `-1` if none) |
| `Stats` | gauges, counter deltas, dynamic name spans, memory/connection values |
| `DrainListeners` | (none) |
| `Terminate` | (none) |
| `TestConnection` | (none; used as a reachability probe) |

An unrecognized request gets a reply with `didnt_recognize_your_last_message = true`.

## 2. Framing on the wire

```
+---------------------------+----------------------------------+
| 8-byte big-endian length  | serialized HotRestartMessage ... |
+---------------------------+----------------------------------+
```

- Sent over an `AF_UNIX`/`SOCK_DGRAM` socket in datagrams of up to `MaxSendmsgSize = 4096`
  bytes; larger protos are split across continuation datagrams and reassembled by length.
- A `PassListenSocket` reply additionally carries the file descriptor as ancillary
  `SCM_RIGHTS` control data, and therefore must fit in a single datagram (asserted).
- `receiveHotRestartMessage` supports a blocking mode (child waiting for a reply) and a
  non-blocking mode (parent's evented read loop), and always provides a control buffer so an
  fd can arrive on any recv.

## 3. The takeover handshake

The child drives the sequence during its own startup (`source/server/server.cc`), blocking
for each reply except the fire-and-forget `DrainListeners` and `Terminate`.

```mermaid
sequenceDiagram
    autonumber
    participant C as Child (epoch N)
    participant P as Parent (epoch N-1)

    Note over C: bind child socket; compute parent address
    opt --skip-hot-restart-on-no-parent
        C->>P: TestConnection (allow_failure)
        Note over C: if refused -> fall back to a normal cold start
    end

    C->>P: ShutdownAdmin
    P->>P: shutdownAdmin() (release admin port)
    P-->>C: Reply{ original_start_time, reuse_port_default }
    Note over C: inherit start time -> uptime stays continuous

    loop per listener / per worker
        C->>P: PassListenSocket{ address, worker_index }
        P->>P: find matching bound listener
        P-->>C: Reply{ fd via SCM_RIGHTS }  (or fd = -1)
    end

    C->>P: Stats
    P-->>C: Reply{ gauges, counter deltas, dynamics, mem, conns }
    Note over C: StatMerger applies deltas -> metrics continuous

    Note over C: build listeners on inherited fds;<br/>start workers; all ports accepting
    C->>P: DrainListeners
    P->>P: drainListeners() (begin graceful drain)
    Note over P,C: undeliverable QUIC/UDP packets<br/>forwarded P -> C on the _udp channel

    Note over C: after parentShutdownTime elapses
    C->>P: Terminate
    P->>P: kill(getpid(), SIGTERM) -> exit
```

## 4. Where each step is triggered in the server

| Handshake step | Triggered from |
|----------------|----------------|
| Construct restarter, attach shared memory | `source/exe/stripped_main_base.cc` |
| `ShutdownAdmin` | `InstanceBase::initializeOrThrow()` — learns `original_start_time_` |
| `PassListenSocket` (per listener/worker) | the listener manager while building listeners |
| `mergeParentStatsIfAny` (`Stats`) | `InstanceBase` during initialization |
| `DrainListeners` | `startWorkers()` completion, after all workers are accepting |
| `startParentShutdownSequence` → `Terminate` | `DrainManagerImpl` arms a `parent_shutdown_timer_` |

## 5. UDP / QUIC forwarding during drain

While the parent is draining, QUIC connections it can no longer deliver to (because the
child now owns the socket) are forwarded to the child on the **separate** `_udp` channel as
`ForwardedUdpPacket` messages. The child's `UdpForwardingContext` routes each packet to the
correct worker. This keeps long-lived QUIC connections alive across the handover. UDP
listeners can also register a "parent drained" callback so they start *paused* and only
begin listening once the parent has finished draining (so a draining QUIC listener catches
its own packets first).

## 6. Failure and edge cases

- **Epoch 0 (no parent):** every child request short-circuits (`parent_terminated_` starts
  `true`), so the same code runs a normal cold start.
- **Incompatible binary:** attaching the parent's shared memory `RELEASE_ASSERT`s on a
  `version_` mismatch — the takeover aborts; ops must do a full restart.
- **Previous Envoy still initializing:** the `SHMEM_FLAGS_INITIALIZING` guard throws
  `"previous envoy process is still initializing"`, so the launcher backs off and retries.
- **Parent unreachable with `--skip-hot-restart-on-no-parent`:** the `TestConnection` probe
  fails gracefully and the child starts cold instead of erroring out.
- **Parent crash:** the robust shared-memory mutexes let survivors recover the log locks
  rather than deadlock; `PR_SET_PDEATHSIG` ensures children of a dead launcher exit.
