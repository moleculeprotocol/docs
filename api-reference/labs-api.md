# ⚙️ Programmatic File Upload

## Overview

The Programmatic File Upload API allows developers to automate file uploads to Molecule Labs datarooms without requiring browser-based user interaction. This enables integration with automated workflows, data pipelines, CI/CD systems, and external applications.

### Use Cases

- **Automated Data Pipelines**: Schedule regular data synchronization from research systems
- **CI/CD Integration**: Automatically publish build artifacts and test results
- **External System Integration**: Connect third-party tools and platforms to your Lab
- **Batch Operations**: Upload multiple files programmatically
- **Monitoring & Alerting**: Automated upload of logs and metrics

> **Ready for Production**: This API is production-ready and actively used by projects for automated data management. To request API access, please join our [Discord community](https://t.co/L0VEiy4Bjk) and reach out to our team.

---

## Authentication

The Labs API has different authentication requirements depending on the operation type:

> **Rule of thumb**: Most **queries** are public (API Key only) and all write **mutations** require a Service Token (API Key + Service Token). The exceptions are called out below — a couple of queries are role-gated, and `generateServiceToken` bootstraps a token with a wallet signature.

### Public Queries (Read-Only)

**These queries** are public and only require an API Key:
- `labs` - List all labs with pagination
- `labWithDataRoomAndFiles` - Get lab details and files
- `labActivity` - Get activity feed for a lab, (available filters: ANNOUNCEMENT | FILE)
- `activities` - Get global activity feed, (available filters: ANNOUNCEMENT | FILE)
- `dataRoomFile` - Get file by path
- `searchLabs` - Search across labs, files, and announcements
- `fileCategoriesAndTags` - List valid file categories and their tags
- `getServiceSignInMessage` - Get the message a service signs to obtain a token
- `getDidLinkStatus` - Get background DID-linking status for a lab
- `legalAgreementStatus` - Check whether a lab's legal agreement is signed
- `onChainActivity` - On-chain event feed for a lab or wallet
- `listLabMembers` - List a lab's members

```bash
x-api-key: YOUR_API_KEY
```

**Authenticated query** — API Key **plus** a Service Token, or an authenticated user session:
- `legalAgreementTemplate` - Get the populated agreement to sign (the signer's authenticated session, or a service token)

### Protected Mutations (Write Operations)

**All write mutations** require authentication with **two headers**:

1. **API Key** - For general API authentication
2. **Service Token** - For lab-specific write access control

**Protected mutations include:**
- `createLab` - Create a lab (data room) for an on-chain lab (OCL) · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `initiateCreateOrUpdateFile` - Initiate file upload · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `finishCreateOrUpdateFile` - Complete file upload · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `updateFileMetadata` - Update file metadata
- `deleteDataRoomFile` - Delete a file
- `createAnnouncement` - Create an announcement · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `updateLabNftMetadata` - Update LabNFT display metadata (OCL admin only)
- `generateLabImageUploadUrl` - Get a presigned URL to upload a LabNFT image (OCL admin only)
- `signLegalAgreement` - Record acceptance of a legal agreement
- `generateDataEncryptionKey` - Generate a standalone data encryption key · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `decryptDataKey` - Decrypt a file's data key for an authorized caller · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `dataRoomPassphrase` - Get Telegram bot passphrase for a dataroom
- `extendServiceToken` - Extend service token expiration
- `revokeServiceToken` - Revoke a service token

> **`generateServiceToken` is the bootstrap exception**: it mints a Service Token, so it needs only an API Key plus either a Privy session or a wallet signature — not a pre-existing Service Token. See [Obtaining Tokens](#obtaining-tokens).

> **Pay-per-call alternative.** Mutations tagged 💳 above can also be called through the [x402 Gateway](x402-gateway.md), which settles a USDC payment on Base per request and mints a short-lived service token on the fly — no long-lived credentials required. Useful for autonomous AI agents and third-party tools that pay for users.

```bash
x-api-key: YOUR_API_KEY
X-Service-Token: YOUR_SERVICE_TOKEN
```

### Obtaining API Key and Service Token

To obtain access credentials:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team and provide:
   - Your wallet address (will be linked to the service token)
   - Intended use case / service name
   - Which lab/dataroom you need access to
   - Desired token expiration period
3. The team will generate and provide you with:
   - **API Key** - Used for all Molecule APIs
   - **Service Token** (JWT string) - Grants access to specific lab
   - **Token ID** - For management operations

### Using Your Credentials

**For all queries** (read-only operations):
```bash
x-api-key: YOUR_API_KEY
```

**For all mutations** (write operations):
```bash
x-api-key: YOUR_API_KEY
X-Service-Token: YOUR_SERVICE_TOKEN
```

**Why two tokens for mutations?**

- **API Key**: Authenticates you as a valid Molecule API user
- **Service Token**: Identifies which specific lab/dataroom you have write access to

**Security Warnings:**

- Service tokens are shown only once during generation - store them securely immediately
- Never commit tokens or API keys to version control
- Never log credentials in application logs
- Store in environment variables or secure secret management systems
- Rotate tokens regularly (quarterly recommended)

---

## API Endpoints

The Programmatic File Upload API uses a 3-step workflow:

### API Base URL

```
Production: https://production.graphql.api.molecule.xyz/graphql
Staging:    https://staging.graphql.api.molecule.xyz/graphql
```

### Step 1: Initiate File Upload

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
    isSuccess
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter     | Type   | Required | Description                                                                         |
| ------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| oclId      | String | Yes      | Canonical 32-byte oclId of the lab (lowercase 0x-hex, e.g. `0x0101…0042`)            |
| contentType   | String | Yes      | MIME type of the file (e.g., `application/pdf`, `image/png`)                        |
| contentLength | Int    | Yes      | File size in bytes                                                                  |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation InitiateFileUpload($oclId: String!, $contentType: String!, $contentLength: Int!) { initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) { uploadToken uploadUrl uploadUrlExpiry method headers { key value } isSuccess error { message code retryable } } }",
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
      "isSuccess": true,
      "error": null
    }
  }
}
```

### Step 2: Upload File to Storage

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

### Step 3: Finish File Upload

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
    isSuccess
    message
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter   | Type     | Required | Description                                                 |
| ----------- | -------- | -------- | ----------------------------------------------------------- |
| oclId       | String   | Yes      | Same oclId used in Step 1                                   |
| uploadToken | String   | Yes      | Token received from Step 1                                  |
| path        | String   | No\*     | File name for NEW files (e.g., `research-data.pdf`)         |
| ref         | String   | No\*     | Dataset ID for NEW VERSIONS of existing files               |
| accessLevel | String   | Yes      | File visibility: `PUBLIC`, `HOLDERS`, or `ADMIN`            |
| changeBy    | String   | Yes      | Wallet address of user making the change                    |
| description | String   | No       | Optional file description                                   |
| tags        | [String] | No       | Optional tags for categorization                            |
| categories  | [String] | No       | Optional categories for organization                        |
| contentText | String   | No       | Optional searchable text content (used for semantic search) |

_\*Use `path` for new files OR `ref` for versions - not both_

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation FinishFileUpload($oclId: String!, $uploadToken: String!, $path: String, $accessLevel: String!, $changeBy: String!, $description: String, $tags: [String!], $categories: [String!], $contentText: String) { finishCreateOrUpdateFile(oclId: $oclId, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel, changeBy: $changeBy, description: $description, tags: $tags, categories: $categories, contentText: $contentText) { datasetId contentHash version isSuccess message error { message code retryable } } }",
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
      "isSuccess": true,
      "message": "File uploaded successfully",
      "error": null
    }
  }
}
```

