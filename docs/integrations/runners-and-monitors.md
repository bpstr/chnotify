# Runner and monitor integration profile

**Status:** proposed integration specification; no chnotify integration is implemented or implied by this document.

**Applies to:** the core protocol in RFC 0001 and future integrations with `taskmd-runner`, `roadmap-runner`, `roadmap-monitor`, and other displays.

**Evidence reviewed:** 2026-11-12.

## 1. Purpose and boundaries

The runners determine what their work means. `chnotify` accepts their notifications locally and distributes them through configured channels. A monitor reduces accepted notifications into current state and renders that state. Reporting failures MUST NOT change task selection, invoke a model, or turn a successful task into a failed task.

The core protocol carries a useful common summary: execution identity, lifecycle, activity, attention, progress, and terminal result. It does not encode an entire roadmap, task database, log stream, or runner control API. A future runner profile is required for complete queue, history, and multiple-attention parity with the existing monitor UI.

This document uses MUST, MUST NOT, SHOULD, and MAY as defined by RFC 0001. Requirements describe the proposed integration. Statements explicitly labelled as existing describe the audited repositories.

## 2. Audited baseline

| Repository | Audited `main` commit | Existing responsibility |
| --- | --- | --- |
| `taskmd-runner` | [`5db4e1d70d569993a344ba1a2968004b010c3a66`](https://github.com/bpstr/taskmd-runner/tree/5db4e1d70d569993a344ba1a2968004b010c3a66) | Produces roadmap-monitor protocol v1 reports. |
| `roadmap-runner` | [`b2128487cfae7a41d7b762efb6679004ea32204f`](https://github.com/bpstr/roadmap-runner/tree/b2128487cfae7a41d7b762efb6679004ea32204f) | Produces the same v1 reports, including roadmap queues. |
| `roadmap-monitor` | [`351a1942a87d6fff710ece1a6def7eaec4ab4c15`](https://github.com/bpstr/roadmap-monitor/tree/351a1942a87d6fff710ece1a6def7eaec4ab4c15) | Validates reports, projects state, retains sparse history, and manages attention. |

Both runners already send JSON over HTTPS using a bearer write key. Their report kinds are `snapshot` and `replay`; a separate `probe` checks compatibility. The monitor uses strict object validation. Sending a chnotify record, or simply adding chnotify fields to a legacy envelope, is not compatible with that endpoint. See [the existing schema][monitor-schema] and [integration documentation][integration-docs].

The legacy stream key is `(installation_id, source_run_id)`. Credentials select the installation. Both runner outboxes use 24-character lowercase hexadecimal run identifiers; the monitor accepts opaque identifiers up to 128 characters. Report IDs are UUIDs, and report sequences are positive JavaScript-safe integers. See [the Taskmd reporter][taskmd-reporter] and [Roadmap reporter][roadmap-reporter].

## 3. Identity and publication ownership

Each integration MUST configure a stable `SOURCE_ID` for its authorized reporting source and preserve the native execution identifier as `RUN_ID` where possible. Restarting a completed invocation MUST create a new run. A PID, task ID, working directory, or process start timestamp alone MUST NOT serve as the run identity.

There MUST be one authoritative publisher and sequence allocator for each `(SOURCE_ID, RUN_ID)`. The publisher assigns a fresh UUIDv4 `EVENT_ID` to each new record and a monotonically increasing positive `SEQ` within the core safe-integer range. A retry preserves both identifiers and the entire original record. A new observation is a new record, not a mutation of an unacknowledged one.

A runner switching between an embedded reporter and chnotify MUST hand over ownership explicitly. It MUST NOT keep two independent publishers assigning sequence numbers to the same run. Its existing outbox and the daemon outbox MUST NOT independently forward the same logical stream to a display. Previously queued records require an explicit drain or migration decision before switching.

The adapter MAY retain native journal IDs in an extension for traceability. These do not replace core event identity. An optional `PARENT_RUN_ID` expresses a relationship to another execution; an ordinary task selection within a run does not automatically create a child run.

## 4. Core projection

All core values are strings, including decimal sequence, lease, and progress values. A snapshot is a complete projection: omitted optional fields are cleared. The publisher MUST emit a coherent snapshot when switching tasks, sessions, roles, waits, or attention, so old task context cannot remain attached to new work.

| Core field | Runner projection |
| --- | --- |
| `STATE` | `starting`, `running`, `stopping`, or `finished`, derived from authoritative execution state. |
| `ACTIVITY` | `working`, `idle`, `waiting`, or `paused`, independently of lifecycle. |
| `ATTENTION` | One summary category; `none` when no summary is active. |
| `STATUS` | Bounded human-readable description of current work. |
| `TASK_ID` | Current native task identifier; omitted when no task is current. |
| `REASON_CODE` | Stable machine-readable explanation, not an exception dump. |
| `PROGRESS_CURRENT`, `PROGRESS_TOTAL`, `PROGRESS_UNIT` | Actual measured progress, where the runner has a meaningful denominator. |
| `RESULT` | Explicit final result, present on terminal records. |
| `OBSERVED_AT` | Original UTC observation time with millisecond precision. |
| `LEASE_MS` | Core-bounded freshness lease, at most `256000`; never renewed by replay. |

`runner.turn_starting` can change current activity and task context. A validated `runner.task_result` can update status or progress, but MUST NOT by itself finish the run. A blocked task can coexist with a running process and independent work. Likewise, an intermediate worker failure is not necessarily an execution failure.

Example proposed Taskmd snapshot in the core field notation:

```text
VERSION=1
EVENT_ID=27d69046-807c-4d85-8f90-00b410533d7d
SOURCE_ID=workstation.taskmd.project
RUN_ID=0123456789abcdef01234567
SEQ=16
OBSERVED_AT=2026-11-12T15:48:00.000Z
KIND=snapshot
STATE=running
ACTIVITY=waiting
ATTENTION=blocker
STATUS=Waiting for the prerequisite of task T128
TASK_ID=T128
REASON_CODE=unmet_dependency
ATTENTION_SCOPE=live
ATTENTION_ID=task-T128-blocked-1
LEASE_MS=128000
X_TASKMD_ROUND=8
```

This notation illustrates the data model; the selected transport and encoding determine the actual framing.

## 5. Terminal results

The adapter MUST derive completion from the runner's domain result. Exit code zero alone is insufficient. The audited implementations currently use these mappings:

| Runner | Native termination condition | Existing v1 outcome / proposed core result |
| --- | --- | --- |
| Taskmd | `empty` reason and `complete` roadmap state | `completed` |
| Taskmd | `interrupted` | `cancelled` |
| Taskmd | `round_limit`, `no_ready_tasks`, `unmet_dependency`, `no_progress` | `needs_attention` |
| Taskmd | Other termination in its existing projection | `failed` |
| Roadmap | `complete` reason and `complete` roadmap state | `completed` |
| Roadmap | `stopped` with exit code 130 | `cancelled` |
| Roadmap | Other `stopped` | `needs_attention` |
| Roadmap | Deferred reason or state | `deferred` |
| Roadmap | Blocked, review-required, awaiting-verification, or needs-next-batch reason/state | `needs_attention` |
| Roadmap | Other termination in its existing projection | `failed` |

These are source-derived mappings, not heuristics for unrelated producers. See the `terminalOutcome` functions in the [Taskmd][taskmd-reporter] and [Roadmap][roadmap-reporter] reporters.

The proposed core additionally permits `unknown` where reliable observation establishes an ended execution but cannot establish its work outcome. Missing heartbeats alone MUST NOT fabricate termination. `unknown` has no equivalent in the current legacy v1 outcome enum; a legacy adapter MUST NOT silently relabel it as `failed` or `completed`.

Example terminal projection:

```text
VERSION=1
EVENT_ID=46b75fd8-e68a-46a4-89a5-1926a724497b
SOURCE_ID=workstation.taskmd.project
RUN_ID=0123456789abcdef01234567
SEQ=17
OBSERVED_AT=2026-11-12T15:49:00.000Z
KIND=terminal
STATE=finished
ACTIVITY=idle
ATTENTION=review
STATUS=Stopped at the configured round limit
RESULT=needs_attention
REASON_CODE=round_limit
ATTENTION_SCOPE=review
ATTENTION_ID=run-round-limit-1
```

The omitted task and lease fields clear the prior snapshot's values. Terminal state is irrevocable under the new core contract. The existing monitor blocks terminal-to-nonterminal resurrection but can accept a higher-sequence terminal replacement; strict terminal immutability therefore requires new receiver behaviour.

## 6. Attention and richer runner data

An active attention summary MUST have a stable `ATTENTION_ID` for the same episode and an explicit `ATTENTION_SCOPE`. A live request requires a nonterminal run and a lease. Review attention may survive completion. A receiver MUST distinguish an expired request to intervene in a live process from a durable request to inspect finished work.

The existing monitor has multiple attention records with task attribution, reason, category, `requires_attention`, and `recovering`. It also contains a legacy compatibility rule that changes generic input requests into non-actionable repair information during certain active worker roles. See [attention reconciliation][attention-code]. The new producer SHOULD supply precise semantics directly; core consumers MUST NOT infer them from runner-specific role names.

The single core summary cannot preserve every existing attention record. If multiple actionable categories coexist, the adapter MUST use a documented, deterministic summary policy and MUST NOT imply the remaining records have been resolved. A future runner profile MUST define stable item identities, simultaneous attention, recovery metadata, and exact clearing semantics before claiming full parity.

The same profile must define nested queue entries, append-only queue positions, sparse immutable history, validated last updates, session/source-revision binding, and `history_gap`. Core extension keys such as `X_TASKMD_ROUND` or `X_ROADMAP_BATCH_ID` MAY carry bounded string metadata, and relays preserve them under RFC 0001. An opaque serialized object in an extension does not establish an interoperable profile. Generic displays MUST remain useful without interpreting these extensions.

## 7. Freshness, delivery, and acknowledgement

The runner owns its observations and heartbeat schedule. A healthy daemon is not proof that its producer is healthy. It MUST NOT refresh a run merely because it can still forward stored records.

Freshness in the new protocol is bounded by the original `OBSERVED_AT` and `LEASE_MS`, using RFC 0001's clock and expiry rules. Receiving an old record now MUST NOT grant a new lease. Retries and recovery preserve original timestamps. Terminal records and historical replay never claim liveness.

The audited reporters already generate fresh snapshots immediately before transmission, send historical data as `replay`, and prevent replay from renewing contact. Their normal reporting timer is approximately 60 seconds; the existing monitor expires live contact after 180 seconds of server-observed silence. Those legacy timings and receipt-time semantics are not the new core freshness contract. See [reporter delivery][roadmap-reporter] and [monitor reduction/watchdog][monitor-storage].

A successful local acknowledgement means the daemon durably accepted responsibility under the configured queue policy. It does not prove remote ingestion, display, push notification, or human attention. Runners SHOULD use a bounded local submission deadline and expose reporting health separately. They MUST NOT wait for remote channel delivery to continue work.

An unacknowledged submission can have an uncertain outcome; retries MUST reuse its original identity. Queue-full, unavailable, rejected, and uncertain outcomes MUST remain distinguishable. Existing terminal state MUST remain recoverable until its ownership has transferred successfully. Compatibility rollout MUST verify this crash boundary rather than assuming a successful socket write is durable.

## 8. Receiver and migration contract

The preferred integration is a native chnotify ingestion endpoint with explicit protocol/profile support. It authenticates the source, applies core validation and ordering, deduplicates identities, persists terminal results, and projects a generic display. Browser subscriptions, SSE, WebSockets, and push delivery are receiver capabilities; they do not alter the producer's acknowledgement contract.

A legacy projection channel is permitted only when it possesses every required v1 field. It must maintain a coherent full snapshot and produce the exact strict envelope, including required process/run metadata, attention and event arrays. It MUST NOT invent queue state, process start times, task results, or runner versions to satisfy validation. Core-only producers therefore cannot assume universal legacy compatibility.

Legacy delivery must validate the expected installation identity. The existing reply contains `installation_id`, `stream_id`, `accepted_sequence`, and `applied`; `applied=false` can describe an already-seen or outdated projection. It is not, by itself, a transport failure. A compatibility adapter needs explicit duplicate/rejection handling and stable mapping of core ordering to report ordering. The legacy schema's probe and response contract is documented in [the integration guide][integration-docs] and implemented by [ingestion][monitor-ingest].

Credentials MUST remain outside notification bodies. A runner profile MUST NOT require prompts, provider transcripts, raw environment values, secrets, or absolute private paths. Existing runner credential stripping and bounded sanitization establish useful migration expectations; see [Taskmd private state][taskmd-state]. Source authorization must bind claimed identities to the authenticated principal.

## 9. Phased qualification

1. **Freeze core fixtures.** Validate the same records through every supported encoding, with exact identity, strings, clearing, and expiry preserved.
2. **Qualify local ownership transfer.** Exercise accepted submission, lost acknowledgement, daemon restart, full queue, and runner crash after acceptance. Confirm bounded runner latency and recoverable terminal state.
3. **Integrate one publisher.** Add opt-in runner emission from existing lifecycle hooks. Verify unchanged task selection, worker count, subprocess credentials, and exit semantics. Capture real records without invoking additional model work.
4. **Add a native receiver.** Test duplicates, out-of-order records, sequence conflicts, terminal immutability, disconnected replay, and expired live attention. Confirm that daemon recovery cannot make an exited process look live.
5. **Specify the runner profile.** Add fixtures for multiple blockers, task switches, resolver/compactor work, queues, validated results, and history gaps. Require explicit parity tests before replacing the existing UI projection.
6. **Switch and retain rollback.** Drain or migrate pending legacy deliveries, transfer sequence ownership, then enable the new path. Reversion must preserve identities and pending terminal records; it must not activate concurrent publishers.

Acceptance includes a drained Taskmd queue, a round-limit stop, cancellation, a recoverable worker error, a Roadmap queue transition, completed work requiring review, stale contact with unknown outcome, and restart after offline terminal delivery. Each scenario must demonstrate correct current state and bounded resource use, not merely an HTTP success response.

[monitor-schema]: https://github.com/bpstr/roadmap-monitor/blob/351a1942a87d6fff710ece1a6def7eaec4ab4c15/shared/protocol/v1.ts
[attention-code]: https://github.com/bpstr/roadmap-monitor/blob/351a1942a87d6fff710ece1a6def7eaec4ab4c15/shared/protocol/attention.ts
[integration-docs]: https://github.com/bpstr/roadmap-monitor/blob/351a1942a87d6fff710ece1a6def7eaec4ab4c15/docs/runner-integration.md
[taskmd-reporter]: https://github.com/bpstr/taskmd-runner/blob/5db4e1d70d569993a344ba1a2968004b010c3a66/src/monitor.ts
[taskmd-state]: https://github.com/bpstr/taskmd-runner/blob/5db4e1d70d569993a344ba1a2968004b010c3a66/src/monitor-state.ts
[roadmap-reporter]: https://github.com/bpstr/roadmap-runner/blob/b2128487cfae7a41d7b762efb6679004ea32204f/src/monitor.ts
[monitor-ingest]: https://github.com/bpstr/roadmap-monitor/blob/351a1942a87d6fff710ece1a6def7eaec4ab4c15/supabase/functions/monitor-ingest/index.ts
[monitor-storage]: https://github.com/bpstr/roadmap-monitor/blob/351a1942a87d6fff710ece1a6def7eaec4ab4c15/supabase/migrations/202611020001_foundation.sql
