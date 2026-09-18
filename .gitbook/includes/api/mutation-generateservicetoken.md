Issues a service token (JWT) for non-browser automation such as agents, bots and pipelines. Supply both `walletAddress` and `messageSignature` to authenticate by wallet signature over the message from `getServiceSignInMessage`, whose nonce is single-use and valid for 10 minutes, or supply neither to authenticate with the caller's session. Fails with `UNAUTHENTICATED` when the nonce is missing, expired or the signature does not match, and `VALIDATION_FAILED` when only one of the two is supplied.

Returns [`ServiceTokenResult!`](/api-reference/types.md#servicetokenresult).

| Name | Type | Description |
| --- | --- | --- |
| `serviceName` | `String!` | Name of the service the token is for; recorded with the token and echoed in `ServiceTokenResult.serviceName`. |
| `expiresIn` | `String` | Lifetime of the token as `<number><unit>` with unit s, m, h, d, w, M (30 days) or y (365 days), e.g. "30d", "6M", "1y". Default "180d"; minimum 1h, maximum 2y. |
| `walletAddress` | `String` | Wallet address for signature-based auth (both walletAddress and messageSignature required together). Use cases: agents, bots, services. |
| `messageSignature` | `String` | Hex-encoded signature of the service sign-in message. |