---

### Update File Metadata

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
    isSuccess
    message
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter   | Type     | Required | Description                                                     |
| ----------- | -------- | -------- | --------------------------------------------------------------- |
| oclId       | String   | Yes      | Canonical 32-byte oclId of the lab                              |
| ref         | String   | Yes      | File reference (DID) from `finishCreateOrUpdateFile` response   |
| accessLevel | String   | Yes      | File visibility: `PUBLIC`, `HOLDERS`, or `ADMIN`                |
| description | String   | No       | Updated file description                                        |
| tags        | [String] | No       | Updated tags for categorization                                 |
| categories  | [String] | No       | Updated categories for organization                             |
| contentText | String   | No       | Updated searchable text content                                 |

> **Note**: The `changeBy` field (wallet address) is automatically derived from your authentication and does not need to be provided as a parameter.

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation UpdateFileMetadata($oclId: String!, $ref: String!, $accessLevel: String!, $description: String, $tags: [String!], $categories: [String!], $contentText: String) { updateFileMetadata(oclId: $oclId, ref: $ref, accessLevel: $accessLevel, description: $description, tags: $tags, categories: $categories, contentText: $contentText) { ref isSuccess message error { message } } }",
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

### Delete File

Remove a file from the dataroom permanently.

**GraphQL Mutation:**

```graphql
mutation DeleteFile($oclId: String!, $path: String!, $changeBy: String!) {
  deleteDataRoomFile(oclId: $oclId, path: $path, changeBy: $changeBy) {
    oclId
    filePath
    isSuccess
    error {
      message
      code
      retryable
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
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation DeleteFile($oclId: String!, $path: String!, $changeBy: String!) { deleteDataRoomFile(oclId: $oclId, path: $path, changeBy: $changeBy) { oclId filePath isSuccess error { message } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "path": "old-data.pdf",
      "changeBy": "0x1234567890123456789012345678901234567890"
    }
  }'
```

---

### Create Announcement

Create project announcements to share updates with your community.

**GraphQL Mutation:**

```graphql
mutation CreateAnnouncement(
  $oclId: String!
  $headline: String!
  $body: String!
  $attachments: [String!]
) {
  createAnnouncement(
    oclId: $oclId
    headline: $headline
    body: $body
    attachments: $attachments
  ) {
    isSuccess
    message
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter   | Type     | Required | Description                                      |
| ----------- | -------- | -------- | ------------------------------------------------ |
| oclId       | String   | Yes      | Canonical 32-byte oclId of the lab               |
| headline    | String   | Yes      | Announcement title/headline                      |
| body        | String   | Yes      | Announcement body (supports Markdown)            |
| attachments | [String] | No       | Array of file DIDs to attach to the announcement |

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation CreateAnnouncement($oclId: String!, $headline: String!, $body: String!, $attachments: [String!]) { createAnnouncement(oclId: $oclId, headline: $headline, body: $body, attachments: $attachments) { isSuccess message error { message } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "headline": "Research Milestone Achieved",
      "body": "We have completed Phase 2 trials with promising results.",
      "attachments": ["did:kamu:fed01..."]
    }
  }'
```

---

### Create Lab

Register a Kamu-backed lab (data room) for an on-chain lab (OCL) that already exists on-chain. The lab is identified by its canonical `oclId` (a 32-byte hex string, 0x-prefixed).

> **Admin Authorization Required**: This mutation requires either a service token (JWT) from the Molecule team OR a valid Privy authentication token. The caller must be the LabNFT owner (or an authorized multisig signer) for the given `oclId`.

**GraphQL Mutation:**

```graphql
mutation CreateLab($oclId: String!) {
  createLab(input: { oclId: $oclId }) {
    isSuccess
    message
    error {
      message
      code
      retryable
    }
    lab {
      oclId
      shortname
      labAccountAddress
      labNftTokenId
    }
  }
}
```

**Parameters:**

The mutation takes a single `CreateLabInput` object:

| Field  | Type   | Required | Description                                                          |
| ------ | ------ | -------- | -------------------------------------------------------------------- |
| oclId  | String | Yes      | Canonical 32-byte oclId (lowercase 0x-hex) of the on-chain lab       |

**Prerequisites:**

1. **LabNFT Ownership**: You must own the LabNFT for the `oclId` or be an authorized signer for it
   - For individual wallets: You must be the owner
   - For multisig/Safe wallets: You must be one of the Safe owners
   - For ERC-4337 accounts: You must be an authorized account owner

2. **Authentication**: One of the following:
   - **Service Token** (recommended for automation): Obtain from Molecule team via Discord
   - **Privy Token** (for user-initiated requests): Use your authenticated Privy session

3. **LabNFT Must Be Minted**: The on-chain lab (LabNFT / `oclId`) must already exist on-chain before registering the lab

**Authentication Options:**

**Option 1: Service Token (Recommended for Automation)**
```bash
x-api-key: YOUR_API_KEY
X-Service-Token: YOUR_SERVICE_TOKEN
```

**Option 2: Privy Token (User-Initiated)**
```bash
x-api-key: YOUR_API_KEY
Authorization: Bearer YOUR_PRIVY_TOKEN
x-wallet-address: YOUR_WALLET_ADDRESS
```

**Example Request (Service Token):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation CreateLab($oclId: String!) { createLab(input: { oclId: $oclId }) { isSuccess message error { message code retryable } lab { oclId shortname labAccountAddress labNftTokenId } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042"
    }
  }'
