# Implementation and acceptance plan

Status: proposed behavior; no implementation or runtime measurements yet.

[RFC 0001](rfcs/0001-chnotify-protocol.md) is the proposed normative contract. The [portable fixtures](../spec/fixtures/v1/README.md) supply representative codec/wire values and reducer scenarios; passing those examples alone is not full implementation conformance. See also the [runner integration qualification](integrations/runners-and-monitors.md).

## Milestones

| Milestone | Deliverable | Gate |
|---|---|---|
| 1. Freeze contracts | External schema profile, codec fixtures, submission and receiver ACK grammars | Fields and message boundaries round-trip without interpretation changes |
| 2. Durable local path | One Rust binary, config validation, sender, Unix ingress, transactional outbox | Durable ACK and admission deduplication survive crash scenarios |
| 3. Distribution | Unix socket, HTTP and file adapters; auth; TSV, YAML, JSON, JSONL and TOML | Format/channel matrix and partial-success recovery pass |
| 4. Reliability and footprint | Retry classification, backpressure, recovery, service lifecycle and measurements | Fault tests and documented idle/backlog benchmarks pass |

Ship compiled adapters first. Add extra plugin machinery or transports only when an actual consumer requires them.

## Default end-to-end fixture

Run one daemon, one Unix socket receiver, one HTTP receiver on loopback, and one dedicated log-file destination. Bind a temporary socket path, an ephemeral HTTP port, and an isolated directory for the file channel. The HTTP fixture supports no-auth, bearer, basic and a configured secret header with test-only credentials.

The reliable socket fixture implements the versioned ACK protocol. Both reliable fixtures persist or otherwise deliberately model their promised acceptance and deduplicate delivery IDs. Also test a raw socket fixture whose weaker completion level is expected.

All three channels receive each submitted notification. Any one can be unavailable while the others continue. The file channel completes after a durable append and explicitly permits identifiable duplicate records after a crash.

The sample configuration shows JSON on HTTP, TSV on the socket, and TOML inside RFC 5424-formatted file records. Parameterize it so the same fixture exercises all 15 combinations:

| Payload format | Unix socket | HTTP | Log file |
|---|---|---|---|
| TSV profile | Required | Required | Required |
| YAML profile | Required | Required | Required |
| JSON | Required | Required | Required |
| JSONL | Required | Required | Required |
| TOML | Required | Required | Required |

Repeat the five file payload cases using the optional JSONL record envelope. Decode the envelope before applying the selected payload codec.

For input decoding, test each supported format against the same logical-notification fixtures. All output codecs must preserve the agreed field/value model.

## Required fault and conformance cases

### 1. Durable admission

Terminate the daemon before commit, after commit before the acceptance reply, and immediately after the reply. Every acknowledged event survives process restart. An unacknowledged submission may be absent or already accepted; retrying the original ID resolves that uncertainty.

A process-kill test does not prove power-loss durability. Add a filesystem/VM crash fixture for that guarantee and record the assumptions about storage flushing.

### 2. Identity and deduplication

Lose the acceptance reply and resubmit the same ID. There is one admitted notification and its original destination set. Reject changed content with the same ID. Exercise retries while pending, after completion, after configuration change, and beyond the documented tombstone window.

### 3. Independent destinations

Let the socket succeed while HTTP is unavailable. Restart the daemon. The recorded socket success remains complete; only the pending HTTP delivery resumes. Reverse the roles too.

### 4. Ambiguous receiver acceptance

Let a receiver record a notification and drop its reply. Replay retains the same delivery ID and frozen bytes. A cooperating receiver suppresses the duplicate effect. A channel with no replay contract remains blocked with an uncertain outcome across restarts until explicitly resolved. Failed and uncertain deliveries retain their payload bytes so explicit retry remains possible.

### 5. Limits and failures

Fill pending-event and payload-byte limits independently. Oversize requests, full storage and unavailable persistence fail admission without a false ACK. Exercise slow local clients, slow receivers and bounded responses. Verify bounded RSS, active clients, active deliveries, database/WAL growth and room to record completions.

Permanent failures remain inspectable. An invalid event can release the next FIFO item; a credential-blocked channel pauses without tight retry loops. Failed destinations do not block unrelated channels.

### 6. HTTP and socket behavior

