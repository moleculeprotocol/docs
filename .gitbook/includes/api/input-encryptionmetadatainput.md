Encryption metadata stored with an encrypted (`ADMIN`) file, passed to `finishCreateOrUpdateFile` after the ciphertext upload. `kms` also requires `encryptedDek`, `iv` and `contentHash`, `bls` adds `keyId`, and omitting `encryptionSystem` requires every `Legacy:` field. A missing field fails with `INTERNAL_ERROR`.

| Name | Type | Description |
| --- | --- | --- |
| `encryptionSystem` | `String` | Encryption system identifier: "kms", "bls", or null/absent for a legacy ciphertext written before the onchain-verified envelope cutover. |
| `accessControlConditions` | `String!` | JSON-encoded array of access control conditions that `decryptDataKey` evaluates onchain, left to right, against the caller's wallet before releasing the key. Elements alternate between conditions and boolean operators `{ "operator": "and" \| "or" }`. A condition is either `{ "conditionType": "evmContract", "chain", "contractAddress", "functionName", "functionParams", "functionAbi", "returnValueTest": { "key", "comparator", "value" } }` or `{ "conditionType": "evmBasic", "chain", "contractAddress", "method", "parameters", "returnValueTest" }`. The literal `":userAddress"` in `functionParams` / `parameters` is replaced by the caller's wallet at evaluation time. Accepted `chain` values: "ethereum", "eth", "base", "sepolia", "sepolia-testnet", "sepolia-base", "baseSepolia". The Molecule app makes a confidential file readable by all lab members with two conditions joined by "or", both on the lab's AccessResolver contract: `hasRole(oclId, ":userAddress", "1")` = true (viewer or above) and `isAuthorizedSignerForTba(":userAddress", labAccountAddress)` = true (the LabNft owner). Evaluation fails closed. |
| `encryptedBy` | `String!` | Address that performed the encryption. |
| `encryptedAt` | `String!` | ISO 8601 timestamp when the file was encrypted. |
| `encryptedDek` | `String` | KMS/BLS: Base64-encoded wrapped key, as returned by `generateDataEncryptionKey`. |
| `iv` | `String` | KMS/BLS: Base64-encoded initialization vector. |
| `contentHash` | `String` | KMS/BLS: Hash of the encrypted content. |
| `keyId` | `String` | BLS: Key identifier. |
| `dataToEncryptHash` | `String` | Legacy: Hash of the data to be encrypted, recorded by the legacy client. |
| `chain` | `String` | Legacy: Blockchain network used for access control by the legacy client. |
| `litSdkVersion` | `String` | Legacy: SDK version that produced the legacy ciphertext. |
| `litNetwork` | `String` | Legacy: Network identifier from the legacy client (e.g., datil-test, habanero). |
| `templateName` | `String` | Legacy: Template name for the access control pattern. |
| `contractVersion` | `String` | Legacy: Version of the encryption contract. |
