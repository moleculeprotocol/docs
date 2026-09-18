Starts a file or file-version upload to a lab's data room, returning a pre-signed `uploadUrl` and the `uploadToken` for the finish step. PUT the bytes to that URL, then call `finishCreateOrUpdateFile` with the token.

Returns `InitiateFileUploadResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `contentType` | `String!` | MIME type to store with the file. |
| `contentLength` | `Int!` | Size of the upload in bytes. |
