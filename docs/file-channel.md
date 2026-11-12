# File log destination

Status: implementation design; no file adapter exists yet.

[RFC 0001](rfcs/0001-chnotify-protocol.md) defines the proposed process profile, receipt meanings and version 1 file metadata. The exact [text record](../spec/fixtures/v1/file-envelope.rfc5424) and [JSONL record](../spec/fixtures/v1/file-envelope.jsonl) are portable fixtures.

The built-in `file` channel writes process notifications to a dedicated local log. Its configuration selects both a payload `format` and a log `record_format`. The payload codecs are TSV, YAML, JSON, JSONL and TOML, plus the compact key/value binding.

For example, `format = "toml"` and `record_format = "rfc5424"` write a TOML notification inside a text log record. They do not make the entire log a TOML document. Appending independent TOML or YAML documents without record framing is not this adapter's contract.

See [the channel configuration](../examples/chnotify.toml) and [the example TOML notification](../examples/notification.toml).

## Default text record

Use RFC 5424 message syntax and define LF as the file record delimiter. Each complete notification occupies exactly one physical line. This reuses syslog message formatting; it does not implement a network syslog service.

Freeze the UTC acceptance timestamp, host and source metadata with the delivery. Defaults are facility `user`, severity `informational`, app name `chnotify`, unknown process ID `-`, and message category `notification`. Explicit schema mappings may select a different severity; the adapter does not infer process health.

Use `STRUCTURED-DATA = -`. Place `record_version`, `source_id`, `event_id`, `delivery_id`, `channel`, `schema_profile`, `payload_format`, `mode`, and `payload` in the message as quoted key/value pairs, in that order. The record version is the string `1`; mode is `observation` or `history` and must match the accepted delivery. This avoids inventing a syslog structured-data namespace. RFC 5424 specifies the header and structured-data rules. [F1]

The complete [RFC 5424 fixture](../spec/fixtures/v1/file-envelope.rfc5424) is one physical line containing a valid `process/1` TOML payload. Its two-character `\n` sequences represent embedded newlines; decoding the quoted payload recovers [snapshot.toml](../spec/fixtures/v1/snapshot.toml) exactly.

The file binding adds the following exact escaping rule: encode every message-field value as a quoted JSON string literal using ASCII output. Escape quotes, backslashes, control characters and non-ASCII characters; use surrogate pairs when required by that string syntax. This produces an ASCII RFC 5424 MSG and prevents newline/terminal-control injection. It does not require a JSON document or change the inner payload format. A small shared quoting routine is sufficient.

Header fields must satisfy the RFC's character and length rules. Reject an invalid configured header instead of inserting raw control characters. The complete encoded record, including its final LF, must fit the channel's record-size bound before admission.

## Optional JSONL record

With `record_format = "jsonl"`, write one compact JSON object plus LF. Required keys are `record_version`, `timestamp`, `severity`, `host`, `app`, `process_id`, `source_id`, `event_id`, `delivery_id`, `channel`, `schema_profile`, `payload_format`, `mode`, and `payload`. `record_version` is the JSON integer `1`; unknown `process_id` is null, otherwise a string. All other values are strings. These envelope types do not alter the inner string-map protocol. The [JSONL fixture](../spec/fixtures/v1/file-envelope.jsonl) supplies a complete record.

Use ASCII string escaping here too, so controls and Unicode cannot introduce ambiguous physical records. The envelope's `payload` remains a string even when the payload codec is JSON or JSONL. Consumers first decode the envelope, then apply `payload_format`. The proposed log-envelope version and its fixtures must be frozen with RFC 0001 before implementation.

Freeze the entire record before accepting its delivery. Retries keep the same timestamp, IDs, metadata and bytes; attempted-write times belong in distribution diagnostics.

## Append and durability

Use an exclusively owned regular file on a local filesystem, open it for append, and serialize all writes to it. Hold an exclusive cooperative lock using a stable sidecar lock identity that survives rename-based rotation. Reject duplicate configured destinations or a second writer. Unrelated processes must not append to this file.

For every delivery:

1. Check the bounded record and active-file size ceiling.
2. Append the complete record, handling short writes.
3. Synchronize the file with `fdatasync` or `fsync`.
4. Persist outbox completion only after synchronization succeeds.

When creating the file, also persist its directory entry before promising durability. A successful file sync alone does not persist the containing directory entry. [F2]

File append and SQLite completion are separate operations. A crash between them can result in a repeated complete record. The file channel explicitly uses `replay = "allow-duplicates"`, preserving the delivery ID so readers can identify repeats. It does not scan historical logs or promise exactly-once append.

Run blocking append, synchronization and recovery work through bounded workers. On write/sync failure, disk-full, missing permissions or the active-file ceiling, retain the delivery and retry or block appropriately; other destinations continue.

## Tail recovery

Before resuming writes after a crash or partial append, inspect the tail under the exclusive writer lock. Bound the read by the maximum record size.

For a file established as belonging exclusively to this channel, truncate only an incomplete final record back to the last complete LF boundary, synchronize the repair, then replay pending work. Handle an incomplete first record by returning to offset zero. Never truncate complete records.

If the tail exceeds the expected bound, contains unexpected corruption, has uncertain ownership, or shows evidence of external writes, block the channel for inspection. Do not append a fresh record onto an incomplete tail. Complete duplicate records remain valid and are not silently removed.

## Rotation and storage bounds

Support external rename-and-create rotation followed by an explicit reopen signal, initially SIGHUP. Finish/synchronize the current append, then reopen the configured path while retaining the channel's writer lock. Keep using the original descriptor until the explicit reopen so an intermediate rename cannot split a record. Failed reopen keeps pending work.

Directory synchronization and ownership checks also apply to a newly created replacement file. The external rotator must preserve rotated files until reopen completes; compression or deletion must wait for the handoff, and its durable-retention guarantees remain part of the deployment contract.

Use a configured active-file size ceiling; stop appending before a record would exceed it. Resume after successful rotation/reopen. This log storage is separate from the outbox quota. External retention manages rotated files.

There is no idle rotation polling and no reliable copytruncate mode in this design. Delivered records may later be removed by the configured retention policy; durable append is not an indefinite archive guarantee.

## References

- [F1: RFC 5424 syslog message format](https://www.rfc-editor.org/rfc/rfc5424.html#section-6)
- [F2: Linux fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)

The record profile, append sequence and operational defaults are chnotify design choices. See [the acceptance plan](acceptance.md) for their required validation.

