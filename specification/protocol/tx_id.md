# Warehouse transaction identifiers

**Status:** v1 additive — `SignalKind.TX_ID_NOT_UUIDV7`

`tx_id` (and the details fact column `t`) is an [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html)
UUID **version 7**, canonical `8-4-4-4-12` string.

| Rule | Detail |
|------|--------|
| MUST | version nibble `7` |
| SHOULD | prefer v7 over v1 and v6 (RFC 9562 §4) |
| MUST NOT | land v1 / v4 / v6 / non-UUID in the warehouse |
| Range expiry | `epoch_hour` (unix hours), week-wide tablets, settle after 4 weeks — **not** the UUID string |

## Remediation

A peer that submits a non-v7 `tx_id` is the **source**. Signals (or any
warehouse writer) **MUST NOT** rewrite the id. It **MUST** return the row
and **SHOULD** call `zndx.engine.v1.Engine/Remediate` on the source with:

```text
capability = reauthor
signal.kind = TX_ID_NOT_UUIDV7
signal.offending = <rejected tx_id>
signal.subject = <product_id>
signal.authority = RFC 9562 §5.7
```

The source remints a v7 and resubmits. Disposition `CORRECTED` means the
`correction` field is the new `tx_id`.
