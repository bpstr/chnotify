# chnotify project RFCs

These documents are project specifications for review. They are not IETF RFCs and do not claim an implementation exists.

| RFC | Title | Status |
|---|---|---|
| [0001](0001-chnotify-protocol.md) | Process notifications and reliable channel delivery, version 1 | Draft 1, 2026-11-12 |

RFC 0001 supplies the field rules, codec bindings and wire grammars left open by the [architecture proposal](../architecture.md). Where the draft and older illustrative examples differ, RFC 0001 is the proposed contract. Acceptance of an RFC and runtime conformance are separate milestones.

The semantic protocol is intended for a dedicated specification repository. Its name is not selected here. Until that repository exists, this RFC and the [portable fixture bundle](../../spec/fixtures/v1/README.md) are the review source. Extract them with their history and publish a pinned version before independent implementations claim version 1 conformance; do not maintain two competing registries.

Changes to required fields, field meaning, codec grammar or receipt meaning require a new protocol/profile version after version 1 is frozen. Editorial clarifications and additional test vectors may retain the version if they do not change accepted behaviour. Draft revisions must identify any breaking changes.

The [runner and monitor integration plan](../integrations/runners-and-monitors.md) records audited existing behaviour and future migration requirements.
