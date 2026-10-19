Issues a personal API credential for the caller, returned by this response only and unrecoverable afterwards. Requires the caller's session; a service token is refused. Requests that would take the caller past 5 active credentials fail with `CONFLICT`. Fails with `FAILED_PRECONDITION` until the caller has accepted the current API terms, whose version and documents the error details name.

Returns [`IssueConsumerCredentialResult!`](/api-reference/types.md#issueconsumercredentialresult).

| Name | Type | Description |
| --- | --- | --- |
| `label` | `String` | Label for telling credentials apart, at most 64 characters. |
| `expiresInDays` | `Int` | Lifetime in whole days, 1 to 365. Default 90. |
| `acceptedTermsVersion` | `String` | Value of `ApiTermsStatus.termsVersion`, sent only once the user has accepted the documents, and recorded as that acceptance. Other values fail with `FAILED_PRECONDITION`. |
