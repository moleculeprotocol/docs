Updates metadata for an existing file without creating a new version. Note: The wallet address for audit trail is automatically derived from authentication.

Returns [`UpdateFileMetadataResult!`](/api-reference/types.md#updatefilemetadataresult).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `ref` | `String!` | Reference (DID) of the file to update. |
| `accessLevel` | `String!` | Access level of the file, `PUBLIC` or `ADMIN`, with `HOLDERS` accepted but unenforced. `ADMIN` restricts reading only for content encrypted at upload; the label alone protects nothing. |
| `description` | `String` | Description to store for the file. |
| `tags` | `[String!]` | Tags to store on the file. |
| `categories` | `[String!]` | Categories to store on the file. |
| `contentText` | `String` | Text content for searchability. |
