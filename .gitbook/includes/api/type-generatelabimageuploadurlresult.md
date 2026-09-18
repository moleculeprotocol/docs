Result of `generateLabImageUploadUrl`. The presigned PUT URL is single-use and expires per `expiresAt`, and the lab's `image` is updated asynchronously once the uploaded object has been processed.

| Name | Type | Description |
| --- | --- | --- |
| `uploadUrl` | `String` | Pre-signed HTTPS URL for a single PUT of the image, sent with the same Content-Type the URL was generated for. Null on failure. |
| `key` | `String` | Storage key of the image (`<oclId>/<uuid>.<extension>`). Null on failure. |
| `expiresAt` | `AWSDateTime` | When `uploadUrl` stops accepting uploads (15 minutes after issue). Null on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
