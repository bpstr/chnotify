# chnotify

**Channel notify: a small Rust tool that accepts process notifications and distributes them to configured channels.**

A long-running process reports a notification to the local daemon. chnotify takes responsibility for durable queuing and delivery, so the process does not need to implement remote authentication, retries, or an outbox.

## Status

Design proposal. The commands, configuration, and guarantees below describe the intended implementation; this repository does not yet contain the Rust daemon.

- [RFC 0001: protocol, lifecycle, codecs, wire framing and delivery](docs/rfcs/0001-chnotify-protocol.md)
- [Runner and monitor integration specification](docs/integrations/runners-and-monitors.md)
- [Portable protocol and wire fixtures](spec/fixtures/v1/README.md)
- [Architecture and delivery contract](docs/architecture.md)
- [Example configuration](examples/chnotify.toml)
- [File log destination and record formats](docs/file-channel.md)
- [Example TOML notification](examples/notification.toml)
- [Implementation and acceptance plan](docs/acceptance.md)

## Intended use

~~~sh
chnotify daemon --config ./examples/chnotify.toml

# In a separate process, submit a complete example observation:
chnotify send --input ./examples/notification.toml --format toml
~~~

The sender returns after the daemon has durably accepted the notification. Delivery to remote channels happens independently.

RFC 0001 defines the proposed `process/1` field meanings, string-map encodings, lifecycle and freshness rules, and the `CHN1` acknowledgement protocol. Its portable contract and fixtures are intended to move into a dedicated specification repository before independent version 1 implementations ship.

The design keeps process state, freshness and delivery state separate. Buffered heartbeats cannot renew an expired lease; terminal results remain final. See the RFC for systemd-inspired CLI shorthands and explicit run/sequence ownership.

The existing runner integrations use roadmap-monitor's different, strict v1 envelope. The integration specification defines a future migration; these documents do not change either runner or claim current wire compatibility.

## Initial capabilities

| Concern | Initial design |
|---|---|
| Implementation | One Rust executable with sender and daemon commands |
| Local submission | A framed Unix socket with a durable acceptance reply |
| Channels | Unix socket, HTTP/HTTPS, and local log-file adapters |
| Authentication | Configurable HTTP bearer, basic, or custom-header credentials |
| Payload formats | TSV, YAML, JSON, JSONL, TOML, plus a compact key/value input |
| File records | RFC 5424-formatted text by default; optional JSONL log envelopes |
| Reliability | Persistent outbox, independent channel outcomes, retries and recovery |
| Resource use | Event-driven idle behavior and bounded memory, storage, and concurrency |

The notification schema, serialization format, and delivery channel are independent. A socket can carry YAML; a web endpoint can receive TSV; a log record can carry a TOML payload. Payload codec and file record format are separate settings. Each channel declares what counts as successful delivery.

The project scope is notification distribution. Processes supply their own status, and downstream tools interpret or display it.