```

**Success Response:**

```json
{
  "data": {
    "createLab": {
      "isSuccess": true,
      "message": "Lab created successfully",
      "error": null,
      "lab": {
        "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
        "shortname": "apob-lab",
        "labAccountAddress": "0x1234567890123456789012345678901234567890",
        "labNftTokenId": "42"
      }
    }
  }
}
```

**Error Responses:**

**Not Authorized (No Token):**
```json
{
  "errors": [{
    "message": "Admin authorization required. Please contact Molecule team for service token access.",
    "extensions": { "code": "UNAUTHORIZED" }
  }]
}
```

**Not the LabNFT Owner:**
```json
{
  "data": {
    "createLab": {
      "isSuccess": false,
      "message": "User is not authorized for this lab",
      "error": {
        "message": "On-chain verification failed: wallet address is not owner or authorized signer",
        "code": "OWNERSHIP_VERIFICATION_FAILED",
        "retryable": false
      },
      "lab": null
    }
  }
}
```

**Lab Already Exists:**
```json
{
  "data": {
    "createLab": {
      "isSuccess": false,
      "message": "Lab already exists for this oclId",
      "error": {
        "message": "A lab with this oclId already exists",
        "code": "CONFLICT",
        "retryable": false
      },
      "lab": null
    }
  }
}
```

**How It Works:**

1. **Authentication Check**: Validates service token or Privy token
2. **On-Chain Verification**: Verifies you own or are an authorized signer for the LabNFT (`oclId`)
3. **Lab Creation**: Registers the Kamu-backed lab and its data room for the `oclId`
4. **Whitelist Update**: Automatically adds your wallet address to the lab whitelist
5. **Returns Result**: Lab details if successful, error details if failed

**Use Cases:**

- **Automate Lab Creation**: Register labs programmatically after minting LabNFTs
- **CI/CD Integration**: Automatically set up data rooms for new research labs
- **Batch Operations**: Register multiple labs for a portfolio of on-chain labs
- **User Self-Service**: Allow users to create their own lab data rooms

**Getting Service Token Access:**

To obtain a service token for automated lab creation:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team
3. Provide:
   - Your wallet address
   - Use case description
   - Intended automation workflow
4. You'll receive:
   - API Key (for all APIs)
   - Service Token (JWT for lab creation)
   - Token expiration date

---

### Querying Projects and Files

Query operations for browsing projects, viewing files, and checking activity.

#### List All Projects

Get all labs. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `labs` query does not require authentication. You only need the `x-api-key` header - no Service Token is needed.

**GraphQL Query:**

```graphql
query ListProjects($walletAddress: String, $page: Int, $perPage: Int) {
  labs(walletAddress: $walletAddress, page: $page, perPage: $perPage) {
    nodes {
      oclId
      shortname
      labAccountAddress
      labNftTokenId
      systemTime
      eventTime
      trlValue
      trlRationale
      isVerified
      account {
        accountName
      }
      dataRoom {
        id
        alias
        status
      }
    }
    totalCount
    pageInfo {
      hasNextPage
      hasPreviousPage
      currentPage
      totalPages
    }
  }
}
```

**Parameters:**

| Parameter     | Type          | Required | Description                                                                                                        |
| ------------- | ------------- | -------- | ------------------------------------------------------------------------------------------------------------------ |
| walletAddress | String        | No       | Filter to labs where this wallet holds any active role (owner/contributor/viewer). Omit to return all labs.        |
| role          | LabMemberRole | No       | Only meaningful with `walletAddress`: restrict to labs where the wallet holds this specific role (`OWNER`, `CONTRIBUTOR`, `VIEWER`). |
| page          | Int           | No       | Page number (0-indexed, default: 0)                                                                                |
| perPage       | Int           | No       | Results per page (default: 20, max: 100)                                                                           |

**CMS-enriched fields (optional):**

These fields are sourced from the Molecule CMS and hydrated only when requested in the selection set. They are `null` when the project has no corresponding CMS entry.

| Field        | Type    | Description                                                       |
| ------------ | ------- | ----------------------------------------------------------------- |
| trlValue     | String  | Technology Readiness Level (TRL) assessment for the project       |
| trlRationale | String  | Explanation supporting the assigned TRL value                     |
| isVerified   | Boolean | Whether the project has been verified by Molecule                 |

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query ListProjects($page: Int, $perPage: Int) { labs(page: $page, perPage: $perPage) { nodes { oclId shortname dataRoom { id alias } } totalCount pageInfo { hasNextPage currentPage totalPages } } }",
    "variables": {
      "page": 0,
      "perPage": 20
    }
  }'
```

#### Get Single Project with Files

Retrieve complete details for a specific lab including all files. This is a **public endpoint** - no authentication required. Look up a lab by its `oclId` or, alternatively, by its human-readable `shortname` — provide exactly one.

> **🔓 Public Endpoint**: The `labWithDataRoomAndFiles` query does not require authentication. You only need the `x-api-key` header - no Service Token is needed. File-level access control is handled via encryption rather than query-level authentication.

**GraphQL Query:**

```graphql
query GetProject($oclId: String!) {
  labWithDataRoomAndFiles(oclId: $oclId) {
    oclId
    shortname
    trlValue
    trlRationale
    isVerified
    dataRoom {
      id
      alias
      files {
        did
        path
        version
        contentType
        accessLevel
        description
        tags
        categories
        downloadUrl
      }
    }
  }
}
```

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query GetProject($oclId: String!) { labWithDataRoomAndFiles(oclId: $oclId) { oclId shortname dataRoom { id files { path contentType accessLevel tags } } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042"
    }
  }'
```

> The optional CMS-enriched fields `trlValue`, `trlRationale`, and `isVerified` (see [List All Projects](#list-all-projects)) are also available on this query and are hydrated only when requested.

#### Get File by Path

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
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "query GetFile($oclId: String!, $path: String!) { dataRoomFile(oclId: $oclId, path: $path) { did path contentType accessLevel downloadUrl } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "path": "research-data.pdf"
    }
  }'
```

#### Project Activity Feed

Get activity timeline for a specific project including file events and announcements. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `labActivity` query does not require authentication. You only need the `x-api-key` header - no Service Token is needed.

> **Filtering**: By default, returns all activity types (file events and announcements). Use the optional `filter` parameter (`ANNOUNCEMENT` or `FILE`) to retrieve only a specific type.

**GraphQL Query:**

```graphql
  query GetProjectActivity($id: String!, $page: Int!, $perPage: Int!, $filter: LabActivityFilter) {
    labActivity(oclId: $id, page: $page, perPage: $perPage, filter: $filter) {
      pageInfo {
        hasNextPage
        hasPreviousPage
        currentPage
        totalPages
      }
      nodes {
        __typename
        ... on LabEventFileAdded {
          entry {
            ref
            path
            tags
            description
            version
            accessLevel
            eventTime
            systemTime
            changeBy
            categories
            contentType
            contentHash
            contentText
          }
        }
        ... on LabEventFileUpdated {
          entry {
            ref
            path
            tags
            description
            version
            accessLevel
            eventTime
            systemTime
            changeBy
            categories
            contentType
            contentHash
            contentText
          }
        }
        ... on LabEventFileRemoved {
          entry {
            ref
            path
            tags
            description
            version
            accessLevel
            eventTime
            systemTime
            changeBy
            categories
            contentType
            contentHash
            contentText
          }
        }
        ... on LabEventAnnouncement {
          announcement {
            id
            headline
            body
            attachments {
              id
              did
              path
              name
              contentType
              accessLevel
            }
            changeBy
            systemTime
            eventTime
          }
        }
      }
    }
  }
```

> **⚠️ Breaking Change**: Announcement `attachments` changed from `[String!]!` (array of DIDs) to `[DataRoomFile!]!` (array of file objects). This enables querying file metadata directly without separate API calls.

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query GetActivity($oclId: String!, $page: Int) { labActivity(oclId: $oclId, page: $page, perPage: 20) { pageInfo { hasNextPage currentPage totalPages } nodes { __typename ... on LabEventAnnouncement { announcement { headline attachments { did path contentType } } } } } }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
      "page": 0
    }
  }'
