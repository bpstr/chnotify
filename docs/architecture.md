# chnotify architecture proposal

Status: draft for implementation review  
Date: 2026-11-12  
Project: [bpstr/chnotify](https://github.com/bpstr/chnotify)

## 1. Purpose and scope

**chnotify means channel notify. It is a small Rust notification distributor.**

A producer submits a process notification locally. The daemon accepts ownership, saves it, and distributes it to configured channels. It owns the outbox, credentials, retry scheduling, and delivery bookkeeping.

The producer supplies the facts about its process. Receivers decide what those facts mean and whether to display or alert on them. Process discovery, inferred health, dashboards, log collection, and job execution belong outside this component.

Use one executable with `send` and `daemon` commands. The daemon runs in the foreground under a service manager. A second daemon binary is unnecessary for the first implementation.

The first acceptance environment is Linux. Portable Rust boundaries should leave other operating systems possible; claiming platform support requires exercising their IPC and service lifecycle.

## 2. Independent contracts

| Contract | Owner | Responsibility |
|---|---|---|
| Notification schema | Dedicated specification repository | Fields, meaning, lexical types, required values, versioning, extensions, format bindings and conformance fixtures |
| Codec | chnotify, implementing those bindings | Decode and encode one notification without changing its meaning |
| Local submission protocol | chnotify | Framing, event identity, admission errors and durable acceptance |
| Channel adapter | chnotify | Endpoint connection, authentication, transmission and acknowledgement interpretation |
| Delivery engine | chnotify | Routing snapshot, durable outbox, limits, scheduling, retry state and recovery |

A channel is a configured adapter instance plus endpoint, codec/profile, authentication, timeouts, acknowledgement policy, and retry policy. Multiple channels may use the same adapter.

Use the external schema as a pinned dependency or vendored conformance contract. Do not fetch schemas on the delivery path or turn the daemon into a general schema engine. The specification repository has not been named or created by this proposal.

The schema is independent of JSON. A schema may describe string fields and their allowed values in prose, a machine-readable model, and fixtures, while supporting several encodings.

## 3. Flow and acceptance

~~~mermaid
flowchart TD
    P["Long-running process"] --> C["Sender or SDK"]
    C --> I["Local submission socket"]
    I --> O["Durable outbox"]
    O -->|"Accepted locally"| C
    O --> D["Delivery scheduler"]
    D --> S["Socket adapter"]
    D --> H["HTTP adapter"]
    S --> SR["Socket receiver"]
    H --> HR["Web endpoint"]
~~~

For each new submission:

1. Receive a size-bounded request with a producer-generated event ID.
2. Decode and validate the supported schema profile.
3. Resolve all configured destinations and preflight their codecs.
4. Atomically store the notification, immutable delivery payloads, routing snapshot, and one pending delivery per destination.
5. Reply with durable local acceptance only after the transaction commits with the promised durability.
6. Schedule destinations independently.

A successful `chnotify send` means **the local daemon owns a durable pending notification**. It does not mean every receiver has accepted it. The producer waits for bounded local admission and disk persistence, never for remote channel availability.

If the daemon is unavailable, the request is invalid, the queue is full, or persistence fails, the sender returns a clear failure. A local timeout is an unknown outcome: acceptance might have happened before the reply was lost.

The CLI creates one event ID per invocation and reuses it during any internal submission retry. It also accepts `--id` so a producer can retry the same logical notification across invocations. An SDK retains the ID until acceptance is known.

Deduplicate submissions by authenticated local producer identity and event ID. Identical retries return the original acceptance result; the same ID with different normalized content fails. Resolve a duplicate before re-evaluating current routing, so a configuration change cannot expand an old submission.

Keep completion tombstones for a configured deduplication window and retain IDs for all pending or failed records. A duplicate after that window can be admitted again; document this limit instead of promising indefinite deduplication.

## 4. Local socket and systemd relationship

Use a Unix stream socket for the reliable submission API. Frame each request and reply with a four-byte network-order length, followed by a versioned body. Bound the length before allocation. The request body carries the event ID, schema/profile reference, input codec and payload; the reply carries the same ID and a structured acceptance or error result.

The exact body grammar must be frozen alongside its fixtures before the sender and daemon are implemented. It can use a compact binary envelope without making JSON mandatory. This is a local transport envelope, separate from the process-notification schema.

Use a dedicated `CHNOTIFY_SOCKET` setting. Authenticate local peers using socket ownership/permissions and platform peer credentials; assign an explicit owner for administrative operations. Restrict queued files to the daemon owner.

systemd notifications use newline-separated variable assignments and a datagram transport. Their return value establishes enqueueing, not successful processing. Borrowing their field syntax does not implement the full service-manager protocol: attribution, barriers, watchdog behavior, and passed file descriptors have additional semantics. [S1]

Consequently:

- The reliable chnotify socket has its own acknowledgement contract.
- A native systemd-compatible datagram input, if added, is a separate adapter with a clearly weaker acceptance guarantee.
- A generic socket destination is not automatically the systemd manager's notification socket. Relaying a producer's messages to PID 1 needs a dedicated integration with correct attribution.

Unix datagrams preserve message boundaries; streams require framing. [S2] Length framing supports multiline YAML and JSON without guessing where a notification ends.

## 5. Schema and format bindings

Propose a deliberately small first schema profile: a flat map from field names to UTF-8 strings. The specification repository owns the final decision and the lexical rules for numeric or boolean-like values.

For that profile, `"0008"`, `"8"`, an empty string, and an absent field remain distinct. JSON/YAML emit string scalars; a codec must not infer types from appearances. If a future schema needs nested or typed data, introduce versioned bindings together.

| Codec | One-notification representation | Boundary |
|---|---|---|
| Key/value text | LF-separated `KEY=VALUE` fields | One submission frame or HTTP body |
| JSON | One object with unique keys and string values | One frame or HTTP body |
| JSONL | One compact JSON object followed by LF | Line record inside the selected transport |
| YAML | One restricted YAML 1.2 mapping document | One frame or HTTP body |
| TSV | Defined two-column field/value profile with a header | One frame or HTTP body |

For TSV, propose a `key<TAB>value` header followed by one row per field. Define the profile's escaping for backslash, tab, LF, CR, and empty values, and require a round trip. The historical TSV registration specifies a header and consistent columns and excludes literal tabs in a field; escaping and empty-value handling therefore need an explicit profile. [S3]

For YAML, reject duplicate keys, aliases, custom tags, nested collections, and unsupported scalar forms under this profile. Quote generated string values. YAML has a broader data model than the proposed notification profile. [S4]

JSONL reuses the JSON codec; it is not a separate semantic schema. Its convention is one JSON value per line. [S5] The HTTP adapter sends one event per request initially, including for JSONL; batch delivery can wait.

The key/value binding splits on the first equals sign. Reject values that cannot be represented by its agreed grammar, rather than inventing an escape convention and calling it systemd-compatible.

Input format and each output format are independently selected. Detect unsupported conversions before acceptance. Preserve permitted extension fields and reject duplicate fields rather than silently choosing a value. Exact key rules, required fields and codec fixtures belong in the schema repository.

## 6. Adapter interface and extensibility

Begin with compiled adapters behind a small internal Rust interface. Optional Cargo features can remove unused transports or codecs from a build.

The conceptual operation is:

~~~text
deliver(immutable_body, delivery_metadata, deadline)
  -> accepted(acknowledgement_level)
   | retryable(retry_after, reason)
   | blocked(reason)
   | permanently_failed(reason)
~~~

Delivery metadata includes a stable delivery ID and content type. The adapter owns I/O, authentication, and response interpretation. The delivery engine owns retries and state transitions. Adapters must not retry secretly or create a second outbox.

This supports additional transports without duplicating codec or queue logic. A notification can be encoded once per distinct codec/profile and shared across matching destination records.

Runtime-loaded Rust libraries are unnecessary for v0.1; Rust's native ABI has no stability guarantee. [S6] If independently installed plugins become necessary, design a separately versioned external-process protocol at that point and measure its costs.

## 7. Initial channels

### Unix socket

Offer a framed stream mode with an explicit receiver-ACK protocol. The frame envelope carries delivery identity and format metadata, followed by the selected encoded body. The test receiver acknowledges the same delivery ID only after meeting its acceptance contract.

Also permit an explicitly selected raw mode for existing receivers. Its framing must be configured: a length prefix for arbitrary multiline bodies, a line record only for a suitable codec, or a datagram boundary for datagram sockets.

A completed raw write means transport submission. It cannot establish application processing or persistence. In raw mode, queue completion is recorded at that weaker level and does not imply receiver-confirmed delivery.

Do not send a custom length prefix to a receiver that expects native datagrams. Socket type, framing, and acknowledgement are independent settings validated together.

### HTTP/HTTPS

Use one POST per notification, a content type matching its codec, bounded connect/request deadlines and a bounded response read. Reuse connections while active.

Support configurable no-auth, bearer token, basic authentication and custom secret headers. Read secret references from files or environment; keep secrets out of payloads and persisted route descriptions. A changed environment requires daemon restart; file-based credentials may be refreshed explicitly.

Use TLS verification for HTTPS. Disable automatic redirects by default so a URL change cannot silently change the destination or credential recipient. Plain HTTP is useful for loopback testing and explicitly configured trusted local endpoints.

Configure which success responses constitute acceptance. The fixture endpoint returns 204 only after recording the event. A 2xx response means whatever the endpoint contract promises; it is not universal proof of durable storage.

A receiver supporting durable replay must deduplicate the stable delivery ID, carried in an agreed idempotency header. The presence of that header alone does not make a generic endpoint idempotent. RFC 9110 requires knowledge of safe replay before retrying a non-idempotent request after an uncertain outcome. [S7]

Require an explicit replay contract for automatic ambiguous retries: receiver deduplication or a receiver where repeated notifications are safe. Otherwise retain an uncertain outcome for inspection.

## 8. Durable outbox and recovery

Use embedded SQLite as the first storage implementation: one serialized writer, a bounded cache, indexed pending deliveries and no external database process. A custom spool would need its own tested atomic updates, crash recovery and compaction.

Use WAL with `synchronous=FULL` for the durable-acceptance mode. SQLite documents that WAL with `NORMAL` can lose committed transactions after power failure. Durability also depends on the filesystem and storage honoring flushes. [S8] Keep storage on an appropriate local persistent filesystem.

Store:

- Stable event identity, local producer identity, normalized content digest and acceptance order.
- Immutable encoded payloads and their profile/codec versions.
- A destination snapshot for each selected channel.
- Per-destination state, attempts, next retry, and bounded diagnostic information.
- Completion tombstones for the advertised admission-deduplication window.

Separate credentials from those snapshots. Re-resolve their references when explicitly refreshed; never silently retarget queued work to a new URL or socket because configuration changed. Removing a required adapter or credential leaves affected work inspectable and blocked.

Minimal per-destination states are pending, in-flight, retry-wait, blocked, delivered and permanently-failed. On restart, unresolved in-flight work is replay-eligible with its original identity only under a configured replay-safe contract. Otherwise persist blocked(reason=uncertain), including across further restarts, until explicitly resolved.

Persist each destination's completion independently. A socket success plus HTTP failure must retry only HTTP after a normal restart. A crash between receiver acceptance and recording local completion may repeat that destination, so its receiver must deduplicate.

Retain payload bytes for every failed, blocked or uncertain destination so explicit retry remains possible. Reclaim an event's payload only after every destination is delivered or explicitly dropped; completed IDs remain for the deduplication window.

### Limits and backpressure

Bound message size, simultaneous local clients, admitted event count, pending payload bytes, configured destinations, in-memory work and active deliveries. Treat tombstones and diagnostics as bounded storage too.

Admission is all-or-nothing for the selected destination set. If it cannot fit, reject the new submission before acknowledging it. Keep existing accepted work.

The example uses a 64 KiB notification limit, 64 MiB queued-payload limit, 65,536 pending-event limit and eight concurrent deliveries. These are proposed defaults for measurement, not measured capacity claims.

A payload limit is not a disk quota: indexes, destination records, tombstones, the database and WAL add overhead. Enforce a separate total storage budget with completion/checkpoint headroom, bounded readers and work-driven WAL checkpoints. Test physical growth and disk-full behavior.

## 9. Ordering and retry policy

For v0.1, allow one in-flight notification per channel and preserve acceptance order on that channel. A retrying head delays later work for the same channel; unrelated channels continue. This deliberately simple FIFO policy can be revisited if real workloads need independent per-producer lanes.

Permanent message rejection is retained as failed and releases the next queued item. Credential or endpoint-configuration failures block the channel until corrected. Explicit retry of an old failed message may arrive after newer messages and must not be advertised as strict ordering.

Use capped exponential backoff with jitter. Honor a valid receiver Retry-After deadline even when it exceeds the local backoff cap. Run one earliest-due timer, not a periodic queue scan.

Suggested HTTP classifications, subject to the receiver's contract:

| Outcome | Action |
|---|---|
| Agreed success response | Persist destination completion |
| Temporary connection failure, 408, 429, transient 5xx | Retry under the configured replay contract |
| 401 or 403 | Block for credential/authorization repair |
| Invalid payload or unsupported content type | Retain as permanently failed |
| Timeout after possible acceptance | Replay the same delivery ID only under a replay-safe contract; otherwise retain uncertainty |
| Redirect | Block for explicit endpoint review |

Delivery is persistent retry with at-least-once attempts where replay is enabled. Exactly-once external effects require receiver cooperation and are not a generic adapter guarantee. Delivery can remain blocked indefinitely if the endpoint never recovers.

## 10. Resource use and operation

Use a current-thread asynchronous runtime for socket and HTTP I/O, plus a small bounded path for blocking persistence work. Tokio supports a current-thread executor; its default multithreaded runtime otherwise starts workers according to available CPU cores. [S9]

A single database worker is a reasonable initial choice. Do not perform blocking disk writes or fsync on the I/O thread. Account separately for DNS, TLS, database and library-created threads when measuring the build.

The idle contract is:

- Sleep while no input, delivery deadline or control signal is pending.
- Perform no application polling loop, process scan, periodic heartbeat, or background network traffic.
- Schedule retry work only for pending deliveries.
- Bound connection pools and close idle resources under an explicit policy.
- Load only configured adapters and keep the backlog on disk.
- Log bounded state changes and failures rather than every successful event by default.

Measure idle RSS/PSS, CPU time, wakeups, open files, binary size, submission latency and recovery behavior. Report socket-only, HTTPS-initialized and backlog cases separately. No memory figure or literal zero-footprint claim is established until a real implementation is benchmarked.

On shutdown, stop admission, finish or reject already-received admissions, record completed deliveries and exit within a bounded deadline. Accepted pending work remains on disk. Avoid waiting indefinitely for an unavailable web endpoint.

Provide `check` for configuration validation and local `outbox list/retry/drop` commands for delivery repair. Administrative deletion is explicit. These controls operate on distribution state.

## 11. Configuration model

Use TOML for daemon configuration; configuration format and payload format are independent. See [the proposed example](../examples/chnotify.toml).

Validate the entire configuration before listening: unique channel IDs, required secrets, known codecs, compatible framing/ACK combinations, positive resource bounds, and at least one enabled destination. Paths in the example resolve relative to the configuration file, not the caller's working directory.

The initial configuration sends each accepted notification to every enabled channel. Named routing can be added later if a concrete use case needs it. Producers cannot supply arbitrary URLs or remote credentials.

Existing queued work uses its original destination and format snapshot after restart. Configuring a new channel affects new submissions only.

## 12. Decisions to finish before implementation

The architectural defaults above are the proposal. Three concrete contracts must be finalized as part of the first implementation milestone:

1. The dedicated schema repository, first profile/version, lexical value rules, TSV escaping profile and shared fixtures.
2. The exact local submission and acknowledged-socket wire grammars, with positive and negative byte fixtures.
3. The receiver replay/ACK contracts and the resource budgets measured on the first supported Linux targets.

These contracts keep schema, format and delivery separate while making the v0.1 implementation testable. The [acceptance plan](acceptance.md) defines the required behaviors.

## Primary references

These sources establish existing protocol and library behavior. Defaults, boundaries and proposed limits above are design recommendations.

- [S1: systemd sd_notify](https://www.freedesktop.org/software/systemd/man/latest/sd_notify.html)
- [S2: Linux unix(7)](https://man7.org/linux/man-pages/man7/unix.7.html)
- [S3: IANA TSV media type registration](https://www.iana.org/assignments/media-types/text/tab-separated-values)
- [S4: YAML 1.2.2 specification](https://yaml.org/spec/1.2.2/)
- [S5: JSON Lines format](https://jsonlines.org/)
- [S6: Rust Reference, external ABIs](https://doc.rust-lang.org/reference/items/external-blocks.html#abi)
- [S7: RFC 9110, idempotent methods](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2)
- [S8: SQLite synchronous modes](https://sqlite.org/pragma.html#pragma_synchronous)
- [S9: Tokio runtime configurations](https://docs.rs/tokio/latest/tokio/runtime/index.html)
