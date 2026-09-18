Generates the OCL membership agreement document for a Lab token and stores it in the public OCL agreements bucket. The returned `agreementKey` and `agreementContentHash` are the inputs to `getOclTermsMessage`. Fails with INVALID_METADATA when `agreementData` is not valid JSON, INVALID_INPUT when it fails validation or `oclId` is malformed, and INTERNAL_ERROR (retryable) when generation or storage fails.

Returns [`GenerateOclMembershipAgreementResult!`](/api-reference/types.md#generateoclmembershipagreementresult).

| Name | Type | Description |
| --- | --- | --- |
| `agreementData` | `AWSJSON!` | JSON object with `oclId` (32-byte OCL id as `0x` plus 64 hex, any case), `symbol` (ticker, 1-20 alphanumeric characters) and optionally `title` (display name). Any other key is rejected. |
