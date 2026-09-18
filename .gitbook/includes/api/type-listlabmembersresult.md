Result type for the listLabMembers query. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Human-readable status message. |
| `members` | `[LabMember!]!` | Active members on the lab. Empty array when the lab has no recorded members. |
