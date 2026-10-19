Revokes one of the caller's own personal API credentials, taking effect on the next request made with it. Requires the caller's session, exactly as `issueConsumerCredential` does. Credentials that do not exist or belong to someone else fail with `NOT_FOUND`.

Returns [`RevokeConsumerCredentialResult!`](/api-reference/types.md#revokeconsumercredentialresult).

| Name | Type | Description |
| --- | --- | --- |
| `consumerId` | `ID!` | Value of `ConsumerCredential.consumerId`, not the credential string. |