```

**Use Cases:**

- Announcement detail pages requiring full file metadata
- Download links for announcement attachments
- Encrypted file access (Onchain-Verified Envelope Encryption for new files, Lit Protocol for legacy)
- Projects with many announcements (efficient pagination)

#### Global Activity Feed

Get all activity across all projects. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `activities` query does not require authentication. You only need the `x-api-key` header - no Service Token is needed.

> **Filtering**: By default, returns all activity types (file events and announcements). Use the optional `filter` parameter (`ANNOUNCEMENT` or `FILE`) to retrieve only a specific type.

**GraphQL Query:**

```graphql
query GetActivities($page: Int, $perPage: Int, $filter: LabActivityFilter) {
    activities(page: $page, perPage: $perPage, filter: $filter) {
      isSuccess
      activities {
        __typename
        ... on LabEventFileAdded {
          entry {
            ref
            path
            tags
            description
            version
            accessLevel
            eventTime
            systemTime
            changeBy
            categories
            contentType
            contentHash
            contentText
          }
        }
        ... on LabEventFileUpdated {
          entry {
            ref
            path
            tags
            description
            version
            accessLevel
            eventTime
            systemTime
            changeBy
            categories
            contentType
            contentHash
            contentText
          }
        }
        ... on LabEventFileRemoved {
          entry {
            ref
            path
            tags
            description
            version
            accessLevel
            eventTime
            systemTime
            changeBy
            categories
            contentType
            contentHash
            contentText
          }
        }
        ... on LabEventAnnouncement {
          announcement {
            id
            headline
            body
            attachments {
              id
              did
              path
              name
              contentType
              accessLevel
            }
            changeBy
            systemTime
            eventTime
          }
        }
      }
      error
    }
  }
```

---

### Searching Labs

Perform semantic search across all projects, files, and announcements in the Labs ecosystem.

**GraphQL Query:**

```graphql
query SearchLabs(
  $prompt: String!
  $filters: SearchLabsFilters
  $page: Int
  $perPage: Int
) {
  searchLabs(
    prompt: $prompt
    filters: $filters
    page: $page
    perPage: $perPage
  ) {
    nodes {
      __typename
      ... on SearchLabsFileHit {
        entry {
          lab {
            oclId
            shortname
          }
          path
          file {
            did
            contentType
            accessLevel
            description
            tags
            categories
            downloadUrl
          }
        }
      }
      ... on SearchLabsAnnouncementHit {
        announcement {
          id
          headline
          body
          systemTime
          attachments {
            id
            did
            path
            name
            contentType
            accessLevel
          }
        }
        lab {
          oclId
          shortname
        }
      }
    }
    totalCount
    pageInfo {
      hasNextPage
      hasPreviousPage
      currentPage
      totalPages
    }
  }
}
```

**Parameters:**

| Parameter | Type              | Required | Description                    |
| --------- | ----------------- | -------- | ------------------------------ |
| prompt    | String            | Yes      | Search query text              |
| filters   | SearchLabsFilters | No       | Filter criteria                |
| page      | Int               | No       | Page number (default: 0)       |
| perPage   | Int               | No       | Results per page (default: 10) |

**Available Filters:**

| Filter         | Type      | Description                                           |
| -------------- | --------- | ----------------------------------------------------- |
| byOclIds       | [String!] | Filter by specific lab oclIds                         |
| byTags         | [String!] | Filter files by tags                                  |
| byCategories   | [String!] | Filter files by categories                            |
| byAccessLevels | [String!] | Filter files by access level (PUBLIC, HOLDERS, ADMIN) |
| byKinds        | [String!] | Filter by result type                                 |

**Example - Basic Search:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "query SearchLabs($prompt: String!, $page: Int, $perPage: Int) { searchLabs(prompt: $prompt, page: $page, perPage: $perPage) { nodes { __typename ... on SearchLabsFileHit { entry { lab { oclId shortname } path file { contentType description tags } } } ... on SearchLabsAnnouncementHit { announcement { headline body } lab { shortname } } } totalCount pageInfo { hasNextPage currentPage totalPages } } }",
    "variables": {
      "prompt": "cancer research",
      "page": 0,
      "perPage": 10
    }
  }'
```

**Example - Filtered Search:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "query SearchLabs($prompt: String!, $filters: SearchLabsFilters) { searchLabs(prompt: $prompt, filters: $filters) { nodes { __typename ... on SearchLabsFileHit { entry { path file { tags accessLevel } } } } totalCount } }",
    "variables": {
      "prompt": "experimental data",
      "filters": {
        "byAccessLevels": ["PUBLIC"],
        "byTags": ["research", "validated"]
      }
    }
  }'
```

**Understanding Results:**

Search results are returned as a union type. Use the `__typename` field to determine result type:

- **SearchLabsFileHit**: File search result
  - Access via: `entry.file`
  - Contains: file metadata, tags, categories, download URL
- **SearchLabsAnnouncementHit**: Announcement search result
  - Access via: `announcement`
  - Contains: headline, body, lab reference, **typed attachments** (file objects)

**JavaScript Example:**

```javascript
const searchResults = await fetch(apiUrl, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-api-key": process.env.API_KEY,
    "X-Service-Token": process.env.SERVICE_TOKEN,
  },
  body: JSON.stringify({
    query: `query SearchLabs($prompt: String!) {
      searchLabs(prompt: $prompt) {
        nodes {
          __typename
          ... on SearchLabsFileHit {
            entry {
              path
              file { description tags }
            }
          }
          ... on SearchLabsAnnouncementHit {
            announcement {
              headline
              attachments {
                did
                path
                contentType
                accessLevel
              }
            }
          }
        }
        totalCount
      }
    }`,
    variables: { prompt: "latest results" },
  }),
});

const { nodes, totalCount } = (await searchResults.json()).data.searchLabs;

