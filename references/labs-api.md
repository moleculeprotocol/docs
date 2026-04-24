---
icon: landmark-flag
---

# Labs API

## Labs API Reference

The Labs API provides programmatic access to Molecule Labs — secure data rooms for managing research files, announcements, and intellectual property. Utilize this API to automate file uploads, integrate with CI/CD pipelines, and build applications on the Molecule ecosystem.

### Base URLs

<table><thead><tr><th width="165.93359375">Environment</th><th>Endpoint</th></tr></thead><tbody><tr><td>Production</td><td><code>https://production.graphql.api.molecule.xyz/graphql</code></td></tr><tr><td>Staging</td><td><code>https://staging.graphql.api.molecule.xyz/graphql</code></td></tr></tbody></table>

### Authentication

The API uses a two-tier authentication model:

<table><thead><tr><th width="172.20703125">Operation Type</th><th>Required Headers</th></tr></thead><tbody><tr><td>Queries (read)</td><td><code>x-api-key</code></td></tr><tr><td>Mutations (write)</td><td><code>x-api-key + X-Service-Token</code></td></tr></tbody></table>

#### Headers

**Read operations**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
-H "Content-Type: application/json" \
-H "x-api-key: YOUR_API_KEY" \
-d '{"query": "{ projectsV2 { nodes { ipnftUid } } }"}'
```

**Write operations**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
-H "Content-Type: application/json" \
-H "x-api-key: YOUR_API_KEY" \
-H "X-Service-Token: YOUR_SERVICE_TOKEN" \
-d '{"query": "mutation { ... }"}'
```

#### Obtaining Credentials

