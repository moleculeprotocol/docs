Standard error of the Molecule GraphQL API, returned inside a mutation's `*Result` envelope and thrown by queries as a GraphQL error with `code` in `errorType` and the rest in `errorInfo`. Null `error` means success. Some failures arrive instead as plain GraphQL errors with no `code`, which clients must handle too.

| Name | Type | Description |
| --- | --- | --- |
| `code` | `String!` | Stable machine-readable code from the error-code catalogue. The only field clients should branch on; treat an unknown code as a non-retryable failure. |
| `message` | `String!` | Human-readable explanation for developers; never empty. Not part of the contract, so do not parse or match on it. |
| `requestId` | `String!` | Correlation id for the request that failed. Include it in bug reports. |
| `retryable` | `Boolean!` | Whether retrying the same request unchanged can plausibly succeed. Retry with exponential backoff. |
| `details` | `AWSJSON` | Structured context for the failure. Keys: `field` (offending input), `reason` (second-level code), `hint` (next step), `docs` (URL); unknown keys may appear and must be ignored. |
