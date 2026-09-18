Result of generating a standalone data encryption key.

| Name | Type | Description |
| --- | --- | --- |
| `plaintextDEK` | `String` | Base64-encoded data encryption key for encrypting content client-side. Null on failure. |
| `encryptedDek` | `String` | Base64-encoded wrapped key to store alongside the ciphertext. Null on failure. |
| `encryptionSystem` | `String` | Encryption system that produced the key; always `kms`. Null on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
