Result of generateOclMembershipAgreement. The document is stored in the public OCL agreements bucket rather than on IPFS, so it is addressed by a storage key and URL instead of a CID. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `agreementKey` | `String` | Storage key of the agreement document, of the form `<oclId>/agreements/<uuid>.json`, to pass to `getOclTermsMessage`. Null on failure. |
| `agreementUrl` | `String` | Public HTTPS URL of the agreement document: the agreements base URL followed by `agreementKey`. Null on failure. |
| `agreementContentHash` | `String` | SHA-256 of the agreement document as `0x` plus 64 lowercase hex characters, the onchain `contentHash` format. Deterministic for a given oclId, symbol and title, and null on failure. |
| `agreementType` | `OclAgreementType` | Always OCL_MEMBERSHIP on success. Null on failure. |
| `generatedAt` | `AWSDateTime` | When this response was produced, which is informational only and not part of the deterministic stored document. Null on failure. |
| `isSuccess` | `Boolean!` | True when the agreement was generated and stored. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |
