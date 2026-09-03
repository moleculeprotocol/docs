# Files

Working with files in a Lab dataroom: the three-step upload flow (initiate → upload → finish), plus metadata updates, deletion, storage limits, and client-side encryption. Creating the Lab itself is covered in [Lab Management](lab-management.md).

> **Looking for a runnable walkthrough?** This page is the per-operation reference. For a first upload with expected responses and failure handling at every step, use [Create a lab and upload a file](../getting-started/create-lab-and-upload-file.md) or [Upload an encrypted file](../getting-started/upload-encrypted-file.md) (encrypted, with a decrypt round trip).

> **Note**: Every mutation on this page returns its failure in-band: the result carries `error: ApiError`, and success means `error` is `null`. Branch on `error.code` — never on `message` text — and quote `requestId` when reporting a problem. See [Error Handling](README.md#error-handling) for the `ApiError` shape, how to read `details`, and the list of error codes.

## Step 1: Initiate File Upload

Initiates the upload process and returns a presigned URL for direct file upload.

**GraphQL Mutation:**

```graphql
mutation InitiateFileUpload(
  $oclId: String!
  $contentType: String!
  $contentLength: Int!
) {
  initiateCreateOrUpdateFile(
    oclId: $oclId
    contentType: $contentType
    contentLength: $contentLength
  ) {
    uploadToken
    uploadUrl
    uploadUrlExpiry
    method
    headers {
      key
      value
    }
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

**Parameters:**

| Parameter     | Type   | Required | Description                                                               |
| ------------- | ------ | -------- | ------------------------------------------------------------------------- |
| oclId         | String | Yes      | Canonical 32-byte oclId of the lab (lowercase 0x-hex, e.g. `0x0101…0042`) |
| contentType   | String | Yes      | MIME type of the file (e.g., `application/pdf`, `image/png`)              |
| contentLength | Int    | Yes      | File size in bytes                                                        |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation InitiateFileUpload($oclId: String!, $contentType: String!, $contentLength: Int!) { initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) { uploadToken uploadUrl uploadUrlExpiry method headers { key value } error { code message requestId retryable details } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "contentType": "application/pdf",
      "contentLength": 381846
    }
  }'
```

**Success Response:**

```json
{
  "data": {
    "initiateCreateOrUpdateFile": {
      "uploadToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "uploadUrl": "https://s3.amazonaws.com/bucket/path?signature=...",
      "uploadUrlExpiry": "2024-01-15T10:45:00.000Z",
      "method": "PUT",
      "headers": [
        {
          "key": "Content-Type",
          "value": "application/pdf"
        }
      ],
      "error": null
    }
  }
}
```

On failure `error` is set and the upload fields (`uploadToken`, `uploadUrl`, `uploadUrlExpiry`, `method`, `headers`) are `null`; do not proceed to Step 2 unless `error` is `null`.

## Step 2: Upload File to Storage

Upload the file directly to the presigned URL returned in Step 1.

**Example Request (curl):**

```bash
curl -X PUT "UPLOAD_URL_FROM_STEP_1" \
  -H "Content-Type: application/pdf" \
  --data-binary @your-file.pdf
```

**Example Request (JavaScript):**

```javascript
const uploadHeaders = {};
headers.forEach((h) => {
  uploadHeaders[h.key] = h.value;
});

const uploadResponse = await fetch(uploadUrl, {
  method: "PUT",
  headers: uploadHeaders,
  body: fileBuffer,
});

if (!uploadResponse.ok) {
  throw new Error(`Upload failed: ${uploadResponse.statusText}`);
}
```

## Step 3: Finish File Upload

Completes the upload process and registers the file in the dataroom.

**GraphQL Mutation:**

```graphql
mutation FinishFileUpload(
  $oclId: String!
  $uploadToken: String!
  $path: String
  $ref: String
  $accessLevel: String!
  $changeBy: String!
  $description: String
  $tags: [String!]
  $categories: [String!]
  $contentText: String
) {
  finishCreateOrUpdateFile(
    oclId: $oclId
    uploadToken: $uploadToken
    path: $path
    ref: $ref
    accessLevel: $accessLevel
    changeBy: $changeBy
    description: $description
    tags: $tags
    categories: $categories
    contentText: $contentText
  ) {
    datasetId
    contentHash
    version
    message
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

**Parameters:**

| Parameter   | Type      | Required | Description                                                 |
| ----------- | --------- | -------- | ----------------------------------------------------------- |
| oclId       | String    | Yes      | Same oclId used in Step 1                                   |
| uploadToken | String    | Yes      | Token received from Step 1                                  |
| path        | String    | No\*     | File name for NEW files (e.g., `research-data.pdf`)         |
| ref         | String    | No\*     | Dataset ID for NEW VERSIONS of existing files               |
| changeBy    | String    | Yes      | Wallet address of user making the change                    |
| description | String    | No       | Optional file description                                   |
| tags        | \[String] | No       | Optional tags for categorization                            |
| categories  | \[String] | No       | Optional categories for organization                        |
| contentText | String    | No       | Optional searchable text content (used for semantic search) |

_\*Use `path` for new files OR `ref` for versions - not both_

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation FinishFileUpload($oclId: String!, $uploadToken: String!, $path: String, $accessLevel: String!, $changeBy: String!, $description: String, $tags: [String!], $categories: [String!], $contentText: String) { finishCreateOrUpdateFile(oclId: $oclId, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel, changeBy: $changeBy, description: $description, tags: $tags, categories: $categories, contentText: $contentText) { datasetId contentHash version message error { code message requestId retryable details } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "uploadToken": "TOKEN_FROM_STEP_1",
      "path": "research-results.pdf",
      "accessLevel": "PUBLIC",
      "changeBy": "0x1234567890123456789012345678901234567890",
      "description": "Q4 2024 Research Results",
      "tags": ["research", "results", "2024"],
      "categories": ["data"],
      "contentText": "Quarterly research findings and experimental data"
    }
  }'
```

**Success Response:**

```json
{
  "data": {
    "finishCreateOrUpdateFile": {
      "datasetId": "did:kamu:...",
      "contentHash": "sha256:abc123...",
      "version": 1,
      "message": "File uploaded successfully",
      "error": null
    }
  }
}
```

**Failure Response** (HTTP 200, no top-level `errors[]` — `message` mirrors `error.message`):

```json
{
  "data": {
    "finishCreateOrUpdateFile": {
      "datasetId": null,
      "contentHash": null,
      "version": null,
      "message": "You are not allowed to perform this operation.",
      "error": {
        "code": "UNAUTHORIZED",
        "message": "You are not allowed to perform this operation.",
        "requestId": "8f1e4c9a-2b7d-4e10-9c3a-5d6f7a8b9c0d",
        "retryable": false,
        "details": "{\"reason\":\"UNAUTHORIZED\"}"
      }
    }
  }
}
```

---

## Complete Example

Here's a complete Node.js example demonstrating the full 3-step workflow:

```javascript
#!/usr/bin/env node

const fs = require("fs");
const fetch = require("node-fetch");

async function uploadFileToLabs(filePath, oclId, serviceToken) {
  const apiUrl = "https://production.graphql.api.molecule.xyz/graphql";
  const fileBuffer = fs.readFileSync(filePath);
  const filename = require("path").basename(filePath);
  const fileSize = fs.statSync(filePath).size;

  try {
    // Step 1: Initiate upload
    console.log("Step 1: Initiating file upload...");
    const initiateResponse = await fetch(apiUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": process.env.CONSUMER_CREDENTIAL,
        "X-Service-Token": serviceToken,
      },
      body: JSON.stringify({
        query: `
          mutation InitiateFileUpload($oclId: String!, $contentType: String!, $contentLength: Int!) {
            initiateCreateOrUpdateFile(
              oclId: $oclId
              contentType: $contentType
              contentLength: $contentLength
            ) {
              uploadToken
              uploadUrl
              method
              headers { key value }
              error { code message requestId retryable details }
            }
          }
        `,
        variables: {
          oclId,
          contentType: "application/octet-stream",
          contentLength: fileSize,
        },
      }),
    });

    const initiateResult = await initiateResponse.json();
    if (initiateResult.errors?.length) {
      // Top-level errors on a mutation mean a transport or request-validation failure
      const e = initiateResult.errors[0];
      throw new Error(
        `${e.errorType ?? "Request failed"}: ${e.message}${
          e.errorInfo?.requestId ? ` (requestId ${e.errorInfo.requestId})` : ""
        }`,
      );
    }
    const initiateError = initiateResult.data.initiateCreateOrUpdateFile.error;
    if (initiateError) {
      throw new Error(
        `${initiateError.code}: ${initiateError.message} (requestId ${initiateError.requestId})`,
      );
    }

    const { uploadToken, uploadUrl, headers } =
      initiateResult.data.initiateCreateOrUpdateFile;
    console.log("✅ Upload initiated");

    // Step 2: Upload to presigned URL
    console.log("Step 2: Uploading file to storage...");
    const uploadHeaders = {};
    headers?.forEach((h) => {
      uploadHeaders[h.key] = h.value;
    });

    const uploadResponse = await fetch(uploadUrl, {
      method: "PUT",
      headers: uploadHeaders,
      body: fileBuffer,
    });

    if (!uploadResponse.ok) {
      throw new Error(`Upload failed: ${uploadResponse.statusText}`);
    }
    console.log("✅ File uploaded to storage");

    // Step 3: Finish upload
    console.log("Step 3: Finalizing upload...");
    const finishResponse = await fetch(apiUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": process.env.CONSUMER_CREDENTIAL,
        "X-Service-Token": serviceToken,
      },
      body: JSON.stringify({
        query: `
          mutation FinishFileUpload(
            $oclId: String!
            $uploadToken: String!
            $path: String!
            $accessLevel: String!
            $changeBy: String!
          ) {
            finishCreateOrUpdateFile(
              oclId: $oclId
              uploadToken: $uploadToken
              path: $path
              accessLevel: $accessLevel
              changeBy: $changeBy
            ) {
              datasetId
              message
              error { code message requestId retryable details }
            }
          }
        `,
        variables: {
          oclId,
          uploadToken,
          path: filename,
          accessLevel: "PUBLIC",
          changeBy: process.env.WALLET_ADDRESS,
        },
      }),
    });

    const finishResult = await finishResponse.json();
    if (finishResult.errors?.length) {
      const e = finishResult.errors[0];
      throw new Error(
        `${e.errorType ?? "Request failed"}: ${e.message}${
          e.errorInfo?.requestId ? ` (requestId ${e.errorInfo.requestId})` : ""
        }`,
      );
    }
    const finishError = finishResult.data.finishCreateOrUpdateFile.error;
    if (finishError) {
      throw new Error(
        `${finishError.code}: ${finishError.message} (requestId ${finishError.requestId})`,
      );
    }

    console.log("🎉 File upload completed successfully!");
    console.log(
      "Dataset ID:",
      finishResult.data.finishCreateOrUpdateFile.datasetId,
    );

    return {
      success: true,
      datasetId: finishResult.data.finishCreateOrUpdateFile.datasetId,
    };
  } catch (error) {
    console.error("❌ Upload failed:", error.message);
    throw error;
  }
}

// Usage
if (require.main === module) {
  const filePath = process.argv[2];
  const oclId = process.argv[3];
  const serviceToken = process.env.SERVICE_TOKEN;

  if (!filePath || !oclId || !serviceToken) {
    console.error(
      'Usage: SERVICE_TOKEN="token" WALLET_ADDRESS="0x..." node upload.js <file> <ocl-id>',
    );
    process.exit(1);
  }

  uploadFileToLabs(filePath, oclId, serviceToken);
}

module.exports = { uploadFileToLabs };
```

**Usage:**

```bash
CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret" SERVICE_TOKEN="your-service-token" WALLET_ADDRESS="0x..." node upload.js data.pdf 0x0101000000000000000000000000000000000000000000000000000000000042
```
---

## Update File Metadata

Update file metadata (description, tags, categories, access level) without creating a new version.

**GraphQL Mutation:**

```graphql
mutation UpdateFileMetadata(
  $oclId: String!
  $ref: String!
  $accessLevel: String!
  $description: String
  $tags: [String!]
  $categories: [String!]
  $contentText: String
) {
  updateFileMetadata(
    oclId: $oclId
    ref: $ref
    accessLevel: $accessLevel
    description: $description
    tags: $tags
    categories: $categories
    contentText: $contentText
  ) {
    ref
    message
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

**Parameters:**

| Parameter   | Type      | Required | Description                                                   |
| ----------- | --------- | -------- | ------------------------------------------------------------- |
| oclId       | String    | Yes      | Canonical 32-byte oclId of the lab                            |
| ref         | String    | Yes      | File reference (DID) from `finishCreateOrUpdateFile` response — the `datasetId`, **not** the file path |
| accessLevel | String    | Yes      | `PUBLIC`, `HOLDERS` or `ADMIN`. Required: this call replaces the metadata rather than patching it, so omitting it fails validation |
| description | String    | No       | Updated file description                                      |
| tags        | \[String] | No       | Updated tags for categorization                               |
| categories  | \[String] | No       | Updated categories for organization                           |
| contentText | String    | No       | Updated searchable text content                               |

> **Note**: The `changeBy` field (wallet address) is automatically derived from your authentication and does not need to be provided as a parameter.

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation UpdateFileMetadata($oclId: String!, $ref: String!, $accessLevel: String!, $description: String, $tags: [String!], $categories: [String!], $contentText: String) { updateFileMetadata(oclId: $oclId, ref: $ref, accessLevel: $accessLevel, description: $description, tags: $tags, categories: $categories, contentText: $contentText) { ref message error { code message requestId retryable details } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "ref": "did:kamu:fed01...",
      "accessLevel": "PUBLIC",
      "description": "Updated research findings with peer review",
      "tags": ["research", "peer-reviewed", "2024"],
      "categories": ["data", "validated"],
      "contentText": "Enhanced searchable content with key findings"
    }
  }'
```

---

## Delete File

Remove a file from the dataroom permanently.

**GraphQL Mutation:**

```graphql
mutation DeleteFile($oclId: String!, $path: String!, $changeBy: String!) {
  deleteDataRoomFile(oclId: $oclId, path: $path, changeBy: $changeBy) {
    oclId
    filePath
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

**Parameters:**

| Parameter | Type   | Required | Description                        |
| --------- | ------ | -------- | ---------------------------------- |
| oclId     | String | Yes      | Canonical 32-byte oclId of the lab |
| path      | String | Yes      | File path to delete                |
| changeBy  | String | Yes      | Wallet address making the deletion |

> **Warning**: This is a destructive operation. The file will be permanently deleted from the dataroom and cannot be recovered.

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation DeleteFile($oclId: String!, $path: String!, $changeBy: String!) { deleteDataRoomFile(oclId: $oclId, path: $path, changeBy: $changeBy) { oclId filePath error { code message requestId retryable details } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "path": "old-data.pdf",
      "changeBy": "0x1234567890123456789012345678901234567890"
    }
  }'
```

---

## Get File by Path

Retrieve a specific file using the lab's `oclId` and the file path.

**GraphQL Query:**

```graphql
query GetFile($oclId: String!, $path: String!) {
  dataRoomFile(oclId: $oclId, path: $path) {
    did
    path
    version
    contentType
    accessLevel
    description
    tags
    categories
    contentText
    downloadUrl
    downloadUrlExpiry
  }
}
```

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query GetFile($oclId: String!, $path: String!) { dataRoomFile(oclId: $oclId, path: $path) { did path contentType accessLevel downloadUrl } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "path": "research-data.pdf"
    }
  }'
```

---

## File Categories & Tags

### Get File Categories and Tags

Return the valid file categories and their tags from the CMS. Use these when tagging or categorizing files. **Public query** — no authentication required.

```graphql
query FileCategoriesAndTags {
  fileCategoriesAndTags {
    data {
      name
      tags
    }
  }
}
```

Each entry in `data` is a `FileCategory` with a `name` and its list of allowed `tags`. Failures arrive as top-level GraphQL `errors[]` entries with `errorType` set to the catalogue code (e.g. `UPSTREAM_UNAVAILABLE`); the result type has no `error` field.

---

## File Requirements & Limits

### Storage Limits

- **Default Limit**: 5GB per lab/project
- **Custom Limits**: Can be increased upon request - contact the Molecule team
- **Note**: the Labs web app additionally caps individual uploads at 100 MB per file; API uploads are not subject to that app-side cap

### Supported File Types

- All file types are supported
- Common types: PDF, PNG, JPEG, CSV, JSON, ZIP, etc.

### Optional Metadata

Enhance file discoverability with optional metadata:

- **description**: Human-readable description of the file
- **tags**: Array of tags for categorization (e.g., `["research", "q4-2024"]`)
- **categories**: Array of categories for organization (e.g., `["data", "results"]`)
- **contentText**: Searchable text content for full-text search

---

## Advanced: Encrypted File Upload

> **Step-by-step version:** [Upload an encrypted file](../getting-started/upload-encrypted-file.md), including both access-condition recipes (owner-only, and owner/contributor/viewer) and a decrypt round trip that verifies the gate actually works.

For files requiring client-side encryption, obtain a data encryption key via the `generateDataEncryptionKey` mutation, encrypt locally, upload as normal, and include an `encryptionMetadata` object on `finishCreateOrUpdateFile`. The full end-to-end model — key wrapping, onchain access conditions, and condition-gated decryption — is documented on the [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md) page.

### Obtain a DEK, then encrypt locally

`generateDataEncryptionKey` (no arguments) returns `plaintextDEK`, `encryptedDek`, and `encryptionSystem`. The client uses `plaintextDEK` to AES-256-GCM encrypt the file locally (Web Crypto `SubtleCrypto`), then wipes it from memory. The upload itself uses the standard `initiateCreateOrUpdateFile` → PUT → `finishCreateOrUpdateFile` flow, with the encrypted bytes uploaded to the presigned URL.

### Encryption Metadata Parameter (Onchain-Verified Envelope Encryption, current default)

```graphql
$encryptionMetadata: EncryptionMetadataInput
```

```json
{
  "encryptionMetadata": {
    "encryptionSystem": "<echo value returned by generateDataEncryptionKey>",
    "encryptedDek": "BASE64_WRAPPED_DEK",
    "iv": "BASE64_AES_GCM_IV",
    "contentHash": "sha256-...",
    "accessControlConditions": "[{...}]",
    "encryptedBy": "0x1234567890123456789012345678901234567890",
    "encryptedAt": "2026-01-15T10:30:00.000Z"
  }
}
```

`encryptionSystem` is **backend-set** — clients must echo the value returned by `generateDataEncryptionKey` rather than hardcode it. This keeps the roadmap rollover to BLS threshold key custody transparent to existing integrations.

#### `accessControlConditions` — gating decryption by role

`accessControlConditions` is a JSON-stringified array of `EvmContractCondition` predicates joined by `BooleanCondition` separators (`and` / `or`). The backend evaluates each predicate against live chain state at decrypt time via viem `readContract`, short-circuits booleans, and fails closed on RPC error. To gate decryption on _LabNFT owner OR active Contributor OR active Viewer_, OR `AccessResolver.isAuthorizedSignerForTba(:userAddress, tba)` against `AccessResolver.hasRole(oclId, :userAddress, ROLE_VIEWER)` — the role-hierarchy collapses Contributor + Viewer into one check on the canonical chain (Base).

The placeholder `:userAddress` in `functionParams` is substituted with the authenticated caller's wallet at evaluate time. The full `EvmContractCondition` JSON shape, the worked OR-composite example, and condition-evaluator semantics are documented on the [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md#worked-example-encrypt-for-owner-or-contributor-or-viewer) page.

#### Role Management (onchain, off this API surface)

Role grants are **onchain transactions on the `AccessResolver` contract**, not Labs API mutations. Lab owners (and active Contributors, for the Viewer slot) call `grantRole(oclId, account, role, expiry, isAgent)` / `revokeRole(oclId, account)` directly via viem / ethers / Safe. The Labs API only _consumes_ role state at decrypt time through the `accessControlConditions` evaluator. See [Roles & Permissions](../../technical-deep-dive/roles-and-permissions.md) for the capability matrix, grant lifecycle (expiry, `isAgent`), and the [`AccessResolver` reference](../../references/contracts/accessresolver.md) for the onchain interface.

**When to Use Encryption:**

- Sensitive research data requiring access control
- Compliance requirements for data protection
- Conditional access based on token ownership or lab role

---

## Data Encryption Keys

### Generate a Data Encryption Key

Generate a standalone data encryption key (DEK) for client-side encryption outside the file-upload flow. Returns both the plaintext DEK (used to encrypt data locally, then wiped) and the KMS-encrypted DEK (stored alongside the ciphertext). Requires authentication (Privy user or service token). See [Advanced: Encrypted File Upload](#advanced-encrypted-file-upload) for the file-upload encryption path.

```graphql
mutation GenerateDataEncryptionKey {
  generateDataEncryptionKey {
    plaintextDEK
    encryptedDek
    encryptionSystem
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

| Field            | Type     | Description                                                |
| ---------------- | -------- | ---------------------------------------------------------- |
| plaintextDEK     | String   | Base64-encoded plaintext DEK (only present on success)     |
| encryptedDek     | String   | Base64-encoded KMS-encrypted DEK (only present on success) |
| encryptionSystem | String   | Encryption system used (always `"kms"`)                    |
| error            | ApiError | `null` on success; non-null means the mutation failed      |

---

