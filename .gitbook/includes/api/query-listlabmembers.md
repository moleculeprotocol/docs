Active members of a lab, from the indexed onchain role state, excluding grants that have expired. Membership is granted onchain through the AccessResolver contract rather than through this API. Public, with no authentication beyond an API key, mirroring the public onchain role state.

Returns `ListLabMembersResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
