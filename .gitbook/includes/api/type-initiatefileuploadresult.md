Result of initiating a file upload.

| Name | Type | Description |
| --- | --- | --- |
| `datasetId` | `ID` | Dataset id of the file. Currently always null at this step: the id is assigned when `finishCreateOrUpdateFile` completes. |
| `uploadToken` | `String` | Opaque token identifying this upload; pass it to `finishCreateOrUpdateFile`. Null on failure. |
| `uploadUrl` | `String` | Pre-signed URL to send the file bytes to. Null on failure. |
| `uploadUrlExpiry` | `AWSDateTime` | When `uploadUrl` stops accepting uploads. Currently always null; the URL is short-lived, so upload promptly. |
| `method` | `String` | HTTP method to use against `uploadUrl` (normally "PUT"). Null on failure. |
| `headers` | `[Header!]` | Headers that must be sent with the upload request. Null on failure. |
| `expectedHeadHash` | `String` | Currently always null; not used by the upload protocol. |
| `useMultipart` | `Boolean` | Whether the upload must be performed as a multipart upload. Null on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
