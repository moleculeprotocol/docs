Result type for the getServiceSignInMessage query.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Message the service must sign with its wallet. Embeds a server-issued single-use nonce, so it must be fetched fresh before each signing. |
| `expiresAt` | `String` | ISO-8601 expiry of the embedded nonce; after this the signature is rejected and a fresh message must be requested. |
