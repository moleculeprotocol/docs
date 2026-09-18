Result of generating or extending a service token. Success when `error == null`; the token fields are populated only on success.

| Name | Type | Description |
| --- | --- | --- |
| `token` | `String` | Generated JWT token for service authentication. |
| `tokenId` | `String` | Unique identifier for the token (for tracking/revocation). |
| `serviceName` | `String` |  |
| `expiresAt` | `AWSDateTime` |  |
| `createdAt` | `AWSDateTime` |  |
| `message` | `String` | Message with additional information; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
