#### CreateLabResult

Result of creating a lab.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Human-readable message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
| `lab` | `LabRef` | Created lab details if successful (minimal fields only). |