Test bearer/basic/custom-header authentication, absent and changed credentials, verified HTTPS, timeout, connection reset, 429 with Retry-After, temporary 5xx, invalid payload, and redirects.

Test split and coalesced stream reads, truncated frames, oversized length prefixes, disconnected receivers, datagram boundaries and ACK identity mismatch. Raw-write completion must never appear as receiver-persisted completion.

### 7. Codec boundaries

Cover empty versus missing values, Unicode, equals signs, tabs, LF/CR, backslashes, leading zeros, large numeric-looking strings, duplicate fields and permitted extension fields.

Reject unsupported YAML structures and ambiguous scalar conversions. For TOML, cover quoted literal dotted keys, multiline strings, duplicate keys, and rejection of tables, arrays, native integers, booleans and dates under the string-map profile. Verify TSV header/escape rules and JSONL's one-object-per-line profile. A format that cannot represent an admitted value causes preflight rejection, not a later silent rewrite.

### 8. Restart, shutdown and configuration

Restart with queued work, changed endpoints, removed channels and rotated secret references. Queued data retains the original destination and payload. Validate failure behavior for an unknown adapter/profile, unusable credentials and a second daemon opening the same state directory.

Shutdown finishes or rejects admissions and leaves accepted work recoverable within a bounded deadline.

### 9. Log-file records and recovery

Parse the default RFC 5424 record header and reverse the file profile's escaping; recover the original payload for every codec, including TOML. Parse JSONL envelopes independently from JSONL payloads. Verify stable timestamp/identity on replay, complete-record LF framing, and no extra physical lines from payload LF/CR, tabs, NUL, escape sequences or Unicode.

Inject short writes, failed sync, disk-full, and a crash after log synchronization but before outbox completion. Only a complete, synchronized record can complete a destination. A replay may add a complete duplicate with the same delivery ID; it must not concatenate onto a partial trailing record.

Exercise bounded incomplete-tail repair under exclusive ownership, unexpected-corruption blocking, second-writer rejection, file creation/directory synchronization, active-file size limits, and rename-and-create rotation followed by explicit reopen. Verify replay across rotation and retention of the pending event when reopen fails. Do not claim reliable copytruncate support.

### 10. Process projection, freshness and migration

Verify complete-snapshot replacement, including clearing omitted optional fields and extensions. Exercise missed sequences, duplicate event IDs, changed content under an existing ID, sequence reuse, backward lifecycle transitions, and first terminal arrival below a nonterminal high-water sequence. A previously occupied sequence remains a conflict. A terminal outcome cannot be overwritten or reopened by a later event.

Test heartbeat delivery before and after its original expiry, replay in history mode, an unleased snapshot, excessive future timestamps, a wall-clock rollback, and receiver restart without a reconstructible monotonic deadline. Duplicate receipts and backlog flush must not grant a new lease. Lost contact must not invent a terminal result.

Verify expiration of live attention and persistence/dismissal of review-scoped attention independently of outcome. Test `completed`, `needs_attention`, `deferred`, `cancelled`, `failed` and known-ended `unknown` results; a successful task or exit code alone cannot complete a run.

Exercise automatic replay-window expiry and receiver deduplication promises. Retain work as blocked rather than silently replaying past the advertised safe horizon. Verify local-source authorization separately from claimed source names, and reject missing or mismatched native HTTP receipt headers.

Capture representative runner lifecycle fixtures without changing task execution or invoking models. Keep the strict existing monitor endpoint separate until a new endpoint or complete legacy projection is qualified; verify one publisher and one sequence authority during handover.

## Performance evidence

Record build features, binary size, platform, kernel, filesystem, hardware, message size and channel count. Measure:

- Empty idle daemon: RSS/PSS, CPU time, wakeups, threads and open files.
- Idle after one authenticated HTTPS delivery and one durable log-file append.
- Steady submission and durable local acceptance latency.
- A disconnected receiver with a growing persistent backlog.
- Restart with the proposed maximum queue.
- Recovery while live producers continue submitting.

Confirm no application polling or background network traffic with an empty queue. Compare idle and backlog memory to detect accidental in-memory mirroring.

A useful initial run can use 65,536 notifications, a 64 KiB message boundary and eight delivery slots, varying one factor at a time. Publish the measured results before adopting a memory or throughput promise.

