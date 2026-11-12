# RFC 0001: Process notifications and reliable channel delivery

| Item | Value |
|---|---|
| Status | Draft 1; proposed project specification |
| Date | 2026-11-12 |
| Repository | [bpstr/chnotify](https://github.com/bpstr/chnotify) |
| Semantic profile | `process/1` |
| Stream framing | `CHN1` |
| Intended implementation | Small Rust sender and daemon; independent SDKs and receivers |
| Supersedes | Unresolved protocol details in the architecture proposal |

## Abstract

chnotify accepts a process notification, records durable responsibility for it, and distributes it to configured channels. A process can report readiness, current work, waiting, attention, heartbeat and an explicit terminal result without implementing remote credentials or delivery retries. Displays can interpret the same reports independently of the producer and transport.

This specification defines a small semantic model, six text bindings, reliable local submission, receiver acknowledgements, bounded delivery and monitor reduction rules. It is inspired by `systemd-notify` but defines an independent protocol. `taskmd-runner` and `roadmap-runner` are planned producers; `roadmap-monitor`, terminal displays and other applications are planned consumers.

**This is a proposed contract, not a description of implemented chnotify behaviour.** The current repository contains design documents. No interoperability, durability or resource benchmark is claimed by this RFC.

## 1. Requirements language and scope

Uppercase requirements words use the meanings in BCP 14, RFC 2119 and RFC 8174. Lowercase prose does not invoke those special meanings. [R1]

The protocol reports process facts. It does not execute jobs, discover processes, collect stdout, supervise or kill processes, replace application logging, or define a dashboard. A notification is a meaningful observation about an execution; a heartbeat is a bounded exception used to establish continuing contact. Per-line output and arbitrary trace events do not belong in this profile.

An implementation MUST distinguish:

1. **Execution state:** what the producer reports about its run.
2. **Freshness:** whether evidence of continuing contact is still valid.
3. **Delivery state:** whether a distributor or destination accepted responsibility.

A stale `running` report is not a failed or completed run. A delivered notification does not mean a human saw it. A healthy daemon does not establish that any producer is alive.

The semantic profile and bindings will move to a dedicated specification repository. This RFC temporarily holds that portable contract; chnotify-specific storage and adapter implementation remains in this repository. Implementations MUST pin the supported specification version and MUST NOT fetch schemas while admitting or delivering events.

## 2. Roles and layering

| Term | Meaning |
|---|---|
| Producer | Code that knows the process's actual state and creates observations |
| Source | Stable, configured logical reporter identity within an authenticated namespace |
| Run | One execution, including deliberate pauses; a new execution receives a new identity |
| Event | Immutable complete observation of one run |
| Sender | CLI or SDK that serializes and submits an event |
| Distributor | chnotify daemon that validates, queues and forwards events |
| Channel | Configured adapter instance, endpoint, codec, credentials and delivery policy |
| Receiver | Destination that accepts events under a declared receipt contract |
| Monitor | Consumer that derives current state, freshness and presentation |

The layers are semantic profile, codec, framing, transport, and delivery policy. Selecting a codec MUST NOT change field meaning. Selecting a channel MUST NOT change event identity. A log record envelope is separate from its payload codec.

The initial daemon uses one executable: `chnotify send`, `chnotify daemon` and administrative subcommands. Independent SDKs MAY send directly to a conforming endpoint; they then own the submission and retry responsibilities otherwise supplied by the daemon.

## 3. Relationship to systemd

systemd's notification protocol provides useful precedents: newline-separated assignments, startup readiness, a human-readable status, shutdown starting, and watchdog contact. `READY=1` signals readiness, `STOPPING=1` signals shutdown beginning, and `WATCHDOG=1` refreshes watchdog contact. None means successful application completion. `sd_notify()` enqueue success and systemd's processing barrier are different from a durable external receipt. [R2]

chnotify borrows these reporting concepts and provides CLI shorthands in Section 14. It uses `CHNOTIFY_SOCKET`, not `NOTIFY_SOCKET`, and its reliable local endpoint is a framed Unix **stream** socket. Native systemd uses a different transport and attribution model. `CHN1` MUST NOT be sent to a systemd notification socket.

This profile does not implement `MAINPID`, descriptor storage, `BARRIER`, service-manager access changes, or watchdog-triggered termination. A future native-systemd bridge needs its own credentials, process-attribution and file-descriptor contract. A raw socket adapter alone does not supply that compatibility.

## 4. Data model and lexical rules

An event is an unordered map of unique field names to Unicode strings. Every codec MUST preserve the exact decoded strings, including empty versus absent values and numeric-looking extension values. There are no nulls, arrays, objects, native booleans, native numbers or native date-time values in `process/1`.

All payloads MUST be valid UTF-8 without a byte-order mark. Values contain Unicode scalar values, never unpaired surrogates or U+0000. Unicode normalization MUST NOT be applied. A field name matches `[A-Z][A-Z0-9_]{0,63}`. Names are case-sensitive.

The profile limits are 64 fields, 8,192 UTF-8 octets per decoded value, and 65,536 octets for the sum of decoded name and value lengths. Each encoded payload also has a 65,536-octet ceiling. Codec expansion can exceed the ceiling even for a valid decoded map; admission MUST reject a route set that cannot encode the event within every selected channel's bound. It MUST NOT silently drop fields or switch codecs.

These lexical types apply where named below:

| Type | Rule |
|---|---|
| `token` | `[A-Za-z0-9][A-Za-z0-9._:/-]{0,127}`; opaque and case-sensitive |
| `uuid` | Lowercase canonical UUID version 4, including the RFC variant; 36 ASCII characters |
| `uint` | `0` or a decimal integer without leading zeros, sign or whitespace; at most `9007199254740991` |
| `positive` | `uint` greater than zero |
| `time` | Valid Gregorian UTC `YYYY-MM-DDTHH:mm:ss.sssZ`; exactly three fractional digits; no leap-second spelling |
| `text` | UTF-8 without U+0000–U+001F or U+007F–U+009F; bounded as specified |

The decimal-string bound permits straightforward JavaScript interoperability without making the wire representation numeric. UUID generation follows RFC 9562; the fixed timestamp spelling is a restricted RFC 3339 profile. Timestamps MUST NOT establish event ordering. [R3] [R4] [R5]

### 4.1 Required fields

| Field | Type / values | Meaning |
|---|---|---|
| `VERSION` | Literal `1` | Semantic version, independent of framing version |
| `EVENT_ID` | `uuid` | Stable identifier for this immutable event |
| `SOURCE_ID` | `token` | Stable logical reporter; authority comes from authentication, not this string |
| `RUN_ID` | `token` | Execution identifier; UUID generation is recommended, UUID syntax is not required |
| `SEQ` | `positive` | Strictly increasing sequence allocated by the run owner |
| `OBSERVED_AT` | `time` | Time the producer observed this state; never the later delivery time |
| `KIND` | `snapshot`, `heartbeat`, `terminal` | Reporting purpose, with constraints below |
| `STATE` | `starting`, `running`, `stopping`, `finished` | Declared execution lifecycle |
| `ACTIVITY` | `working`, `idle`, `waiting`, `paused` | Current activity independently of lifecycle |
| `ATTENTION` | `none`, `input`, `blocker`, `review`, `deferred`, `quota`, `capacity`, `recovery_failed` | One summary attention category |
| `STATUS` | `text`, at most 1,024 UTF-8 octets; empty allowed | Human-readable description, never parsed to infer machine state |

`SOURCE_ID` SHOULD survive process restarts. `RUN_ID` MUST change for a new execution; an OS PID, wall-clock timestamp or source name alone is insufficient. Existing opaque 24-character runner identifiers are valid. A resumed execution MAY retain its run ID only if it retains the same ordered state and has not emitted a terminal event.

Exactly one logical publisher MUST allocate `SEQ` for each authenticated `(SOURCE_ID, RUN_ID)`. Concurrent workers either submit observations through that publisher or use distinct child runs. The allocator MUST persist enough state to avoid reuse after recovery. Missing sequence numbers are allowed; reuse and wraparound are forbidden. Independent CLI invocations MUST NOT each invent the next sequence.

### 4.2 Optional fields

| Field | Type / restriction | Meaning |
|---|---|---|
| `RESULT` | `completed`, `failed`, `cancelled`, `deferred`, `needs_attention`, `unknown` | Explicit terminal outcome; only on `terminal` |
| `LEASE_MS` | `positive`, 1,024–256,000 | Bounded freshness claim; forbidden on terminal events |
| `ATTENTION_ID` | `token` | Stable identity of the currently summarized attention episode |
| `ATTENTION_SCOPE` | `live`, `review` | Live intervention or retained finished-work review |
| `REASON_CODE` | `[a-z][a-z0-9_]{0,63}` | Machine-readable reason whose detailed vocabulary belongs to the producer profile |
| `TASK_ID` | `token` | Current work item, if any; not an identity for the whole run |
| `PARENT_RUN_ID` | `token` | Parent execution in the same source namespace; cannot equal `RUN_ID` |
| `PROGRESS_CURRENT` | `uint` | Completed units for the current declared progress scope |
| `PROGRESS_TOTAL` | `positive` | Known total; requires current and current must not exceed total |
| `PROGRESS_UNIT` | `text`, at most 64 UTF-8 octets, nonempty | Unit such as `tasks`, `batches` or `bytes`; requires current |
| `PROGRESS_SCOPE` | `token` | Scope whose counters are being reported; requires current |
| `EXIT_CODE` | `uint`, 0–255 | Optional process exit status on terminal; does not determine `RESULT` |
| `PID` | `positive` | Diagnostic process identifier, never identity or proof of life |
| `HOST` | `text`, at most 255 UTF-8 octets | Diagnostic host label; not an authentication assertion |

Progress is optional. Unknown totals are omitted, not zero. A producer SHOULD retain counter meaning within a scope and change `PROGRESS_SCOPE` when starting a different unit of work. A monitor MUST NOT infer completion from a fraction or exit code alone.

If `ATTENTION=none`, both attention identity and scope MUST be absent. Otherwise both MUST be present. `ATTENTION_SCOPE=review` requires `ATTENTION=review`. A terminal event permits only `ATTENTION=none` or `ATTENTION=review` with scope `review`. Live scope requires a nonterminal event and a valid `LEASE_MS`.

Nonterminal review may use either scope: `live` for a running process awaiting review, `review` for a durable work-review request. A review-scoped summary remains reviewable until a later applied snapshot clears/replaces it or the display records a local dismissal. A terminal review is never cleared by reopening the run.

### 4.3 Extensions and complete snapshots

Unknown core names MUST be rejected. Extensions match `X_<NAMESPACE>_<NAME>`, with uppercase alphanumeric namespace and uppercase alphanumeric/underscore name, inside the same 64-character name limit. Namespace ownership and meaning MUST be documented by the extension author. Unknown extensions MUST be preserved by distributors and MAY be ignored by monitors. They MUST NOT redefine core fields or silently introduce a required capability.

**Every event is a complete snapshot**, including a heartbeat. Optional fields and extensions omitted from a newer applied event are cleared from the derived current state. They are not inherited from a previous report. Consumers MUST NOT treat events as patches. This permits useful recovery after sequence gaps without adding a snapshot-fetch protocol to the small daemon.

The core carries one attention summary, not a complete queue, a list of every blocker, or a detailed execution history. A future runner profile must specify those richer requirements. Embedding arbitrary application state in `STATUS` does not establish an interoperable extension.

## 5. Lifecycle and producer rules

| Kind | Required state and behaviour |
|---|---|
| `snapshot` | Nonterminal state; `RESULT` and `EXIT_CODE` absent; lease optional |
| `heartbeat` | Same complete nonterminal snapshot; positive `LEASE_MS` required; no result or exit code |
| `terminal` | `STATE=finished`, `ACTIVITY=idle`, explicit `RESULT`, no lease; attention constrained by Section 4 |

The valid lifecycle is `starting` → `running` → `stopping` → `finished`, with forward skips allowed. Starting or running may therefore finish directly. A run MUST NOT move backward. Missing earlier observations do not prevent a receiver from accepting a first observation in a later state.

`running` includes useful work, waiting, pauses and recovery. `stopping` means shutdown has begun; it does not certify shutdown completion. `finished` means this execution has ended according to the authoritative producer or an explicitly configured supervisor. `RESULT=unknown` represents known termination with unknown outcome; lost contact alone MUST NOT generate it.

`completed` means the producer's intended work completed. `needs_attention` means the execution ended with unresolved work requiring a decision. `deferred` means it ended with work deliberately postponed. Neither is an active waiting process; starting later requires a new run. A nonterminal blocked task or intermediate failure MUST NOT automatically become a failed run.

Producers SHOULD emit on meaningful state, activity, attention and coarse progress changes, coalescing rapid intermediate progress before submission. Accepted events MUST NOT be coalesced away silently. A typical producer can use a 16-second heartbeat interval and a 64-second lease; these are example policies, not mandatory timing. The existing runners use different intervals, recorded in the integration plan.

Heartbeat generation belongs to the producer or a separately specified supervisor that can observe it. The distributor MUST NOT manufacture heartbeats, infer producer health from its own timer, or refresh `OBSERVED_AT`. A one-shot CLI invocation cannot promise continuing contact after it exits.

After a terminal event, the producer MUST NOT create another event for that run. It MAY retransmit the identical terminal event. A new outcome, resumed job or new batch execution uses a new run ID; corrections require a future explicit correction protocol.

## 6. Monitor projection, ordering and finality

Receiver state is namespaced by authenticated reporting principal, source and run. A claimed `SOURCE_ID` MUST be authorized for that principal. `EVENT_ID` deduplication is scoped by principal and source, and includes the run in the compared content.

A conforming state consumer MUST apply this algorithm:

1. Validate authentication, profile, codec and record constraints. An invalid event cannot affect state.
2. Look up its event ID. Identical content is a duplicate: retain the original acceptance facts, return `duplicate=1`, and do not reapply state or freshness. Successful replies always echo the current request's delivery identity, including when another delivery ID previously carried this event. Changed content under the same ID is `id-conflict`.
3. If this sequence is already occupied by a different event, record `sequence-conflict` and do not project it. Sequence-identity indexes must survive for the supported replay window.
4. A first terminal event freezes the terminal snapshot and records `terminal_seq`. It may be applied even below the nonterminal high-water sequence if that sequence was previously unseen; record the contradictory higher nonterminal observation as a protocol fault. Preserve `highwater = max(highwater, terminal_seq)`.
5. After finalization, nonterminal events cannot alter the terminal projection or restore freshness. A new terminal event is a protocol conflict, not an outcome update. Identical retransmission remains a duplicate.
6. Otherwise, a nonterminal event applies only above the high-water sequence and without a backward lifecycle transition. A backward transition is a protocol conflict. Older events may be retained as history but cannot restore fields or contact.
7. Replace the current snapshot completely, then compute its freshness under Section 7. Sequence gaps may be exposed as diagnostics; a complete later snapshot does not wait for missing events.

`applied=false` means a valid event did not change the projection; it does not automatically mean delivery failed. A native receiver MUST persist accepted identity with its inbox record or projection transaction so a crash/replay cannot duplicate that acceptance. It MAY project a durable inbox later; its receipt promises inbox acceptance, not completed projection. Separate actions such as email or push delivery need their own transactional scheduling and idempotency contract; this receipt does not promise exactly-once downstream side effects.

Terminal state and event/sequence deduplication metadata MUST be retained for at least the agreed maximum replay horizon. If that cannot be guaranteed, the receiver cannot advertise deduplicated replay for that horizon. After archival expiry, reconstruction without the old history must be identified as incomplete, not presented as proof a historical run is active.

## 7. Freshness, replay and attention

Freshness is `fresh`, `stale`, `unknown` or `ended`. Terminal state is `ended`. A snapshot without a lease is `unknown`; it does not inherit or renew the previous lease. A valid lease whose remaining lifetime is zero is `stale`.

Only an applied nonterminal snapshot/heartbeat delivered in `mode=observation` is eligible to establish fresh contact. An applied nonterminal event in `mode=history` has unknown freshness, regardless of its lease; it does not inherit a prior live deadline. Ordinary retry is not automatically history; its original observation deadline still applies. Buffered observations MUST NOT receive a new timestamp when flushed.

The receiver MUST have operator-established confidence in the producer/receiver clock relationship to assert freshness. Authentication alone does not establish clock accuracy. This profile allows at most 8,192 ms of future timestamp tolerance; a timestamp farther in the future makes freshness unknown while the otherwise valid state may still be retained. A deployment requiring tighter evidence SHOULD configure a smaller tolerance. Clock uncertainty is not a terminal failure.

At the first accepted receipt of an eligible event, capture wall-clock `received_at` and monotonic `received_mono`, then compute:

~~~text
remaining_ms = max(0, min(LEASE_MS, OBSERVED_AT + LEASE_MS - received_at))
deadline_mono = received_mono + remaining_ms
~~~

The receiver MUST use that monotonic deadline for expiry. Duplicate delivery MUST NOT recompute it. A newer sequence with old observation time can update last-known state while remaining stale. Tolerated future clock error cannot extend contact beyond one lease from first receipt.

On a clock discontinuity, loss of clock confidence, or restart where the deadline cannot be safely reconstructed without extending the original lease, the receiver MUST mark freshness unknown until new qualifying evidence arrives. It MUST NOT revive an expired observation because the wall clock moved backward. Implementations MAY reconstruct a remaining deadline only using persisted expiry and trusted current time without extending it.

Live attention is actionable only while the associated snapshot is fresh. On stale/unknown contact, retain it as **last reported attention**, not a current request from a known-live process. Review-scoped attention remains reviewable independently of freshness; a terminal review may persist until a user or monitor workflow dismisses it. Such dismissal is keyed to the same run and `ATTENTION_ID` and MUST survive routine repeated snapshots of that episode. A genuinely new episode uses a new attention ID. Dismissal does not change the immutable producer event or outcome.

A display SHOULD show state, freshness, age and attention distinctly. For example: “Running · contact stale · last reported blocker: dependency unavailable.” It MUST NOT translate stale contact into “failed,” or hide a recorded terminal outcome when its heartbeat window passes.

## 8. Codec bindings

All bindings encode the same map. Duplicate decoded field names, trailing documents, unsupported values and oversized input MUST be rejected. Input is never format-sniffed. The selected codec is explicit and any transport declaration must agree with it.

| Codec ID | Binding | HTTP content type |
|---|---|---|
| `kv` | LF-separated `NAME=VALUE` | `text/plain; charset=utf-8` |
| `json` | One JSON object of string values | `application/json` |
| `jsonl` | One compact JSON object and one LF | `application/x-ndjson` (project convention) |
| `yaml` | One restricted YAML 1.2 mapping | `application/yaml` |
| `tsv` | Two-column escaped field/value document | `text/tab-separated-values; charset=utf-8` |
| `toml` | One root-level TOML string map | `application/toml` |

### 8.1 Key/value

A document consists of one or more `NAME=VALUE` lines, each terminated by LF, including the last. Split each line on its first `=`. No spaces are trimmed; quotes and backslashes in values are literal characters. Blank lines, comment lines and duplicate fields are invalid. CR and LF in decoded values are not representable; a KV output channel MUST reject such a map at admission. An empty string is `NAME=` followed by LF. This profile uses no invented systemd escape syntax.

### 8.2 JSON and JSONL

JSON follows RFC 8259 with a stricter unique-key, string-value map. JSONL is exactly one compact such object plus one LF, without a blank line or additional object. Escaped newlines inside a string are allowed. Arrays, numbers, booleans and null are invalid even when they appear to represent a known field's type. [R5]

### 8.3 YAML

Use YAML 1.2.2, exactly one mapping document with scalar string keys and values. Reject aliases, anchors, merge keys, explicit tags, directives, nested collections, duplicate keys and implicit non-string values. Generated keys and values SHOULD be double quoted; generated values MUST be strings. Optional document start/end markers are allowed, but a second document is not. A safe parser and explicit scalar validation are required; default library coercions must not change the data model. [R6]

### 8.4 TOML

Accept TOML 1.1 root-level string-valued assignments. Reject tables, inline tables, arrays, native date/time values, booleans and numbers. Dotted keys that introduce structure are invalid. Quoted literal keys are allowed if the decoded key satisfies the field-name grammar. Emit the common TOML 1.0-compatible string subset so simple implementations can interoperate; fixtures do not require the newer syntax. Multiline strings remain subject to decoded field constraints. [R7]

### 8.5 TSV

The first line is exactly `key<TAB>value<LF>`. Each following line contains the field name, one literal TAB, the escaped value, and LF. At least one field line is required. Every physical line therefore has exactly two columns. Order is not significant.

Escape backslash as `\\`, TAB as `\t`, LF as `\n`, and CR as `\r`. Encode an empty value as `\e` alone. A literal backslash followed by `e` is encoded as `\\e`. No other escape is valid; `\e` inside a longer value is invalid. Unescaped CR, an extra TAB, and an empty physical value column are invalid. This explicitly named project binding supplies escaping and empty-value semantics beyond the basic TSV registration. [R8]

### 8.6 Equality and immutable encoding

Semantic equality compares decoded maps, not whitespace or field order. For a stable content digest, create canonical bytes as follows:

1. ASCII `CHNOTIFY-MAP/1` followed by LF.
2. Sort names by unsigned ASCII byte order.
3. For each field append its name length as unsigned 16-bit big-endian, its UTF-8 name, its value length as unsigned 32-bit big-endian, and its UTF-8 value.

Compute SHA-256 over ASCII profile, one zero byte, ASCII admission mode, one zero byte, and those canonical bytes. The zero bytes here are binary separators, not payload field values. This digest covers all known and extension fields. A changed codec alone does not change equality. Changing `mode` on a retry does.

The daemon MUST pre-encode and freeze each distinct output representation and file record before durable acceptance. Delivery retries reuse those exact bytes, event IDs and delivery IDs. New software or codec versions MUST NOT silently regenerate queued payloads.

## 9. Reliable stream framing: CHN1

The reliable local Unix stream and acknowledged socket destination use the same framing. Streams can split or coalesce reads arbitrarily; callers MUST read exact lengths and MUST NOT treat a read call as a message boundary.

~~~text
frame = body_length:u32be || body
body  = magic:4 || type:u8 || metadata_length:u16be || metadata || payload
magic = ASCII "CHN1"
~~~

`body_length` excludes its own four octets. It MUST be 7–131,072. `metadata_length` MUST be 0–8,192 and no greater than `body_length - 7`. The remaining octets are the payload, at most 65,536. Receivers MUST bound the length before allocation. Reply bodies have no payload.

Metadata consists of LF-terminated ASCII `name=value` lines. Names match `[a-z][a-z0-9-]{0,31}`; values are printable ASCII U+0020–U+007E, without trimming. Split at the first equals sign. Duplicates, blank lines and unterminated final lines are invalid. Metadata order is irrelevant. Unknown names prefixed `x-` may be ignored; other unknown names are rejected. Required names and enum values below are exact.

| Type byte | Name | Required metadata | Optional metadata |
|---|---|---|---|
| 1 | `SUBMIT` | `event-id`, `profile`, `codec`, `mode` | none |
| 2 | `ACCEPTED` | `event-id`, `accepted-at`, `dedup-until`, `duplicate` | none |
| 3 | `ERROR` | `code` | `event-id`, `delivery-id`, `retry-after-ms` |
| 4 | `DELIVER` | `event-id`, `delivery-id`, `profile`, `codec`, `mode` | none |
| 5 | `DELIVERED` | `event-id`, `delivery-id`, `receipt`, `dedup-until`, `duplicate` | `applied` |

Here `profile=process/1`; codec is from Section 8; mode is `observation` or `history`. Event and delivery IDs have UUID syntax from Section 4; event ID MUST match the decoded payload. Times have the Section 4 time spelling. `duplicate` and optional `applied` are `0` or `1`. `receipt=durable` is the only native successful destination receipt in version 1. `retry-after-ms` is a `uint`. Fields not permitted for a message type are invalid, apart from `x-` extensions.

Only `SUBMIT` and `DELIVER` carry payloads; a missing payload is invalid. Reply type 2 only answers type 1; type 5 only answers type 4. A sender MUST verify IDs and expected reply type. A mismatched receipt cannot complete delivery. An `ERROR` without IDs may answer the sole outstanding request; if IDs are supplied they MUST match it. Successful replies always require the listed IDs. After a response, the connection MAY be reused, with one request outstanding per connection; version 1 does not pipeline.

A malformed length, unknown magic/type, truncated frame or metadata that cannot be parsed causes connection closure. If enough of a well-formed request is available, a receiver MAY first return `ERROR`. EOF or timeout before a trustworthy reply after any request bytes have been sent is an uncertain result, never success.

Error codes are `invalid`, `unsupported-profile`, `unsupported-codec`, `too-large`, `unauthorized`, `id-conflict`, `sequence-conflict`, `state-conflict`, `queue-full`, `storage-unavailable`, `no-route`, and `internal`. An explicit error asserts that this request acquired no new durable responsibility. A receiver MUST NOT send an error after a possibly successful commit; it must resolve the commit result or close the connection so the sender retries the same identity. Semantic projection conflicts MAY instead be durably accepted as diagnostics with `applied=0`.

The [fixture bundle](../../spec/fixtures/v1/README.md) includes complete hexadecimal frames and expected metadata. Socket-path discovery and credentials are outside the frame, not untrusted fields in it.

## 10. Local admission and responsibility

`CHNOTIFY_SOCKET` selects the daemon socket; explicit `--socket` takes precedence. Configuration supplies a default path. The daemon MUST enforce filesystem ownership/permissions and platform peer credentials and map those credentials to allowed source identities. A writable socket is not permission to impersonate every configured source.

For a new request the distributor MUST, in order:

1. Enforce frame/codec/profile/authorization and parsing/allocation bounds. New-work queue quotas are checked only after duplicate lookup.
2. Resolve admission deduplication by authenticated principal, source and event ID. Identical retries return the original acceptance; conflicting content fails before routing changes matter.
3. Select the enabled destination set and preflight every codec, route and complete file record. An empty route set is `no-route`.
4. Check new-work capacity, allocate one fresh UUIDv4 delivery ID per event/channel, and atomically commit the event, digest, admission mode, immutable encodings, route snapshot and pending destination records with the configured durability. Each delivery ID is persisted before acceptance and reused for every attempt.
5. Return `ACCEPTED` only after the commit's durability condition is met.

`accepted-at` is the original acceptance time. `dedup-until` is an inclusive lower bound on retention of the admission identity, initially no earlier than acceptance plus the configured admission-deduplication period. All pending, blocked and failed events retain their identity beyond that date while retained. Completed events keep bounded tombstones. A duplicate returns the original acceptance time and guaranteed retention date with `duplicate=1`; it does not gain new routes or consume new-work quota.

The sender waits for bounded local admission and persistence, never remote availability. A successful send transfers responsibility to the daemon; it does not guarantee eventual remote delivery if an endpoint never recovers. Queue rejection and persistence failures MUST be explicit. A timeout may mean the commit happened and its reply was lost, so a retry MUST preserve event ID, mode and map.

One event ID is created per logical observation, not per network attempt. Retry state MUST retain it. If a producer wants retry protection across separate CLI invocations, it MUST supply the same `--id`, sequence, timestamp and fields. Automatically creating a fresh timestamp during a retry is a changed event.

Automatic admission retries MUST have a persisted deadline no later than the daemon's guaranteed admission-retention duration measured from the producer's first attempt, not its latest attempt. The native default guarantee is at least 24 hours; clients may use a shorter window. Do not begin an attempt whose timeout crosses that deadline. A client without a known retention agreement MUST NOT assume indefinite safe replay. After the deadline or loss of clock confidence, retain uncertainty for explicit resolution rather than reset it and risk a new admission after the old tombstone expired.

## 11. Channel delivery profiles

### 11.1 Acknowledged Unix socket

Use `DELIVER`/`DELIVERED` over a stream. A receiver returns `receipt=durable` only after persisting its inbox/event and duplicate identity under its documented durability contract. Its response MUST echo both IDs. It MUST reject changed content for an existing delivery or event ID, and MUST deduplicate its inbox/projection acceptance within the agreed retention window. Later external effects have the separate responsibility described in Section 6.

A raw stream/datagram compatibility mode MAY be configured separately with explicit framing and weaker completion: `transport-submitted`. A successful write is not receiver persistence. Raw mode MUST NOT emit a `durable` receipt or be advertised as reliable native delivery. Length framing, LF framing and native datagram boundaries are distinct choices; multiline payloads cannot use an unescaped line boundary.

### 11.2 Native HTTP receiver

Send one POST per event, using its codec's content type and these request headers:

~~~text
Chnotify-Protocol: 1
Chnotify-Profile: process/1
Chnotify-Codec: <codec>
Chnotify-Mode: <mode>
Chnotify-Event-Id: <EVENT_ID>
Chnotify-Delivery-Id: <delivery UUID>
Idempotency-Key: <same delivery UUID>
~~~

The body is exactly one encoded event, not a JSON-only wrapper. Header names are case-insensitive as in HTTP. Literal values above are exact; placeholders select a Section 8 codec, `observation` or `history` mode, and the corresponding IDs. Header event identity MUST agree with the body. Batches and compression negotiation are outside version 1.

A native successful response is `204 No Content`, with `Chnotify-Event-Id`, `Chnotify-Delivery-Id`, `Chnotify-Receipt: durable`, `Chnotify-Dedup-Until: <time>`, and `Chnotify-Duplicate: 0|1`. Optional `Chnotify-Applied: 0|1` distinguishes projection from receipt. The response body is empty. Missing/mismatched receipt headers are not native success, even on a 2xx status.

Recommended error statuses are 400 for malformed data, 401/403 for authorization, 409 for identity/sequence conflicts, 413 for size, 415 for codec, 422 for profile/semantic rejection, 429 for capacity and 503 for temporary unavailability. A non-success response MAY carry `Chnotify-Error-Code` using Section 9 codes. Response reads and headers MUST be bounded. A timeout or reset after possible receiver acceptance is uncertain.

### 11.3 Generic HTTP endpoints and credentials

Generic webhooks MAY declare another success/status contract, including response-only acceptance. Configuration MUST name the weaker evidence level; an arbitrary 2xx MUST NOT be upgraded to durable receipt. The native endpoint profile and a generic webhook are separate configurations.

Automatic retries of uncertain POSTs require documented receiver deduplication or explicit permission for duplicate effects; an idempotency header alone does not provide it. Otherwise uncertain delivery is retained for inspection. This reflects HTTP's requirement to know replay safety for non-idempotent requests. [R9]

Support no-auth, bearer, HTTP basic or explicitly named secret headers as channel settings. Secrets MUST come from protected files or environment references, never event fields or URL userinfo. Verify TLS and use HTTPS for remote credentials. Plain HTTP is limited to explicitly configured trusted/local testing deployments. Redirects are disabled; redirects MUST NOT silently forward credentials to a new destination. Logs and receipts MUST exclude secrets.

Transport credentials authenticate the reporter, and the receiver MUST authorize its source scope. Browser login credentials, reporting tokens and daemon administrative rights are distinct. `rm_` tokens are a legacy roadmap-monitor detail, not a chnotify token syntax requirement.

### 11.4 File destination

The file adapter MUST follow the [file-channel contract](../file-channel.md): default RFC 5424 message syntax, `STRUCTURED-DATA=-`, ASCII-escaped message values and one physical LF-terminated record; or a JSONL log envelope. The inner payload can use any supported codec. This is a file profile, not a network syslog implementation. [R10]

The envelope includes `record_version=1` plus the fixed acceptance timestamp, source metadata, IDs, channel, profile, codec, mode and exact encoded payload. In JSONL, `record_version` is a JSON integer and unknown process ID may be null; those envelope types are not part of the all-string notification map. In the text MSG all values are quoted JSON string literals. Header characters/lengths follow RFC 5424.

File completion means complete append plus successful file synchronization, and directory synchronization on creation. It does not mean a display read the record. One writer, bounded partial-tail repair, explicit rename/reopen rotation and independent storage bounds are required by the file contract. Append and outbox completion are not one transaction: replay may create an identifiable duplicate with the same frozen bytes. The configuration MUST declare `allow-duplicates`.

## 12. Outbox, replay and resource bounds

The delivery engine owns retries. An adapter MUST NOT add a hidden outbox, unbounded internal retries, or a thread per event. Initially adapters are compiled behind an internal interface; optional features can remove unused codecs/transports. Runtime-loaded Rust libraries are not part of the version 1 contract.

Each channel delivery progresses through `pending`, `in-flight`, `retry-wait`, `blocked`, `delivered` or `permanently-failed`. A timeout can be represented as `blocked` with reason `uncertain` when replay safety is not established. Restart MUST recover unresolved in-flight work with the original IDs and bytes under the same replay rule.

A route snapshot includes endpoint, adapter, codec, acknowledgement policy and credential references. It MUST NOT include secret values. Configuration changes apply to new admissions; queued events MUST NOT silently move to a different URL, socket or file. Removed adapters or credentials block affected deliveries visibly.

Use one in-flight event per channel, in local acceptance order, for the initial implementation. A retry blocks that channel's later events but not other channels. A permanent event rejection is retained as failed and may release the next event. Credential/configuration failure blocks the channel. Retrying an old retained failure may arrive after newer events; consumers therefore still need Section 6 ordering. Large replay queues can delay contact, so monitors MUST use observation deadlines, not arrival time.

Retry temporary failures with capped exponential backoff and jitter; honor a valid later `Retry-After` deadline. A minimum retry delay MUST prevent busy loops. Schedule work using the next actual deadline, not periodic full-queue scans.

A `deduplicate` channel MUST have a configured maximum automatic replay age no longer than the receiver's promised duplicate-retention horizon. Measure this age from the event's original durable local acceptance, never from a retry, reconnection, configuration reload, credential refresh or daemon restart; persist its fixed deadline. Receiver identities are retained for at least the promised horizon after their first durable acceptance. A direct SDK measures its retry age from its first attempt, which cannot be later than the receiver's first acceptance. An outbox handover preserves the original deadline. A delivery attempt MUST NOT start if its bounded timeout would cross the sender's deadline. On loss of clock confidence after restart, block rather than extend the deadline.

At that age, retain the delivery as `blocked` with reason `replay-window-expired`; do not silently drop it or retry beyond the promise. Operator replay outside the window requires explicit acceptance of duplicate effects or a new verified receiver contract. Unlimited retries require unlimited retained duplicate identities and MUST NOT be inferred from a finite retention promise.

The first implementation uses SQLite WAL with `synchronous=FULL`, bounded caches and a serialized writer for local durable admission. This is an implementation choice, not a mandatory database for other distributors. Real power-loss durability also depends on filesystem/storage flush behaviour; a process-kill test alone does not prove it. [R11]

| Initial bounded setting | Proposed default / rule |
|---|---|
| Encoded payload | 65,536 octets |
| Clients | 32 |
| Local admission deadline | 2,048 ms |
| Pending events | 65,536 |
| Pending payload bytes | 64 MiB |
| Total outbox storage | 128 MiB, including DB/WAL/indexes/tombstones with completion headroom |
| Active channel deliveries | 8 total; 1 per channel |
| Retry base / cap | 256 ms / 65,536 ms; server Retry-After may extend the cap |
| Admission identity retention | At least 24 hours after acceptance; retained work remains deduplicated |
| Automatic deduplicated replay | Example: 7 days, only with a receiver promise covering that horizon |

These are design defaults, not measured capacity or footprint. Limits MUST cover failed records, tombstones and diagnostics as well as pending payloads. Admission is atomic for its full route set; overflow rejects new work without discarding accepted work. Storage exhaustion MUST leave space/policy for completion and recovery, not produce false success.

The daemon MUST use event-driven idle behaviour, bounded blocking workers for database/file I/O, bounded connection pools and no process polling or self-generated network heartbeats. It SHOULD have no application timer wakeups with an empty queue and no active deadlines. “Near zero footprint” is a target to verify with CPU, memory, wakeup and descriptor measurements, not a literal guarantee.

Shutdown stops new admission, resolves already-started admissions where possible, persists completion state and exits within a bounded deadline. Pending accepted work survives. Admin operations MUST expose blocked/failed/uncertain records, explicit retry and explicit drop; no accepted loss may be hidden by cleanup.

## 13. Security and failure considerations

Payloads, source labels, status text, file names in messages and extension fields are untrusted data. They MUST NOT become shell commands, URLs, file paths, credentials or unsanitized HTML. Channels and output paths come from authorized configuration. Displays MUST escape text and neutralize terminal controls; log envelopes MUST prevent record injection.

The daemon SHOULD run without root where possible. Restrict its socket, state directory, outbox, credential files and log destinations to their intended owners. Peer PID is diagnostic; use OS credentials and explicit authorization for source admission. Use a stable lock so two daemons cannot both own one outbox or file channel.

Decoders MUST enforce size and type limits before expensive allocation or parsing. No schema fetches, YAML object construction, extension code execution or unbounded decompression belongs on the input path. Receiver responses and error diagnostics are also bounded untrusted inputs.

Stored notifications may include sensitive project or task information. Source profiles SHOULD avoid secrets and full prompts/output, and operators must choose destination retention/access accordingly. Authentication protects authority; it does not prove the reported process state or clock is truthful. A compromised authorized producer can lie about its own run and must not gain authority over other sources.

This version provides no end-to-end signature through untrusted relays. Deployments requiring that property need an explicit later signature profile, not a signature of arbitrary codec bytes that change during conversion.

## 14. CLI and SDK contract

The minimal proposed commands are:

~~~sh
chnotify daemon --config ./examples/chnotify.toml
chnotify check --config ./examples/chnotify.toml
chnotify run-id
chnotify send --input ./examples/notification.toml --format toml
chnotify outbox list
~~~

`run-id` creates a new random UUID and performs no background monitoring. `send --input` reads exactly one complete record and MUST NOT silently override fields with timestamps or generated identities. Line-by-line pipe ingestion is a future explicit streaming mode, not a property of arbitrary TOML/YAML input.

For inline fields the CLI supplies `VERSION=1`, a fresh event UUID and current observation time only when not explicitly provided. `--source`, `--run`, `--seq`, `--id`, `--observed-at`, and `--kind` supply the corresponding core fields. Default kind is `snapshot`. `--mode observation|history` selects admission metadata for both inline and file input; its default is `observation`. Source/run may alternatively come from `CHNOTIFY_SOURCE_ID` and `CHNOTIFY_RUN_ID`; explicit flags take precedence. Sequence MUST be supplied, not guessed. State, activity, attention and status remain required; the tool MUST NOT invent them from prose.

~~~sh
# The enclosing producer owns this run ID and sequence allocation.
chnotify send --source example/migration --run "$run_id" --seq 1 \
  READY=1 ACTIVITY=working ATTENTION=none \
  STATUS="Migration started" LEASE_MS=64000

chnotify send --source example/migration --run "$run_id" --seq 2 \
  --kind terminal STATE=finished ACTIVITY=idle ATTENTION=none \
  RESULT=completed STATUS="Migration completed"
~~~

`READY=1` is an input shorthand for `STATE=running`; `STOPPING=1` for `STATE=stopping`; `WATCHDOG=1` for `KIND=heartbeat`. Only value `1` is accepted. Conflicting shorthands/explicit values are invalid. These aliases are normalized before encoding and are not extra core fields. A watchdog shorthand still requires a complete state and lease; readiness/shutdown shorthands do not establish terminal success. Other systemd assignments are not implicitly supported.

Use exit 0 for durable local acceptance, 2 for local input/configuration error, 3 for definite non-acceptance, and 4 for an uncertain admission after transmission without a trustworthy result. Diagnostics MUST explain that exit 0 is local ownership, not remote delivery. The default CLI MUST NOT return success merely because bytes were written to a socket.

An SDK SHOULD expose an explicit immutable event builder, canonical validation, sequence ownership, typed receipts and a bounded async submission API. A runner's execution SHOULD continue according to its own reporting-failure policy when reporting is unavailable. It must surface that degraded state; bounded best-effort submission does not guarantee preservation of an event that never received local acceptance.

## 15. Runner and monitor integration

The [integration plan](../integrations/runners-and-monitors.md) audits current `taskmd-runner`, `roadmap-runner` and `roadmap-monitor` behaviour with pinned source links and maps their lifecycle concepts to this profile.

The existing monitor uses a strict, nested v1 report envelope. It is **not** a `process/1` receiver. Deployment MUST use a new explicitly supported endpoint/profile or a separately named legacy projection that has all the data required by that old envelope. Sending these fields to the current endpoint is not a migration.

Preserve existing run identity and one sequence authority. Select one publisher/delivery owner per run during handover. Do not leave two reporters independently emitting snapshots or treating different outbox acknowledgements as the same promise. Detailed queue, history and multiple-attention parity requires a subsequent runner profile; the core intentionally provides a portable process summary.

## 16. Conformance and implementation gates

Conformance is claimed per role, profile, codec and channel, not by project name alone. A core-only consumer may ignore extensions but must obey lifecycle, freshness and terminal rules. A raw socket sink cannot claim native durable-receiver conformance.

The portable fixtures define representative exact values and wire bytes; they are not an exhaustive parser test suite. Before release, implementations MUST additionally pass the [acceptance plan](../acceptance.md), including:

- All five required interchange codecs across socket, HTTP and file channels, plus KV input and both file envelopes.
- Duplicate, changed-identity, sequence-collision, out-of-order and missing-sequence scenarios.
- Delayed heartbeat, history replay, clock skew/rollback, terminal finality and live/review attention expiry.
- Split/coalesced/truncated frames, mismatched ACKs, invalid lengths and decoder resource attacks.
- Crash before/after local commit, lost ACK, partial multi-channel success, receiver-accepted response loss, disk-full and uncertain recovery.
- Stable file replay, partial-tail recovery, rotation and directory persistence.
- Replay-horizon expiry, source authorization, secret handling and receipt-level reporting.
- Measured idle and disconnected-backlog resource usage on the declared supported platform.

Milestone order is: review/freeze/extract the portable protocol; implement codecs and durable local path; implement channels and failure handling; add independent receiver/SDK interoperability; integrate each runner and monitor behind explicit configuration. A Rust daemon, TypeScript SDK and shell-friendly CLI can then share the same portable fixture bundle.

## 17. Versioning, registrations and unresolved deployment choices

`VERSION=1`, `process/1`, `CHN1`, and file `record_version=1` are separate versioned contracts. Changing framing does not silently change process semantics. Unsupported major profiles/versions MUST be rejected; downgrade requires explicit configuration and a specified transformation. Optional ignorable extensions may evolve independently while preserving core meaning.

This draft requests no IANA allocation. The HTTP headers, profile identifier, JSONL media-type convention and binary magic are project conventions. RFC references establish existing standards, not endorsement of this proposal.

The following remain deployment/release choices rather than unspecified wire behaviour: the dedicated specification repository's name, the first supported Rust/OS builds and measured budgets, the native monitor endpoint location, and the richer runner-profile design. Selecting them must not reinterpret version 1 field or receipt semantics.

## References

Normative references supply language syntax or protocol vocabulary. The exact chnotify restrictions, defaults and algorithms above are this project's design choices.

- [R1: RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html), requirement words.
- [R2: systemd sd_notify](https://freedesktop.org/software/systemd/man/latest/sd_notify.html) and [systemd-notify](https://freedesktop.org/software/systemd/man/latest/systemd-notify.html), informative inspiration.
- [R3: RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html), UUIDs.
- [R4: RFC 3339](https://www.rfc-editor.org/rfc/rfc3339.html), timestamps.
- [R5: RFC 8259](https://www.rfc-editor.org/rfc/rfc8259.html), JSON.
- [R6: YAML 1.2.2](https://yaml.org/spec/1.2.2/), YAML syntax.
- [R7: TOML 1.1.0](https://toml.io/en/v1.1.0), TOML syntax and types.
- [R8: IANA TSV registration](https://www.iana.org/assignments/media-types/text/tab-separated-values), informative baseline for the project binding.
- [R9: RFC 9110, Section 9.2.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2), HTTP replay considerations.
- [R10: RFC 5424](https://www.rfc-editor.org/rfc/rfc5424.html), syslog message syntax.
- [R11: SQLite synchronous modes](https://sqlite.org/pragma.html#pragma_synchronous), informative implementation durability reference.
