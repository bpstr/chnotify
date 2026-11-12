# chnotify

**Channel notify: a small Rust tool that accepts process notifications and distributes them to configured channels.**

A long-running process reports a notification to the local daemon. chnotify takes responsibility for durable queuing and delivery, so the process does not need to implement remote authentication, retries, or an outbox.

## Status

Design proposal. The commands, configuration, and guarantees below describe the intended implementation; this repository does not yet contain the Rust daemon.

- [Architecture and delivery contract](docs/architecture.md)
- [Example configuration](examples/chnotify.toml)
- [File log destination and record formats](docs/file-channel.md)
- [Example TOML notification](examples/notification.toml)
- [Implementation and acceptance plan](docs/acceptance.md)

## Intended use

~~~sh
chnotify daemon --config ./examples/chnotify.toml

# In a separate process:
chnotify send READY=1 STATUS="Migration started"
chnotify send STATUS="Processed another batch"
~~~

The sender returns after the daemon has durably accepted the notification. Delivery to remote channels happens independently.

The field syntax above borrows from systemd notification messages. Notification meanings, required fields, schema versions, and format bindings will be specified in a dedicated repository.

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
