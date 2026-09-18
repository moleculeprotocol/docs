Per-transaction onchain activity for a lab or wallet, newest first; distinct from labActivity, the data room feed. Requires at least one of `oclId` or `wallet`, AND-ed when both are given, and fails with VALIDATION_FAILED when both are missing or `oclId` is malformed.

Returns [`[OnChainEvent!]!`](/api-reference/types.md#onchainevent).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String` | Restrict the feed to this lab. 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `wallet` | `String` | Restrict the feed to events involving this wallet address, case-insensitive. Covers role grants and revocations for the account and transfers to or from it. |
| `limit` | `Int` | Maximum transaction groups to return, 1 to 200. Values outside that range fall back to 50. |
| `cursor` | `String` | `id` of the last entry of the previous page. A value missing either half, or with a non-integer block number, is ignored and the first page returned. |
