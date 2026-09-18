Result type for file metadata update operations.

| Name | Type | Description |
| --- | --- | --- |
| `ref` | `String` | Reference (DID) of the updated file. |
| `message` | `String!` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