// Handle different result types
nodes.forEach((node) => {
  if (node.__typename === "SearchLabsFileHit") {
    console.log("File:", node.entry.path);
  } else if (node.__typename === "SearchLabsAnnouncementHit") {
    console.log("Announcement:", node.announcement.headline);
    // NEW: Attachments are now full file objects
    node.announcement.attachments.forEach((file) => {
      console.log("  Attachment:", file.path, file.contentType);
    });
  }
});
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
        "x-api-key": process.env.API_KEY,
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
              isSuccess
              error { message }
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
    if (!initiateResult.data?.initiateCreateOrUpdateFile?.isSuccess) {
      throw new Error(
        initiateResult.data?.initiateCreateOrUpdateFile?.error?.message ||
          "Failed to initiate upload",
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
        "x-api-key": process.env.API_KEY,
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
              isSuccess
              message
              error { message }
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
    if (!finishResult.data?.finishCreateOrUpdateFile?.isSuccess) {
      throw new Error(
        finishResult.data?.finishCreateOrUpdateFile?.error?.message ||
          "Failed to finish upload",
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
API_KEY="your-api-key" SERVICE_TOKEN="your-service-token" WALLET_ADDRESS="0x..." node upload.js data.pdf 0x0101000000000000000000000000000000000000000000000000000000000042
```

---

## Service Token Management

### Obtaining Tokens

Service tokens must be requested from the Molecule team (see [Authentication](#authentication) section above).

Alternatively, a service can obtain a token **self-service** by proving control of its wallet — useful for autonomous agents, bots, and CI/CD pipelines that don't have a browser-based Privy session. This is a two-step flow: fetch the deterministic sign-in message, sign it with the service wallet, then exchange the signature for a token.

**Step 1 — Get the sign-in message (`getServiceSignInMessage`):**

```graphql
query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
  getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) {
    message
  }
}
```

| Parameter     | Type   | Required | Description                                         |
| ------------- | ------ | -------- | --------------------------------------------------- |
| walletAddress | String | Yes      | Wallet address of the service (e.g. an agent's EOA) |
| serviceName   | String | Yes      | Name of the service requesting a token              |

Public query — no authentication required.

**Step 2 — Exchange the signature for a token (`generateServiceToken`):**

Sign the returned `message` with the service wallet, then submit the signature:

```graphql
mutation GenerateServiceToken($serviceName: String!, $walletAddress: String!, $messageSignature: String!, $expiresIn: String) {
  generateServiceToken(
    serviceName: $serviceName
    walletAddress: $walletAddress
    messageSignature: $messageSignature
    expiresIn: $expiresIn
  ) {
    token
    tokenId
    serviceName
    expiresAt
    createdAt
    isSuccess
    message
  }
}
```

| Parameter        | Type   | Required | Description                                                                  |
| ---------------- | ------ | -------- | ---------------------------------------------------------------------------- |
| serviceName      | String | Yes      | Name of the service the token is issued for                                  |
| walletAddress    | String | No\*     | Service wallet address (required together with `messageSignature`)           |
| messageSignature | String | No\*     | Hex-encoded signature of the sign-in message (required with `walletAddress`) |
| expiresIn        | String | No       | Token lifetime (e.g. `"30d"`, `"720h"`)                                      |

\* `walletAddress` and `messageSignature` must be provided together for signature-based issuance. The returned `token` is the JWT to pass as `X-Service-Token` on subsequent requests.

### Extending Token Expiration

You can extend your service token's expiration using the `extendServiceToken` mutation:

```graphql
mutation ExtendServiceToken($tokenId: String!, $expiresIn: String!) {
  extendServiceToken(tokenId: $tokenId, expiresIn: $expiresIn) {
    token
    tokenId
    expiresAt
    isSuccess
    message
  }
}
```

**Parameters:**

| Parameter | Type   | Description                                     |
| --------- | ------ | ----------------------------------------------- |
| tokenId   | String | Token ID provided when token was generated      |
| expiresIn | String | New duration (e.g., `"30d"`, `"720h"`, `"90d"`) |

**Example:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation ExtendServiceToken($tokenId: String!, $expiresIn: String!) { extendServiceToken(tokenId: $tokenId, expiresIn: $expiresIn) { token tokenId expiresAt isSuccess message } }",
    "variables": {
      "tokenId": "your-token-id",
      "expiresIn": "90d"
    }
  }'
```

**Important:** Extension returns a **new JWT token** - update your stored token accordingly.

### Revoking Tokens

Revoke a service token immediately (e.g., if compromised):

```graphql
mutation RevokeServiceToken($tokenId: String!) {
  revokeServiceToken(tokenId: $tokenId) {
    tokenId
    isSuccess
    message
    revokedAt
  }
}
```

**Example:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation RevokeServiceToken($tokenId: String!) { revokeServiceToken(tokenId: $tokenId) { tokenId isSuccess message revokedAt } }",
    "variables": {
      "tokenId": "your-token-id"
    }
  }'
```

---

## Dataroom Utilities

### Get Dataroom Passphrase

Retrieve the Telegram bot passphrase for connecting to a dataroom's notification channel. This passphrase is used to authenticate with the Molecule Telegram bot for receiving real-time updates about dataroom activity.

**GraphQL Mutation:**

```graphql
mutation GetDataRoomPassphrase($oclId: String!) {
  dataRoomPassphrase(oclId: $oclId)
}
```

**Parameters:**

| Parameter | Type   | Required | Description                        |
| --------- | ------ | -------- | ---------------------------------- |
| oclId     | String | Yes      | Canonical 32-byte oclId of the lab |

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation GetDataRoomPassphrase($oclId: String!) { dataRoomPassphrase(oclId: $oclId) }",
    "variables": {
      "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042"
    }
  }'
```

**Success Response:**

```json
{
  "data": {
    "dataRoomPassphrase": "apple-banana-cherry"
  }
}
```

**Use Case:**
- Connect the Molecule Telegram bot to receive notifications about file uploads, announcements, and other dataroom activity
- The passphrase is a 3-word phrase separated by dashes

> **Note**: This is a mutation (not a query) because it may have side effects related to notification channel setup. Requires admin access to the specified lab.

---

## Error Handling

All API responses follow a consistent error format:

### Error Response Structure

```json
{
  "data": {
    "initiateCreateOrUpdateFile": {
      "isSuccess": false,
      "error": {
        "message": "Error description",
        "code": "ERROR_CODE",
        "retryable": true
      }
    }
  }
}
```

### Common Error Codes

| Status Code | Error                 | Description                                                      |
| ----------- | --------------------- | ---------------------------------------------------------------- |
| 401         | Unauthorized          | Missing or invalid service token                                 |
| 403         | Forbidden             | Service token does not have access to the specified lab          |
| 400         | Bad Request           | Invalid parameters (e.g., missing oclId, invalid contentType)    |
| 404         | Not Found             | Lab or dataroom not found                                        |
| 413         | Payload Too Large     | File exceeds size limits                                         |
| 500         | Internal Server Error | Server error - check if retryable and try again                  |

### Troubleshooting

**"Missing service token" error:**

- Ensure `X-Service-Token` header is included in requests
- Verify token is not empty or malformed

**"Service does not have access to lab" error:**

- Verify your wallet address (linked to service token) has admin access to the lab/dataroom
- Check that the oclId format is correct: a 32-byte hex string with `0x` prefix

**"Token expired" error:**

- Request a new token from the Molecule team, or
- Use `extendServiceToken` mutation to extend expiration

**Upload to presigned URL fails:**

- Ensure binary file upload (use `--data-binary` in curl)
- Verify headers match those returned in Step 1
- Check that presigned URL hasn't expired (expires after ~15 minutes)

**"File not found" error (updateFileMetadata, deleteDataRoomFile):**

- Verify the file `ref` (DID) or `path` is correct
- Check that the file exists in the specified dataroom
- Ensure you have access to the dataroom

**"Invalid search filters" error (searchLabs):**

- Verify filter values match expected types (arrays of strings)
- Check that access level values are: PUBLIC, HOLDERS, or ADMIN
- Ensure oclId format is correct if using byOclIds filter

---

## File Requirements & Limits

### Storage Limits

- **Default Limit**: 5GB per lab/project
- **Custom Limits**: Can be increased upon request - contact the Molecule team

### Supported File Types

- All file types are supported
- Common types: PDF, PNG, JPEG, CSV, JSON, ZIP, etc.

### Access Levels

| Level   | Description                                  |
| ------- | -------------------------------------------- |
| PUBLIC  | Visible to anyone with access to the project |
| HOLDERS | Visible only to IP Token (IPT) holders       |
| ADMIN   | Visible only to project administrators       |

### Optional Metadata

Enhance file discoverability with optional metadata:

- **description**: Human-readable description of the file
- **tags**: Array of tags for categorization (e.g., `["research", "q4-2024"]`)
- **categories**: Array of categories for organization (e.g., `["data", "results"]`)
- **contentText**: Searchable text content for full-text search

---

## Advanced: Encrypted File Upload

For files requiring client-side encryption, pass `encryption: true` to `initiateCreateOrUpdateFile` and include an `encryptionMetadata` object on `finishCreateOrUpdateFile`. The full end-to-end model — key wrapping, on-chain access conditions, and condition-gated decryption — is documented on the [Data Privacy & Access](../core-concepts/data/data-privacy-and-access.md) page.

### Initiate with encryption

`initiateCreateOrUpdateFile(..., encryption: true)` returns `plaintextDEK`, `encryptedDEK`, and `encryptionSystem` in addition to the upload URL. The client uses `plaintextDEK` to AES-256-GCM encrypt the file locally (Web Crypto `SubtleCrypto`), then wipes it from memory.

### Encryption Metadata Parameter (Onchain-Verified Envelope Encryption, current default)

```graphql
$encryptionMetadata: EncryptionMetadataInput
```

```json
{
  "encryptionMetadata": {
    "encryptionSystem": "<echo value returned by initiateCreateOrUpdateFile>",
    "encryptedDek": "BASE64_WRAPPED_DEK",
    "iv": "BASE64_AES_GCM_IV",
    "contentHash": "sha256-...",
    "accessControlConditions": "[{...}]",
    "encryptedBy": "0x1234567890123456789012345678901234567890",
    "encryptedAt": "2026-01-15T10:30:00.000Z"
  }
}
```

`encryptionSystem` is **backend-set** — clients must echo the value returned on `initiateCreateOrUpdateFile` rather than hardcode it. This keeps the roadmap rollover to BLS threshold key custody transparent to existing integrations.

#### `accessControlConditions` — gating decryption by role

`accessControlConditions` is a JSON-stringified array of `EvmContractCondition` predicates joined by `BooleanCondition` separators (`and` / `or`). The backend evaluates each predicate against live chain state at decrypt time via viem `readContract`, short-circuits booleans, and fails closed on RPC error. To gate decryption on _LabNFT owner OR active Contributor OR active Viewer_, OR `AccessResolver.isAuthorizedSignerForTba(:userAddress, tba)` against `AccessResolver.hasRole(oclId, :userAddress, ROLE_VIEWER)` — the role-hierarchy collapses Contributor + Viewer into one check on the canonical chain (Base).

The placeholder `:userAddress` in `functionParams` is substituted with the authenticated caller's wallet at evaluate time. The full `EvmContractCondition` JSON shape, the worked OR-composite example, and condition-evaluator semantics are documented on the [Data Privacy & Access](../core-concepts/data/data-privacy-and-access.md#worked-example-encrypt-for-owner-or-contributor-or-viewer) page.

#### Role Management (on-chain, off this API surface)

Role grants are **on-chain transactions on the `AccessResolver` contract**, not Labs API mutations. Lab owners (and active Contributors, for the Viewer slot) call `grantRole(oclId, account, role, expiry, isAgent)` / `revokeRole(oclId, account)` directly via viem / ethers / Safe. The Labs API only _consumes_ role state at decrypt time through the `accessControlConditions` evaluator. See [Roles & Permissions](../core-concepts/roles-and-permissions.md) for the capability matrix, grant lifecycle (expiry, `isAgent`), and the [`AccessResolver` reference](../references/contracts/accessresolver.md) for the on-chain interface.

### Encryption Metadata Parameter (Lit Protocol, legacy)

> **Legacy.** Lit Protocol is retained read-only for files encrypted before the migration to Onchain-Verified Envelope Encryption. New uploads should use the metadata shape above. Files with the shape below continue to decrypt through the Lit SDK.

```json
{
  "encryptionMetadata": {
    "dataToEncryptHash": "0xabc123...",
    "accessControlConditions": "[{...}]",
    "encryptedBy": "0x1234567890123456789012345678901234567890",
    "encryptedAt": "2024-01-15T10:30:00.000Z",
    "chain": "ethereum",
    "litSdkVersion": "3.0.0",
    "litNetwork": "datil-test",
    "templateName": "standard-access-control",
    "contractVersion": "1.0.0"
  }
}
```

**When to Use Encryption:**

- Sensitive research data requiring access control
- Compliance requirements for data protection
- Conditional access based on token ownership or lab role

---

## Best Practices

### Token Security

- **Never commit tokens** to version control (add to `.gitignore`)
- **Use environment variables** to store tokens
- **Rotate tokens regularly** (quarterly recommended)
- **Use secrets management systems** in production (AWS Secrets Manager, HashiCorp Vault, etc.)
- **Revoke immediately** if a token is compromised

### Storage Management

- Monitor your 5GB storage limit per project
- Organize files with meaningful names and metadata
- Use categories and tags for easy file discovery
- Clean up old or unnecessary files regularly

### Metadata Best Practices

- **Use descriptive tags**: `["experiment-1", "2024-q4", "preliminary"]`
- **Organize with categories**: `["raw-data", "analysis", "results"]`
- **Add descriptions**: Help collaborators understand file contents
- **Include searchable text** (`contentText`): Enables full-text search via `searchLabs`
- **Update metadata as needed**: Use `updateFileMetadata` to refine tags and descriptions without re-uploading files

### Access Control

- Use `ADMIN` for sensitive internal documents
- Use `HOLDERS` for IPT holder-exclusive content
- Use `PUBLIC` for community-facing data
- Review access levels regularly as your project evolves

### Search and Discovery

- **Use contentText**: Populate `contentText` field when uploading files to enable full-text search
- **Tag consistently**: Use consistent tag names across files for better filtering
- **Filter strategically**: Combine filters (tags + access levels) to narrow search results
- **Test search queries**: Use `searchLabs` to verify your files are discoverable

---

## Lab Members

### List Lab Members

Return the active members of a lab (owner, contributors, viewers), sourced from the indexed `ocl_user` table. Expired grants are excluded.

> **Public query** — only an API Key is required. The same data is also exposed on the public `Lab` / `LabRef.members` field.

```graphql
query ListLabMembers($oclId: String!) {
  listLabMembers(oclId: $oclId) {
    isSuccess
    message
    members {
      walletAddress
      role
      source
      expiry
      isAgent
      grantedAt
    }
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter | Type   | Required | Description                        |
| --------- | ------ | -------- | ---------------------------------- |
| oclId     | String | Yes      | Canonical 32-byte oclId of the lab |

**Member fields:**

| Field         | Type            | Description                                                                                      |
| ------------- | --------------- | ------------------------------------------------------------------------------------------------ |
| walletAddress | String          | Lowercased wallet address of the member                                                          |
| role          | LabMemberRole   | Effective role: `OWNER`, `CONTRIBUTOR`, or `VIEWER`                                              |
| source        | LabMemberSource | Row that defines the membership: `ONCHAIN_EVENT`, `MULTISIG_RESOLUTION`, `ACCESS_CONTRACT`, or `ACCESS_RESOLVER_EVENT` |
| expiry        | String          | Unix-seconds expiry as a decimal string; `null` means the grant is permanent                     |
| isAgent       | Boolean         | True if the member is an agent identity (surfaced for UI; not used for authorization)            |
| grantedAt     | String          | ISO-8601 timestamp the row was first persisted                                                   |

---

## File Categories & Tags

### Get File Categories and Tags

Return the valid file categories and their tags from the CMS. Use these when tagging or categorizing files. **Public query** — no authentication required.

```graphql
query FileCategoriesAndTags {
  fileCategoriesAndTags {
    isSuccess
    data {
      name
      tags
    }
    error {
      message
      code
      retryable
    }
  }
}
```

Each entry in `data` is a `FileCategory` with a `name` and its list of allowed `tags`.

---

## LabNFT Metadata

### Update LabNFT Metadata

Partial update of the LabNFT display metadata (`name`, `description`, `image`, `externalUrl`). Omitted fields are left unchanged; an explicit `null` clears a field.

> **Authorization**: Restricted to the OCL admin (LabNFT owner + multisig signers).

```graphql
mutation UpdateLabNftMetadata($oclId: String!, $input: UpdateLabNftMetadataInput!) {
  updateLabNftMetadata(oclId: $oclId, input: $input) {
    isSuccess
    oclId
    message
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter | Type                     | Required | Description                        |
| --------- | ------------------------ | -------- | ---------------------------------- |
| oclId     | String                   | Yes      | Canonical 32-byte oclId of the lab |
| input     | UpdateLabNftMetadataInput| Yes      | Patch object (all fields optional) |

`UpdateLabNftMetadataInput` fields (all optional): `name`, `description`, `image`, `externalUrl`.

### Generate LabNFT Image Upload URL

Generate a single-use presigned PUT URL to which the OCL admin uploads a LabNFT display image. `contentType` must be one of `image/jpeg`, `image/png`, `image/webp`, `image/gif`, or `image/svg+xml`. The public URL is patched onto the lab asynchronously by the image processor once the object lands in S3.

> **Authorization**: Restricted to the OCL admin (LabNFT owner + multisig signers).

```graphql
mutation GenerateLabImageUploadUrl($oclId: String!, $contentType: String!) {
  generateLabImageUploadUrl(oclId: $oclId, contentType: $contentType) {
    uploadUrl
    key
    expiresAt
    isSuccess
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter   | Type   | Required | Description                                                                             |
| ----------- | ------ | -------- | --------------------------------------------------------------------------------------- |
| oclId       | String | Yes      | Canonical 32-byte oclId of the lab                                                      |
| contentType | String | Yes      | Image MIME type (`image/jpeg`, `image/png`, `image/webp`, `image/gif`, `image/svg+xml`) |

---

## Legal Agreements

The legal-agreement flow is three operations: fetch the populated template + `contentHash`, sign it as an EIP-712 typed-data payload, then submit the signature. The backend regenerates and verifies the document server-side and stores the signed artifact in the lab's data room.

> `type` is a `LegalAgreementType` enum. Current value: `ASSIGNMENT_AGREEMENT`.

### Get Legal Agreement Template (query)

Return the populated agreement the given wallet is expected to sign, plus the `contentHash` for the EIP-712 envelope. Read-only.

```graphql
query LegalAgreementTemplate(
  $oclId: String!
  $type: LegalAgreementType!
  $walletAddress: String!
  $signerName: String
  $entity: String
  $title: String
) {
  legalAgreementTemplate(
    oclId: $oclId
    type: $type
    walletAddress: $walletAddress
    signerName: $signerName
    entity: $entity
    title: $title
  ) {
    isSuccess
    agreement
    contentHash
    templateVersion
    agreementType
    issuedAt
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter     | Type              | Required | Description                                                                                   |
| ------------- | ----------------- | -------- | --------------------------------------------------------------------------------------------- |
| oclId         | String            | Yes      | Canonical 32-byte oclId of the lab                                                            |
| type          | LegalAgreementType | Yes      | Agreement type (`ASSIGNMENT_AGREEMENT`)                                                        |
| walletAddress | String            | Yes      | Intended signer. On the user-auth path this must equal the authenticated wallet               |
| signerName    | String            | No       | Signer identity (natural person). Echoed VERBATIM into `signLegalAgreement`; covered by `contentHash` |
| entity        | String            | No       | Signing entity, if the signer represents an organization                                      |
| title         | String            | No       | Signer title                                                                                  |

> `agreement` is for display only — the client must NOT re-serialize or re-hash it; sign over `contentHash` as given, and echo `issuedAt`, `signerName`, `entity`, and `title` back unchanged at sign time or the regenerated hash won't match.

### Check Legal Agreement Status (query)

Whether (and at which template versions) the agreement has been signed for the lab. **Public** — same surface as `labs`. Also available inline as `Lab` / `LabRef.legalAgreementStatus`.

```graphql
query LegalAgreementStatus($oclId: String!, $type: LegalAgreementType!) {
  legalAgreementStatus(oclId: $oclId, type: $type) {
    isSuccess
    signed
    isCurrentVersionSigned
    currentTemplateVersion
    signedVersions {
      templateVersion
      path
      signer
      signedAt
      contentHash
      issuedAt
      signature
    }
    error {
      message
      code
      retryable
    }
  }
}
```

Check `isSuccess` before reading the other fields. `signed` is true if any version has been signed; `isCurrentVersionSigned` reflects the current template version (the FE routes the signing flow on this). The `signedVersions` enrichment is fetched lazily, only when its sub-fields are selected.

### Sign Legal Agreement (mutation)

Record acceptance of a legal agreement. The backend regenerates the document from `(type, oclId, walletAddress, issuedAt)`, verifies the EIP-712 signature and LabNFT ownership, embeds the signature into a self-verifying envelope, and uploads it to the lab's data room. Rejects if the current template version is already signed.

```graphql
mutation SignLegalAgreement($input: SignLegalAgreementInput!) {
  signLegalAgreement(input: $input) {
    isSuccess
    oclId
    path
    contentHash
    templateVersion
    datasetId
    version
    message
    error {
      message
      code
      retryable
    }
  }
}
```

**`SignLegalAgreementInput` fields:**

| Field         | Type              | Required | Description                                                                                    |
| ------------- | ----------------- | -------- | ---------------------------------------------------------------------------------------------- |
| oclId         | String            | Yes      | Canonical 32-byte oclId of the lab                                                             |
| type          | LegalAgreementType| Yes      | Agreement type (`ASSIGNMENT_AGREEMENT`)                                                         |
| walletAddress | String            | Yes      | Signer's wallet. Must equal the EIP-712 signer, the lab's current LabNFT owner, and (user path) the authenticated wallet |
| signature     | String            | Yes      | EIP-712 signature (0x…) over the `LegalAgreementAcceptance` typed data                          |
| issuedAt      | AWSTimestamp      | Yes      | Echoed VERBATIM from `legalAgreementTemplate` — used to regenerate the document                |
| signerName    | String            | No       | Echoed VERBATIM from the template call (covered by `contentHash`)                              |
| entity        | String            | No       | Echoed verbatim; see `signerName`                                                              |
| title         | String            | No       | Echoed verbatim; see `signerName`                                                              |

---

## DID Linking

### Get DID Link Status

Public read-only snapshot of DID-linking state for an OCL. DID-linking runs automatically in the background after `createLab`; this query is for diagnostic and support visibility. No authentication required.

```graphql
query GetDidLinkStatus($oclId: String!) {
  getDidLinkStatus(oclId: $oclId) {
    isSuccess
    message
    didLinkStatus {
      oclId
      status
      userOpHash
      txHash
      accountDid
      dataRoomDid
      linkedDidCount
      attempts
      updatedAt
    }
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter | Type   | Required | Description                        |
| --------- | ------ | -------- | ---------------------------------- |
| oclId     | String | Yes      | Canonical 32-byte oclId of the lab |

`status` is a `DidLinkingStatus`: `PENDING`, `SUBMITTED`, `LINKED`, or `FAILED` (`null` before the first linking attempt). `linkedDidCount` reflects the number of active on-chain DID links observed by the event indexer.

---

## On-Chain Activity

### On-Chain Activity Feed

Return the on-chain event feed for an OCL or a wallet. Exactly one of `oclId` / `wallet` must be supplied. Paginate with a cursor of the form `"<block_number>:<log_index>"` — pass the last row's `id` to fetch the next page.

```graphql
query OnChainActivity($oclId: String, $wallet: String, $limit: Int, $cursor: String) {
  onChainActivity(oclId: $oclId, wallet: $wallet, limit: $limit, cursor: $cursor) {
    id
    chainId
    contractAddress
    contractName
    eventName
    blockNumber
    blockTimestamp
    txHash
    logIndex
    args
  }
}
```

**Parameters:**

| Parameter | Type   | Required | Description                                                       |
| --------- | ------ | -------- | ----------------------------------------------------------------- |
| oclId     | String | No\*     | Canonical 32-byte oclId of the lab                                |
| wallet    | String | No\*     | Wallet address to filter events by                                |
| limit     | Int    | No       | Max rows to return (default: 50)                                  |
| cursor    | String | No       | Pagination cursor `"<block_number>:<log_index>"` (last row's `id`) |

\* Provide exactly one of `oclId` or `wallet`. `contractName` is one of `accessresolver`, `ocl`, `ipnft`, `ipt`, or `bio-agent`. `args` is a JSON object of the decoded event arguments (BigInts as decimal strings, addresses lowercased).

---

## Data Encryption Keys

### Generate a Data Encryption Key

Generate a standalone data encryption key (DEK) for client-side encryption outside the file-upload flow. Returns both the plaintext DEK (used to encrypt data locally, then wiped) and the KMS-encrypted DEK (stored alongside the ciphertext). Requires authentication (Privy user or service token). See [Advanced: Encrypted File Upload](#advanced-encrypted-file-upload) for the file-upload encryption path.

```graphql
mutation GenerateDataEncryptionKey {
  generateDataEncryptionKey {
    isSuccess
    plaintextDEK
    encryptedDek
    encryptionSystem
    error {
      message
      code
      retryable
    }
  }
}
```

| Field            | Type    | Description                                                |
| ---------------- | ------- | ---------------------------------------------------------- |
| plaintextDEK     | String  | Base64-encoded plaintext DEK (only present on success)     |
| encryptedDek     | String  | Base64-encoded KMS-encrypted DEK (only present on success) |
| encryptionSystem | String  | Encryption system used (always `"kms"`)                    |

---

## Deprecated & Renamed Operations

The legacy `*V2` operations and the pre-OCL naming have been **removed**. The current API is `oclId`-based. If you are migrating from an older integration, use the current names below.

### Renamed queries

| Legacy (removed)                                   | Current                      | Notes                                                        |
| -------------------------------------------------- | ---------------------------- | ------------------------------------------------------------ |
| `projects` / `projectsV2`                          | `labs`                       | Same paginated shape; adds an optional `role` filter         |
| `projectWithDataRoomAndFiles` / `…V2`              | `labWithDataRoomAndFiles`    | Look up by `oclId` (or `shortname`) instead of `ipnftUid`    |
| `dataRoomFileV2`                                   | `dataRoomFile`               | Identified by `oclId` + `path`                               |
| `projectActivity` / `projectActivityV2`            | `labActivity`                | —                                                            |
| `activitiesV2`                                     | `activities`                 | —                                                            |
| `projectAnnouncementsV2` / `projectAnnouncementV2` | `labActivity` / `activities` | Removed — use the `filter: ANNOUNCEMENT` argument            |

### Renamed mutations

| Legacy (removed)               | Current                      | Notes                                                                  |
| ------------------------------ | ---------------------------- | ---------------------------------------------------------------------- |
| `createProject`                | `createLab`                  | Now takes `input: { oclId }` instead of `ipnftSymbol` / `ipnftTokenId` |
| `initiateCreateOrUpdateFileV2` | `initiateCreateOrUpdateFile` | —                                                                      |
| `finishCreateOrUpdateFileV2`   | `finishCreateOrUpdateFile`   | —                                                                      |
| `createAnnouncementV2`         | `createAnnouncement`         | Takes `oclId`; the legacy `moleculeAccessLevel` param was removed      |
| `updateFileMetadataV2`         | `updateFileMetadata`         | —                                                                      |
| `deleteDataRoomFileV2`         | `deleteDataRoomFile`         | —                                                                      |

### Renamed fields

Top-level identifiers on `Lab` / `LabRef` were renamed away from the legacy IP-NFT naming:

| Legacy field   | Current field       | Notes                                                       |
| -------------- | ------------------- | ----------------------------------------------------------- |
| `ipnftUid`     | `oclId`             | Now a 32-byte hex string (`0x…`), not `<address>_<tokenId>` |
| `ipnftSymbol`  | `shortname`         | Human-readable slug derived from the lab name               |
| `ipnftAddress` | `labAccountAddress` | ERC-6551 token-bound account address                        |
| `ipnftTokenId` | `labNftTokenId`     | LabNFT tokenId (decimal string)                             |

> The linked legacy IP-NFT (for labs migrated from one) is still available as the nested `ipnft` object and the `ipnftId` field on `Lab` / `LabRef`.

---
## Getting Support

If you encounter any issues or have questions about the Programmatic File Upload API:

1. Check this documentation and [troubleshooting section](#troubleshooting)
2. Review the [complete example](#complete-example) for implementation guidance
3. Join our [Discord community](https://t.co/L0VEiy4Bjk) for support
4. Contact the Molecule Labs development team directly

---

_Last updated: July 2026_
