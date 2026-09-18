Completes an upload once the bytes have been PUT to the `uploadUrl` from `initiateCreateOrUpdateFile`. Provide `path` for a new file or `ref` for a new version of an existing one, never both.

Returns [`FinishFileUploadResult!`](/api-reference/types.md#finishfileuploadresult).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `uploadToken` | `String!` | Value of `uploadToken` returned by `initiateCreateOrUpdateFile`. |
| `path` | `String` | Path for a new file, e.g. `foo.txt`. Underscores are not allowed. |
| `ref` | `String` | Dataset ref of an existing file, to store a new version of it. |
| `accessLevel` | `String!` | Access level of the file, `PUBLIC` or `ADMIN`, with `HOLDERS` accepted but unenforced. `ADMIN` restricts reading only for content encrypted at upload; the label alone protects nothing. |
| `changeBy` | `String` | Deprecated and ignored: the change is attributed to the authenticated caller. Removed after 2027-01-01. |
| `description` | `String` | Description of the file or version. |
| `tags` | `[String!]` | Tags for the file or version. |
| `contentText` | `String` | Text content for searchability. |
| `categories` | `[String!]` | Categories for the file or version. |
| `encryptionMetadata` | [`EncryptionMetadataInput`](/api-reference/types.md#encryptionmetadatainput) | Encryption metadata for encrypted files (KMS, BLS, or legacy). |
