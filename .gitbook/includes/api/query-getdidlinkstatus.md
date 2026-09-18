Public read-only snapshot of DID-linking state for the given OCL. Exposes the state-machine fields (status, attempts, userOpHash, txHash, account/data room DIDs, count of active onchain DIDs). DID-linking runs automatically in the background after createLab; this query is provided for diagnostic and support visibility. No authentication required.

Returns `DidLinkStatusResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
