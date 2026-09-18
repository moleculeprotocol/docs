Failure detail of a tokenization-service operation, present exactly when the result's `isSuccess` is false. Codes: INVALID_INPUT, INVALID_METADATA, MISSING_METADATA_FIELD, METADATA_UPLOAD_FAILED, IMAGE_UPLOAD_FAILED, UNSUPPORTED_IMAGE_TYPE, INVALID_TERMS_SIGNATURE, SIGNOFF_FAILED, TERMS_MESSAGE_FAILED, INTERNAL_ERROR.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String` | Human-readable explanation for developers. Not part of the contract, so do not parse or match on it. |
| `code` | `String` | Machine-readable failure code, from the set listed on `EvmTokenizationError`. |
| `retryable` | `Boolean!` | Whether retrying the same request unchanged can plausibly succeed. True only for transient upload, storage or signing failures, never for validation failures. |
| `details` | `AWSJSON` | Structured context. Currently always null for this service. |
