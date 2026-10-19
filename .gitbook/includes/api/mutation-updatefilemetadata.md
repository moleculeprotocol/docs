Updates a file's metadata, recorded as a new version with unchanged content. The change is attributed to the authenticated caller.

Returns [`UpdateFileMetadataResult!`](/api-reference/types.md#updatefilemetadataresult).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `ref` | `String!` | Reference (DID) of the file to update. |
| `accessLevel` | `String!` | Access level of the file, `PUBLIC` or `ADMIN`, with `HOLDERS` accepted but unenforced. `ADMIN` restricts reading only for content encrypted at upload; the label alone protects nothing. |
| `description` | `String` | Description to store for the file. |
| `tags` | `[String!]` | Tags to store on the file. |
| `categories` | `[String!]` | Categories to store on the file. |
| `contentText` | `String` | Searchable text, stored and returned unencrypted even on `ADMIN` files, so it should hold no more than the title. |
