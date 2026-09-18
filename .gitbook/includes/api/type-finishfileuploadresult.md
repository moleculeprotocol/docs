Result of finishing a file upload.

| Name | Type | Description |
| --- | --- | --- |
| `datasetId` | `ID` | Dataset id of the file; its stable `ref` for later versions and metadata updates. Null on failure. |
| `contentHash` | `String` | Hash of the stored file content. Null on failure. |
| `version` | `Int` | Version number of the file after this upload (1 for a new file). Null on failure. |
| `newHead` | `String` | Head hash of the data room after the commit, usable as `expectedHead` for optimistic concurrency in later operations. Null on failure. |
| `message` | `String` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
