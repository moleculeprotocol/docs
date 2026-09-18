Single lab member entry, derived from the indexed onchain role state.

| Name | Type | Description |
| --- | --- | --- |
| `walletAddress` | `String!` | Lowercased wallet address of the member. |
| `role` | `LabMemberRole!` | Effective role on the lab. |
| `source` | `LabMemberSource!` | Source row that authoritatively defines this membership. |
| `expiry` | `String` | Unix-seconds expiry as a decimal string, encoded as a string because BigInt unix-seconds is unsafe as a JSON Int. Null means the grant is permanent. |
| `isAgent` | `Boolean!` | True if the member is an agent identity (separate from human auth, surfaced for UI but not used for authorization). |
| `grantedAt` | `String!` | ISO-8601 timestamp the row was first persisted. |
