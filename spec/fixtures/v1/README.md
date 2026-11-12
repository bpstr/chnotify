# Proposed process/1 fixtures

These fixtures accompany [RFC 0001](../../../docs/rfcs/0001-chnotify-protocol.md). They are portable contract examples, not a Rust implementation or an exhaustive conformance suite. Freeze and extract this bundle with the semantic specification into the dedicated protocol repository before declaring version 1 stable.

## Contents

| Files | Expected meaning |
|---|---|
| `snapshot.json`, `.jsonl`, `.yaml`, `.toml`, `.tsv`, `.kv` | The same complete event, including empty, Unicode, equals-sign, quote and literal-backslash extension values |
| `escaped.json`, `.jsonl`, `.yaml`, `.toml`, `.tsv` | The same event with decoded TAB/LF/CR/backslash content in one extension; KV output must reject this map |
| `manifest.json` | Expected semantic digest, equivalent payloads, exact frame lengths/hashes and metadata |
| `submit-toml.frame.hex` | Complete CHN1 SUBMIT frame, including the four-octet length prefix |
| `accepted.frame.hex` | Durable local acceptance receipt |
| `deliver-toml.frame.hex`, `delivered.frame.hex` | Native destination request and durable receipt |
| `queue-full.frame.hex` | Definite non-acceptance with a retry delay |
| `invalid-*.frame.hex` | Too-short/oversize lengths, truncated body and metadata overrun; reject or close without false success |
| `file-envelope.rfc5424`, `file-envelope.jsonl` | The same TOML bytes inside both version 1 log envelopes; each file has exactly one physical record |
| `invalid-cases.json` and its referenced files | Representative malformed encoding and semantic-validation cases |
| `projection-cases.json` | Fourteen ordering, freshness, finality, clearing and attention scenarios |

Hexadecimal fixture lines are presentation only: remove ASCII whitespace and hex-decode the entire file before parsing CHN1. SHA-256 values in the manifest cover those decoded binary bytes. Payload equality compares the decoded map, not serialized whitespace or field order.

The RFC semantic digest for `snapshot.*` in `profile=process/1`, `mode=observation` is:

```text
61ff858a75bab3ace70d8a375bd0e41f0d97a36f4f5ee402ee31afa0900d5d51
```

The fixture timestamps are fixed historical test data. Sending them to a real monitor later must not show a current live process. The example TOML notification in `examples/` contains the same map plus explanatory comments.

## Scenario convention

Each scenario starts with an empty receiver projection and a valid authenticated namespace. Its event records are already decoded string maps. Wall time is expressed as the specified timestamp; `received_mono_ms` uses an independent monotonic test clock. Clock confidence is true initially unless changed by a step.

An event step supplies `mode`, first receipt clocks, and selected expected state facts. An `advance_mono_ms` step advances the monotonic clock to that absolute offset and expires leases. A clock-change step can change wall time or confidence without reversing the monotonic clock. Expected objects describe assertions, not a complete implementation-specific state representation.

`conflict` means the event must not change the current projection. An implementation may durably store the rejection as a diagnostic before returning `applied=0`; the scenario does not require one particular diagnostic storage backend. `post-terminal-observation` is an explanatory diagnostic label for a first terminal event below previously seen nonterminal state, not an additional wire error code.

For semantic invalid cases with `set` or `remove`, begin with `snapshot.json`, apply those changes, then require rejection. File-based invalid examples can violate more than one rule; no rule allows them to be accepted merely because a different error is encountered first.

## Release qualification

An implementation must additionally test boundary sizes, every codec's malformed syntax, duplicate-key detection before ordinary parser normalization, split/coalesced frames, clock/restart failures, file durability, authenticated HTTP receipts and multi-channel recovery. The [acceptance plan](../../../docs/acceptance.md) defines those gates. Checking this bundle validates specimen consistency; it does not prove runtime durability, footprint or compatibility with the legacy monitor endpoint.
