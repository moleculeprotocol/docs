Extends the lifetime of one of the caller's own service tokens, counted from now. Authenticated by the `x-service-token` header (fails with `UNAUTHENTICATED` when missing or invalid). A token the caller does not own is reported as not found; a revoked token fails with `FAILED_PRECONDITION`. Success when `error == null`.

Returns `ServiceTokenResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `tokenId` | `String!` | Value of `ServiceTokenResult.tokenId`, not the token itself. |
| `expiresIn` | `String!` | New lifetime from now, in the `expiresIn` format of `generateServiceToken` (minimum 1h, maximum 2y). |