Contact the Molecule team via [Discord](https://discord.gg/molecule) with:

* Wallet address
* Use case description
* Target Lab (ipnftUid)
* Desired token expiration

#### Token Management

**Extend token expiration**

```graphql
mutation {
  extendServiceToken(tokenId: "tok_xxx", expiresIn: "30d") {
    token
    expiresAt
    isSuccess
  }
}
```

**Revoke token**

```graphql
mutation {
  revokeServiceToken(tokenId: "tok_xxx") {
    isSuccess
    revokedAt
  }
}
```

### Core Concepts

#### IPNFT UID Format

All operations reference Labs by their ipnftUid: `{contractAddress}_{tokenId}`

* Example: `0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_9`

#### Access Levels

| Level   | Who Can Access                                    |
| ------- | ------------------------------------------------- |
| PUBLIC  | Anyone                                            |
| HOLDERS | IP Token holders (requires on-chain verification) |
| ADMIN   | Lab administrators only                           |

**Note:** `HOLDERS` access level is accepted by the API but token-gating is not currently enforced. Files marked as `HOLDERS` behave the same as `ADMIN` until token-gated access is implemented.

#### File References

* **path**: Human-readable file path (e.g., `research/data.csv`)
* **ref**: Kamu dataset ID (DID) for versioning operations

#### Pagination

All list queries support cursor-based pagination:

| Parameter | Type | Default | Max | Notes          |
| --------- | ---- | ------- | --- | -------------- |
| `page`    | Int  | 0       | —   | 0-indexed      |
| `perPage` | Int  | 10      | 100 | Items per page |

Example:

```graphql
projectsV2(page: 0, perPage: 50) { ... }     
```

### URL Lifetimes

{% code overflow="wrap" %}
```markdown
 ### URL Lifetimes                                                                                         
                                                                                                           
 | URL Type | Expiry | Source |                                                                            
 |----------|--------|--------|                                                                            
 | Download URL (`url` field) | ~15 minutes | S3 presigned |                                               
 | Upload URL (from `initiateCreateOrUpdateFileV2`) | ~60 minutes | S3 presigned |                         
                                                                                                           
 Always fetch fresh download URLs before sharing or downloading. Do not cache these URLs.                  
```
{% endcode %}

### Queries

#### List Projects

Retrieve all Labs with pagination.

```graphql
query {
  projectsV2(
    walletAddress: "0x..." # Optional: filter by admin wallet
    page: 0
    perPage: 20
  ) {
    nodes {
      ipnftUid
      ipnftSymbol
      ipnftAddress
      ipnftTokenId
      account { accountName }
      dataRoom {
        id
        alias
        description
        createdAt
        files { path accessLevel }
      }
    }
    totalCount
    pageInfo {
      hasNextPage
      currentPage
      totalPages
    }
  }
}
```

#### Get Project with Files

Retrieve a specific Lab with all data room files.

```graphql
query {
  projectWithDataRoomAndFilesV2(ipnftUid: "0xcaD...Fc1_9") {
    ipnftUid
    ipnftSymbol
    dataRoom {
      id
      alias
      description
      telegramChatId
      files {
        id
        path
        version
        contentType
        contentHash
        accessLevel
        createdAt
        updatedAt
        createdBy
        description
        tags
        categories
        downloadUrl
        downloadHeaders { key value }
        downloadUrlExpiry
        encryptionMetadata {
          dataToEncryptHash
          accessControlConditions
          encryptedBy
          encryptedAt
          chain
          litNetwork
        }
      }
    }
    announcements {
      id
      headline
      body
      changeBy
      systemTime
    }
  }
}
```

#### Get File by Path

Retrieve a specific file with download URL.

```graphql
query {
  dataRoomFileV2(
    ipnftUid: "0xcaD...Fc1_9"
    path: "research/experiment-001.pdf"
  ) {
    id
    path
    version
    contentType
    contentHash
    accessLevel
    downloadUrl
    downloadHeaders { key value }
    downloadUrlExpiry
    encryptionMetadata {
      dataToEncryptHash
      accessControlConditions
      chain
      litNetwork
    }
  }
}
```

#### Project Activity

Retrieve timeline of file events and announcements.

```graphql
query {
  projectActivityV2(
    ipnftUid: "0xcaD...Fc1_9"
    page: 0
    perPage: 20
  ) {
    nodes {
      ... on ProjectEventFileAddedV2 {
        entry { ref }
      }
      ... on ProjectEventFileUpdatedV2 {
        entry { ref }
      }
      ... on ProjectEventFileRemovedV2 {
        entry { ref }
      }
      ... on ProjectEventAnnouncementV2 {
        announcement {
          id
          headline
          body
          attachments { path downloadUrl }
        }
      }
    }
    pageInfo { hasNextPage totalPages }
  }
}
```

#### Project Announcements

Retrieve announcements only (more efficient than activity query).

```graphql
query {
  projectAnnouncementsV2(
    ipnftUid: "0xcaD...Fc1_9"
    page: 0
    perPage: 20
  ) {
    nodes {
      id
      headline
      body
      changeBy
      systemTime
      attachments {
        path
        downloadUrl
        accessLevel
      }
    }
    totalCount
    pageInfo { hasNextPage }
  }
}
```

#### Search Labs

Semantic search across all Labs, files, and announcements.

```graphql
query {
  searchLabs(
    prompt: "longevity research gene therapy"
    filters: {
      byIpnftUids: ["0xcaD...Fc1_9"]
      byTags: ["genomics", "clinical"]
      byCategories: ["research-data"]
      byAccessLevels: ["PUBLIC", "HOLDERS"]
      byKinds: ["file", "announcement"]
    }
    page: 0
    perPage: 10
  ) {
    nodes {
      ... on SearchLabsFileHit {
        entry {
          path
          ref
          file { contentType accessLevel downloadUrl }
          project { ipnftUid ipnftSymbol }
        }
      }
      ... on SearchLabsAnnouncementHit {
        announcement { headline body }
        project { ipnftUid ipnftSymbol }
      }
    }
    totalCount
    pageInfo { hasNextPage }
  }
}
```

#### Global Activity Feed

Retrieve announcements across all Labs.

```graphql
query {
  activitiesV2(page: 0, perPage: 20) {
    isSuccess
    announcements {
      id
      headline
      body
      project { ipnftUid ipnftSymbol }
      systemTime
    }
    error
  }
}
```

### Mutations

#### File Upload (3-Step Flow)

**Step 1: Initiate Upload**

```graphql
mutation {
  initiateCreateOrUpdateFileV2(
    ipnftUid: "0xcaD...Fc1_9"
    contentType: "application/pdf"
    contentLength: 1048576 # bytes
  ) {
    uploadToken
    uploadUrl
    uploadUrlExpiry
    method
    headers { key value }
    isSuccess
    error { message code retryable }
  }
}
```

**Step 2: Upload to Storage**

```bash
curl -X PUT "https://s3.amazonaws.com/..." \
-H "Content-Type: application/pdf" \
-H "x-amz-meta-token: ..." \
--data-binary @file.pdf
```

**Step 3: Finalize Upload**

For new files — use path:

```graphql
mutation {
  finishCreateOrUpdateFileV2(
    ipnftUid: "0xcaD...Fc1_9"
    uploadToken: "tok_xxx"
    path: "research/experiment-001.pdf"
    accessLevel: "ADMIN"
    changeBy: "0xYourWalletAddress"
    description: "Initial experimental results"
    tags: ["experiment", "q1-2025"]
    categories: ["research-data"]
    contentText: "Searchable text content..."
  ) {
    datasetId
    contentHash
    version
    isSuccess
    message
    error { message code retryable }
  }
}
```

For file versions — use ref:

```graphql
mutation {
  finishCreateOrUpdateFileV2(
    ipnftUid: "0xcaD...Fc1_9"
    uploadToken: "tok_xxx"
    ref: "did:kamu:existing-file-id"
    accessLevel: "ADMIN"
    changeBy: "0xYourWalletAddress"
    description: "Updated with Q2 data"
  ) {
    datasetId
    contentHash
    version
    isSuccess
  }
}
```

#### Encrypted File Upload

For confidential files, include `encryptionMetadata` on `finishCreateOrUpdateFileV2`. New uploads use Molecule's Onchain-Verified Envelope Encryption — `encryptionSystem` is backend-set, so clients should echo the value returned from `initiateCreateOrUpdateFileV2` rather than hardcode it. The example below shows the legacy Lit Protocol metadata shape, which is still accepted for files encrypted before the migration. See [Data Privacy & Access](../core-concepts/data/data-privacy-and-access.md) for the current flow and metadata fields.

{% code overflow="wrap" %}
```graphql
mutation {
  finishCreateOrUpdateFileV2(
    ipnftUid: "0xcaD...Fc1_9"
    uploadToken: "tok_xxx"
    path: "confidential/patient-data.csv"
    accessLevel: "ADMIN"
    changeBy: "0xYourWalletAddress"
    encryptionMetadata: {
      dataToEncryptHash: "0x1234..."
      accessControlConditions: "[{"contractAddress":"0xcaD...","functionName":"canRead",...}]"
      encryptedBy: "0xYourWalletAddress"
      encryptedAt: "2025-01-25T12:00:00Z"
      chain: "base"
      litSdkVersion: "7.0.0"
      litNetwork: "datil"
      templateName: "ipnft_read"
      contractVersion: "1.0.0"
    }
  ) {
    datasetId
    contentHash
    isSuccess
  }
}
```
{% endcode %}

#### Update File Metadata

Modify metadata without creating a new version.

```graphql
mutation {
  updateFileMetadataV2(
    ipnftUid: "0xcaD...Fc1_9"
    ref: "did:kamu:file-id"
    accessLevel: "HOLDERS"
    description: "Updated description"
    tags: ["updated", "reviewed"]
    categories: ["final-results"]
    contentText: "New searchable content"
  ) {
    ref
    isSuccess
    message
    error { message code }
  }
}
```

#### Delete File

Permanently remove a file from the data room.

```graphql
mutation {
  deleteDataRoomFileV2(
    ipnftUid: "0xcaD...Fc1_9"
    path: "deprecated/old-file.pdf"
    changeBy: "0xYourWalletAddress"
  ) {
    ipnftUid
    filePath
    isSuccess
    error { message code }
  }
}
```

#### Create Announcement

Share updates with Lab followers.

```graphql
mutation {
  createAnnouncementV2(
    ipnftUid: "0xcaD...Fc1_9"
    headline: "Phase 2 Clinical Trial Results"
    body: "We are excited to announce..."
    attachments: ["did:kamu:file-1", "did:kamu:file-2"]
  ) {
    isSuccess
    message
    error { message code }
  }
}
```

#### Create Project

Create a new Lab for an IP-NFT you own.

```graphql
mutation {
  createProject(input: {
    ipnftSymbol: "GENE"
    ipnftTokenId: "42"
  }) {
    isSuccess
    message
    project {
      ipnftUid
      ipnftSymbol
    }
    error { message code }
  }
}
```

#### Telegram Integration

Get passphrase for connecting Telegram bot to Lab.

```graphql
mutation {
  dataRoomPassphrase(ipnftUid: "0xcaD...Fc1_9")
}
```

### Types Reference

#### DataRoom File

```graphql
type DataRoomFile {
  id: ID!
  did: String!
  path: String!
  name: String
  version: Int
  contentType: String!
  contentHash: String
  accessLevel: DataRoomAccessLevel! # PUBLIC | HOLDERS | ADMIN
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime
  createdBy: String
  changeBy: String
  description: String
  tags: [String!]
  categories: [String!]
  contentText: String
  downloadUrl: String
  downloadHeaders: [Header!]
  downloadUrlExpiry: AWSDateTime
  encryptionMetadata: EncryptionMetadata
}
```

#### Encryption Metadata

```graphql
type EncryptionMetadata {
  dataToEncryptHash: String! # Hash from Lit encryption
  accessControlConditions: AWSJSON! # Lit access conditions
  encryptedBy: String! # Wallet that encrypted
  encryptedAt: AWSDateTime! # When encrypted
  chain: String! # Blockchain network
  litSdkVersion: String! # Lit SDK version
  litNetwork: String! # datil | datil-test
  templateName: String! # Access control template
  contractVersion: String! # Contract version
}
```

#### ProjectV2

```graphql
type ProjectV2 {
  ipnftUid: String!
  ipnftSymbol: String!
  ipnftAddress: String!
  ipnftTokenId: String!
  systemTime: AWSDateTime!
  eventTime: AWSDateTime!
  account: ProjectAccount!
  dataRoom: DataRoom!
  announcements: [AnnouncementV2]!
}
```

#### InternalLabsError

```graphql
type InternalLabsError {
  message: String # Human-readable description
  code: String # Machine-readable code
  retryable: Boolean # Whether operation can be retried
}
```

### Error Handling

#### Error Response Format

```json
{
  "data": {
    "finishCreateOrUpdateFileV2": {
      "isSuccess": false,
      "error": {
        "message": "File path already exists",
        "code": "FILE_EXISTS",
        "retryable": false
      }
    }
  }
}
```

#### Error Codes

<table><thead><tr><th width="217.87890625">Code</th><th width="99.859375">HTTP</th><th width="253.15625">Description</th><th>Retryable</th></tr></thead><tbody><tr><td>UNAUTHORIZED</td><td>401</td><td>Invalid or missing API key</td><td>No</td></tr><tr><td>FORBIDDEN</td><td>403</td><td>Valid credentials but insufficient permissions</td><td>No</td></tr><tr><td>NOT_FOUND</td><td>404</td><td>Lab or file does not exist</td><td>No</td></tr><tr><td>FILE_EXISTS</td><td>400</td><td>File path already exists (use ref for versioning)</td><td>No</td></tr><tr><td>INVALID_INPUT</td><td>400</td><td>Malformed request parameters</td><td>No</td></tr><tr><td>UPLOAD_EXPIRED</td><td>400</td><td>Upload token has expired</td><td>Yes (re-initiate)</td></tr><tr><td>PAYLOAD_TOO_LARGE</td><td>413</td><td>File exceeds size limit</td><td>No</td></tr><tr><td>RATE_LIMITED</td><td>429</td><td>Too many requests</td><td>Yes (with backoff)</td></tr><tr><td>INTERNAL_ERROR</td><td>500</td><td>Server error</td><td>Yes</td></tr><tr><td>KAMU_ERROR</td><td>502</td><td>Upstream storage error</td><td>Yes</td></tr></tbody></table>

### Rate Limits

| Tier       | Queries/min | Mutations/min | Max File Size | Storage/Lab |
| ---------- | ----------- | ------------- | ------------- | ----------- |
| Standard   | 60          | 20            | 5 GB          | 5 GB        |
| Pro        | 300         | 100           | 10 GB         | 50 GB       |
| Enterprise | Unlimited   | 500           | 50 GB         | Unlimited   |

Rate limit headers:

* X-RateLimit-Limit: 60
* X-RateLimit-Remaining: 45
* X-RateLimit-Reset: 1706198400

### SDK

#### Installation

```bash
npm install @moleculexyz/client-sdk
```

#### Usage

```javascript
import { DesciSdk } from '@moleculexyz/client-sdk';

const sdk = new DesciSdk({
  apiKey: process.env.MOLECULE_API_KEY,
  serviceToken: process.env.MOLECULE_SERVICE_TOKEN,
  environment: 'production'
});

// List projects
const projects = await sdk.labs.getProjects({ page: 0, perPage: 20 });

// Upload file
const result = await sdk.labs.uploadFile({
  ipnftUid: '0xcaD...Fc1_9',
  file: buffer,
  path: 'research/data.csv',
  accessLevel: 'ADMIN',
  changeBy: '0xYourWallet'
});

// Search
const hits = await sdk.labs.search({
  prompt: 'gene therapy results',
  filters: { byAccessLevels: ['PUBLIC'] }
});
```

#### Complete Example

```javascript
import { readFileSync } from 'fs';

const API_URL = 'https://production.graphql.api.molecule.xyz/graphql';
const API_KEY = process.env.MOLECULE_API_KEY;
const SERVICE_TOKEN = process.env.MOLECULE_SERVICE_TOKEN;

async function uploadFile(ipnftUid: string, filePath: string, fileName: string) {
  const fileBuffer = readFileSync(filePath);

  // Step 1: Initiate
  const initResponse = await fetch(API_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY,
      'X-Service-Token': SERVICE_TOKEN
    },
    body: JSON.stringify({
      query: `
          mutation InitUpload($ipnftUid: String!, $contentType: String!, $contentLength: Int!) {
            initiateCreateOrUpdateFileV2(
              ipnftUid: $ipnftUid
              contentType: $contentType
              contentLength: $contentLength
            ) {
              uploadToken
              uploadUrl
              headers { key value }
              isSuccess
              error { message }
            }
          }
      `,
      variables: {
        ipnftUid,
        contentType: 'application/pdf',
        contentLength: fileBuffer.length
      }
    })
  });

  const { data: { initiateCreateOrUpdateFileV2: initResult } } = await initResponse.json();

  if (!initResult.isSuccess) {
    throw new Error(initResult.error?.message || 'Failed to initiate upload');
  }

  // Step 2: Upload to S3
  const headers: Record<string, string> = {};
  initResult.headers.forEach((h: {key: string, value: string}) => {
    headers[h.key] = h.value;
  });

  await fetch(initResult.uploadUrl, {
    method: 'PUT',
    headers,
    body: fileBuffer
  });

  // Step 3: Finalize
  const finishResponse = await fetch(API_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY,
      'X-Service-Token': SERVICE_TOKEN
    },
    body: JSON.stringify({
      query: `
          mutation FinishUpload(
            $ipnftUid: String!
            $uploadToken: String!
            $path: String!
            $accessLevel: String!
            $changeBy: String!
          ) {
            finishCreateOrUpdateFileV2(
              ipnftUid: $ipnftUid
              uploadToken: $uploadToken
              path: $path
              accessLevel: $accessLevel
              changeBy: $changeBy
            ) {
              contentHash
              version
              isSuccess
              error { message }
            }
          }
      `,
      variables: {
        ipnftUid,
        uploadToken: initResult.uploadToken,
        path: fileName,
        accessLevel: 'ADMIN',
        changeBy: '0xYourWalletAddress'
      }
    })
  });

  const { data: { finishCreateOrUpdateFileV2: result } } = await finishResponse.json();

  if (!result.isSuccess) {
    throw new Error(result.error?.message || 'Failed to finalize upload');
  }

  return result;
}

// Usage
uploadFile('0xcaD...Fc1_9', './report.pdf', 'research/q1-report.pdf')
.then(result => console.log('Uploaded:', result.contentHash))
.catch(err => console.error('Error:', err));
```
