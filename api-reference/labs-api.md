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

The Labs API has different authentication requirements depending on the operation:

### Public Queries (Read-Only)

The following queries are **public** and only require an API Key:
- `projectsV2` - List all projects with pagination
- `projectWithDataRoomAndFilesV2` - Get project details and files

```bash
x-api-key: YOUR_API_KEY
```

### Protected Mutations (Write Operations)

All mutations (file uploads, deletions, announcements, etc.) require **two authentication headers**:

1. **API Key** - For general API authentication
2. **Service Token** - For lab-specific write access control

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

**For public queries** (projectsV2, projectWithDataRoomAndFilesV2):
```bash
x-api-key: YOUR_API_KEY
```

**For mutations** (file uploads, deletions, announcements, metadata updates):
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
  $ipnftUid: String!
  $contentType: String!
  $contentLength: Int!
) {
  initiateCreateOrUpdateFileV2(
    ipnftUid: $ipnftUid
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
| ipnftUid      | String | Yes      | IP-NFT unique identifier in format `contractAddress_tokenId` (e.g., `0xcaD...1_37`) |
| contentType   | String | Yes      | MIME type of the file (e.g., `application/pdf`, `image/png`)                        |
| contentLength | Int    | Yes      | File size in bytes                                                                  |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation InitiateFileUpload($ipnftUid: String!, $contentType: String!, $contentLength: Int!) { initiateCreateOrUpdateFileV2(ipnftUid: $ipnftUid, contentType: $contentType, contentLength: $contentLength) { uploadToken uploadUrl uploadUrlExpiry method headers { key value } isSuccess error { message code retryable } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
      "contentType": "application/pdf",
      "contentLength": 381846
    }
  }'
```

**Success Response:**

```json
{
  "data": {
    "initiateCreateOrUpdateFileV2": {
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
  $ipnftUid: String!
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
  finishCreateOrUpdateFileV2(
    ipnftUid: $ipnftUid
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
| ipnftUid    | String   | Yes      | Same IP-NFT identifier used in Step 1                       |
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
    "query": "mutation FinishFileUpload($ipnftUid: String!, $uploadToken: String!, $path: String, $accessLevel: String!, $changeBy: String!, $description: String, $tags: [String!], $categories: [String!], $contentText: String) { finishCreateOrUpdateFileV2(ipnftUid: $ipnftUid, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel, changeBy: $changeBy, description: $description, tags: $tags, categories: $categories, contentText: $contentText) { datasetId contentHash version isSuccess message error { message code retryable } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
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
    "finishCreateOrUpdateFileV2": {
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
  $ipnftUid: String!
  $ref: String!
  $accessLevel: String!
  $description: String
  $tags: [String!]
  $categories: [String!]
  $contentText: String
) {
  updateFileMetadataV2(
    ipnftUid: $ipnftUid
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
| ipnftUid    | String   | Yes      | IP-NFT unique identifier                                        |
| ref         | String   | Yes      | File reference (DID) from `finishCreateOrUpdateFileV2` response |
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
    "query": "mutation UpdateFileMetadata($ipnftUid: String!, $ref: String!, $accessLevel: String!, $description: String, $tags: [String!], $categories: [String!], $contentText: String) { updateFileMetadataV2(ipnftUid: $ipnftUid, ref: $ref, accessLevel: $accessLevel, description: $description, tags: $tags, categories: $categories, contentText: $contentText) { ref isSuccess message error { message } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
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
mutation DeleteFile($ipnftUid: String!, $path: String!, $changeBy: String!) {
  deleteDataRoomFileV2(ipnftUid: $ipnftUid, path: $path, changeBy: $changeBy) {
    ipnftUid
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
| ipnftUid  | String | Yes      | IP-NFT unique identifier           |
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
    "query": "mutation DeleteFile($ipnftUid: String!, $path: String!, $changeBy: String!) { deleteDataRoomFileV2(ipnftUid: $ipnftUid, path: $path, changeBy: $changeBy) { ipnftUid filePath isSuccess error { message } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
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
  $ipnftUid: String!
  $headline: String!
  $body: String!
  $attachments: [String!]
) {
  createAnnouncementV2(
    ipnftUid: $ipnftUid
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
| ipnftUid    | String   | Yes      | IP-NFT unique identifier                         |
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
    "query": "mutation CreateAnnouncement($ipnftUid: String!, $headline: String!, $body: String!, $attachments: [String!]) { createAnnouncementV2(ipnftUid: $ipnftUid, headline: $headline, body: $body, attachments: $attachments) { isSuccess message error { message } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
      "headline": "Research Milestone Achieved",
      "body": "We have completed Phase 2 trials with promising results.",
      "attachments": ["did:kamu:fed01..."]
    }
  }'
```

---

### Querying Projects and Files

Query operations for browsing projects, viewing files, and checking activity.

#### List All Projects

Get all IP-NFT projects. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `projectsV2` query does not require authentication. You only need the `x-api-key` header - no Service Token is needed.

**GraphQL Query:**

```graphql
query ListProjects($walletAddress: String, $page: Int, $perPage: Int) {
  projectsV2(walletAddress: $walletAddress, page: $page, perPage: $perPage) {
    nodes {
      ipnftUid
      ipnftSymbol
      ipnftAddress
      ipnftTokenId
      systemTime
      eventTime
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

| Parameter     | Type   | Required | Description                                              |
| ------------- | ------ | -------- | -------------------------------------------------------- |
| walletAddress | String | No       | Filter projects by admin wallet address                  |
| page          | Int    | No       | Page number (0-indexed, default: 0)                      |
| perPage       | Int    | No       | Results per page (default: 20, max: 100)                 |

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query ListProjects($page: Int, $perPage: Int) { projectsV2(page: $page, perPage: $perPage) { nodes { ipnftUid ipnftSymbol dataRoom { id alias } } totalCount pageInfo { hasNextPage currentPage totalPages } } }",
    "variables": {
      "page": 0,
      "perPage": 20
    }
  }'
```

#### Get Single Project with Files

Retrieve complete details for a specific project including all files. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `projectWithDataRoomAndFilesV2` query does not require authentication. You only need the `x-api-key` header - no Service Token is needed. File-level access control is handled via encryption rather than query-level authentication.

**GraphQL Query:**

```graphql
query GetProject($ipnftUid: ID!) {
  projectWithDataRoomAndFilesV2(ipnftUid: $ipnftUid) {
    ipnftUid
    ipnftSymbol
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
    "query": "query GetProject($ipnftUid: ID!) { projectWithDataRoomAndFilesV2(ipnftUid: $ipnftUid) { ipnftUid ipnftSymbol dataRoom { id files { path contentType accessLevel tags } } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37"
    }
  }'
```

#### Get File by Path

Retrieve a specific file using IP-NFT UID and file path.

**GraphQL Query:**

```graphql
query GetFile($ipnftUid: String!, $path: String!) {
  dataRoomFileV2(ipnftUid: $ipnftUid, path: $path) {
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
    "query": "query GetFile($ipnftUid: String!, $path: String!) { dataRoomFileV2(ipnftUid: $ipnftUid, path: $path) { did path contentType accessLevel downloadUrl } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
      "path": "research-data.pdf"
    }
  }'
```

#### Project Activity Feed

Get activity timeline for a specific project including file events and announcements.

**GraphQL Query:**

```graphql
query GetActivity($ipnftUid: ID!, $page: Int, $perPage: Int) {
  projectActivityV2(ipnftUid: $ipnftUid, page: $page, perPage: $perPage) {
    pageInfo {
      hasNextPage
      hasPreviousPage
      currentPage
      totalPages
    }
    nodes {
      __typename
      ... on ProjectEventFileAddedV2 {
        entry {
          ref
        }
      }
      ... on ProjectEventAnnouncementV2 {
        announcement {
          id
          headline
          body
          systemTime
          changeBy
          attachments {
            id
            did
            path
            name
            contentType
            accessLevel
          }
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
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "query GetActivity($ipnftUid: ID!, $page: Int) { projectActivityV2(ipnftUid: $ipnftUid, page: $page, perPage: 20) { pageInfo { hasNextPage currentPage totalPages } nodes { __typename ... on ProjectEventAnnouncementV2 { announcement { headline attachments { did path contentType } } } } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
      "page": 0
    }
  }'
```

**Performance Note**: Attachment fields in `projectActivityV2` are limited to essential display fields (did, path, name, contentType, accessLevel) for optimal performance. For full attachment metadata including downloadUrl and encryption details, use `projectAnnouncementsV2` instead.

#### Project Announcements (Dedicated Endpoint)

Get announcements for a specific project with full attachment details. More efficient than `projectActivityV2` when you only need announcements.

**GraphQL Query:**

```graphql
query GetProjectAnnouncements($ipnftUid: String!, $page: Int, $perPage: Int) {
  projectAnnouncementsV2(ipnftUid: $ipnftUid, page: $page, perPage: $perPage) {
    totalCount
    pageInfo {
      hasNextPage
      hasPreviousPage
      currentPage
      totalPages
    }
    nodes {
      id
      headline
      body
      systemTime
      eventTime
      changeBy
      project {
        ipnftUid
        ipnftSymbol
        ipnftAddress
        ipnftTokenId
      }
      attachments {
        id
        did
        path
        name
        contentType
        accessLevel
        version
        contentHash
        description
        categories
        tags
        downloadUrl
        downloadUrlExpiry
        encryptionMetadata {
          dataToEncryptHash
          encryptedBy
          chain
          litNetwork
        }
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
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "query GetProjectAnnouncements($ipnftUid: String!, $page: Int, $perPage: Int) { projectAnnouncementsV2(ipnftUid: $ipnftUid, page: $page, perPage: $perPage) { totalCount pageInfo { hasNextPage currentPage } nodes { headline body attachments { did path contentType downloadUrl } } } }",
    "variables": {
      "ipnftUid": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37",
      "page": 0,
      "perPage": 20
    }
  }'
```

**Use Cases:**

- Announcement detail pages requiring full file metadata
- Download links for announcement attachments
- Encrypted file access with Lit Protocol
- Projects with many announcements (efficient pagination)

#### All Announcements (Global Feed)

Get all announcements across all projects.

**GraphQL Query:**

```graphql
query GetAllAnnouncements($page: Int, $perPage: Int) {
  activitiesV2(page: $page, perPage: $perPage) {
    isSuccess
    announcements {
      id
      headline
      body
      systemTime
      project {
        ipnftUid
        ipnftSymbol
      }
      attachments {
        id
        did
        path
        name
        contentType
        accessLevel
      }
    }
  }
}
```

> **Note**: Attachment fields in `activitiesV2` are limited for performance. Use `projectAnnouncementsV2` for full attachment metadata including downloadUrl.

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
          project {
            ipnftUid
            ipnftSymbol
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
        project {
          ipnftUid
          ipnftSymbol
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
| byIpnftUids    | [String!] | Filter by specific project UIDs                       |
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
    "query": "query SearchLabs($prompt: String!, $page: Int, $perPage: Int) { searchLabs(prompt: $prompt, page: $page, perPage: $perPage) { nodes { __typename ... on SearchLabsFileHit { entry { project { ipnftUid ipnftSymbol } path file { contentType description tags } } } ... on SearchLabsAnnouncementHit { announcement { headline body } project { ipnftSymbol } } } totalCount pageInfo { hasNextPage currentPage totalPages } } }",
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
  - Contains: headline, body, project reference, **typed attachments** (file objects)

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

async function uploadFileToLabs(filePath, ipnftUid, serviceToken) {
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
          mutation InitiateFileUpload($ipnftUid: String!, $contentType: String!, $contentLength: Int!) {
            initiateCreateOrUpdateFileV2(
              ipnftUid: $ipnftUid
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
          ipnftUid,
          contentType: "application/octet-stream",
          contentLength: fileSize,
        },
      }),
    });

    const initiateResult = await initiateResponse.json();
    if (!initiateResult.data?.initiateCreateOrUpdateFileV2?.isSuccess) {
      throw new Error(
        initiateResult.data?.initiateCreateOrUpdateFileV2?.error?.message ||
          "Failed to initiate upload",
      );
    }

    const { uploadToken, uploadUrl, headers } =
      initiateResult.data.initiateCreateOrUpdateFileV2;
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
              datasetId
              isSuccess
              message
              error { message }
            }
          }
        `,
        variables: {
          ipnftUid,
          uploadToken,
          path: filename,
          accessLevel: "PUBLIC",
          changeBy: process.env.WALLET_ADDRESS,
        },
      }),
    });

    const finishResult = await finishResponse.json();
    if (!finishResult.data?.finishCreateOrUpdateFileV2?.isSuccess) {
      throw new Error(
        finishResult.data?.finishCreateOrUpdateFileV2?.error?.message ||
          "Failed to finish upload",
      );
    }

    console.log("🎉 File upload completed successfully!");
    console.log(
      "Dataset ID:",
      finishResult.data.finishCreateOrUpdateFileV2.datasetId,
    );

    return {
      success: true,
      datasetId: finishResult.data.finishCreateOrUpdateFileV2.datasetId,
    };
  } catch (error) {
    console.error("❌ Upload failed:", error.message);
    throw error;
  }
}

// Usage
if (require.main === module) {
  const filePath = process.argv[2];
  const ipnftUid = process.argv[3];
  const serviceToken = process.env.SERVICE_TOKEN;

  if (!filePath || !ipnftUid || !serviceToken) {
    console.error(
      'Usage: SERVICE_TOKEN="token" WALLET_ADDRESS="0x..." node upload.js <file> <ipnft-uid>',
    );
    process.exit(1);
  }

  uploadFileToLabs(filePath, ipnftUid, serviceToken);
}

module.exports = { uploadFileToLabs };
```

**Usage:**

```bash
API_KEY="your-api-key" SERVICE_TOKEN="your-service-token" WALLET_ADDRESS="0x..." node upload.js data.pdf 0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37
```

---

## Service Token Management

### Obtaining Tokens

Service tokens must be requested from the Molecule team (see [Authentication](#authentication) section above).

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

## Error Handling

All API responses follow a consistent error format:

### Error Response Structure

```json
{
  "data": {
    "initiateCreateOrUpdateFileV2": {
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
| 403         | Forbidden             | Service token does not have access to the specified IP-NFT       |
| 400         | Bad Request           | Invalid parameters (e.g., missing ipnftUid, invalid contentType) |
| 404         | Not Found             | IP-NFT or dataroom not found                                     |
| 413         | Payload Too Large     | File exceeds size limits                                         |
| 500         | Internal Server Error | Server error - check if retryable and try again                  |

### Troubleshooting

**"Missing service token" error:**

- Ensure `X-Service-Token` header is included in requests
- Verify token is not empty or malformed

**"Service does not have access to IPNFT" error:**

- Verify your wallet address (linked to service token) has admin access to the IP-NFT/dataroom
- Check that the ipnftUid format is correct: `contractAddress_tokenId`

**"Token expired" error:**

- Request a new token from the Molecule team, or
- Use `extendServiceToken` mutation to extend expiration

**Upload to presigned URL fails:**

- Ensure binary file upload (use `--data-binary` in curl)
- Verify headers match those returned in Step 1
- Check that presigned URL hasn't expired (expires after ~15 minutes)

**"File not found" error (updateFileMetadataV2, deleteDataRoomFileV2):**

- Verify the file `ref` (DID) or `path` is correct
- Check that the file exists in the specified dataroom
- Ensure you have access to the dataroom

**"Invalid search filters" error (searchLabs):**

- Verify filter values match expected types (arrays of strings)
- Check that access level values are: PUBLIC, HOLDERS, or ADMIN
- Ensure ipnftUid format is correct if using byIpnftUids filter

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

For files requiring client-side encryption, you can include encryption metadata using the Lit Protocol.

### Encryption Metadata Parameter

```graphql
$encryptionMetadata: EncryptionMetadataInput
```

**Structure:**

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
- Conditional access based on token ownership

For more information about Lit Protocol encryption, visit the [Lit Protocol documentation](https://developer.litprotocol.com/).

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
- **Update metadata as needed**: Use `updateFileMetadataV2` to refine tags and descriptions without re-uploading files

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

## Deprecated Operations

The following V1 operations have been replaced by improved V2 versions. **V1 operations are deprecated and will be removed in a future release.**

### Deprecated Queries

| V1 Query                      | V2 Replacement                   | Key Changes                              |
| ----------------------------- | -------------------------------- | ---------------------------------------- |
| `projects`                    | `projectsV2`                     | Added systemTime and eventTime fields    |
| `projectWithDataRoomAndFiles` | `projectWithDataRoomAndFilesV2`  | Enhanced temporal tracking               |
| `dataRoomFile(did: String!)`  | `dataRoomFileV2(ipnftUid, path)` | Query by IPNFT UID + path instead of DID |
| `projectActivity`             | `projectActivityV2`              | Improved activity feed structure         |
| `activities`                  | `activitiesV2`                   | Enhanced announcement structure          |

### Deprecated Mutations

| V1 Mutation          | V2 Replacement                 | Key Changes                                            |
| -------------------- | ------------------------------ | ------------------------------------------------------ |
| `initiateFileUpload` | `initiateCreateOrUpdateFileV2` | Simplified parameters, no dataset alias needed         |
| `finishFileUpload`   | `finishCreateOrUpdateFileV2`   | Added metadata support (tags, categories, contentText) |
| `deleteDataRoomFile` | `deleteDataRoomFileV2`         | Streamlined parameters                                 |
| `createAnnouncement` | `createAnnouncementV2`         | **Breaking**: Removed `moleculeAccessLevel` parameter  |

### Migration Guide

#### Breaking Change 1: createAnnouncement → createAnnouncementV2

The `moleculeAccessLevel` parameter has been removed in V2:

```diff
# V1 (Deprecated)
- createAnnouncement(
-   ipnftUid: "...",
-   headline: "...",
-   body: "...",
-   attachments: [...],
-   moleculeAccessLevel: "PUBLIC"  ❌ Removed in V2
- )

# V2 (Current)
+ createAnnouncementV2(
+   ipnftUid: "...",
+   headline: "...",
+   body: "...",
+   attachments: [...]  ✓ Access level no longer needed
+ )
```

#### Breaking Change 2: Announcement Attachments Type Change

**⚠️ BREAKING CHANGE**: Announcement attachments changed from string arrays to full file objects.

**Previous Structure (V1):**

```json
{
  "announcement": {
    "attachments": ["did:kamu:fed01...", "did:kamu:abc23..."]
  }
}
```

**New Structure (V2):**

```json
{
  "announcement": {
    "attachments": [
      {
        "id": "file-1",
        "did": "did:kamu:fed01...",
        "path": "/research-data.pdf",
        "name": "research-data.pdf",
        "contentType": "application/pdf",
        "accessLevel": "PUBLIC"
      },
      {
        "id": "file-2",
        "did": "did:kamu:abc23...",
        "path": "/results.csv",
        "name": "results.csv",
        "contentType": "text/csv",
        "accessLevel": "HOLDERS"
      }
    ]
  }
}
```

**Migration Required**: If you query announcements via `projectActivityV2`, `activitiesV2`, `searchLabs`, or `projectAnnouncementsV2`, update your code to handle attachment objects instead of strings:

**Before:**

```javascript
announcement.attachments.forEach((did) => {
  // Had to make separate API call to get file details
  fetchFileByDid(did).then((file) => displayFile(file));
});
```

**After:**

```javascript
announcement.attachments.forEach((file) => {
  // File details already available
  displayFile(file);
  console.log(file.contentType, file.accessLevel);
});
```

**Benefits:**

- ✅ No additional API calls needed for file metadata
- ✅ Better developer experience
- ✅ Reduced API load
- ✅ Type-safe attachment handling

**Field Availability by Query:**

| Query                    | Attachment Fields Available                                        |
| ------------------------ | ------------------------------------------------------------------ |
| `projectActivityV2`      | Minimal (did, path, name, contentType, accessLevel)                |
| `activitiesV2`           | Minimal (did, path, name, contentType, accessLevel)                |
| `searchLabs`             | Minimal (did, path, name, contentType, accessLevel)                |
| `projectAnnouncementsV2` | **Full** (includes downloadUrl, encryptionMetadata, version, etc.) |

**All other V1 → V2 migrations are backward-compatible** with additional optional parameters in V2.

---

## Getting Support

If you encounter any issues or have questions about the Programmatic File Upload API:

1. Check this documentation and [troubleshooting section](#troubleshooting)
2. Review the [complete example](#complete-example) for implementation guidance
3. Join our [Discord community](https://t.co/L0VEiy4Bjk) for support
4. Contact the Molecule Labs development team directly

---

## Recent Updates (January 2025)

### Authentication Changes

#### 🔓 Public Query Endpoints

The following queries are now **publicly accessible** and no longer require Service Token authentication:

| Query | Previous Auth | New Auth | Notes |
|-------|---------------|----------|-------|
| `projectsV2` | Service Token OR User Auth | **API Key only** | Now supports pagination parameters |
| `projectWithDataRoomAndFilesV2` | Service Token + Admin Auth | **API Key only** | File access controlled via encryption |

**What This Means:**
- **Simplified Integration**: Query projects and files with just an API key
- **Reduced Complexity**: No need to manage Service Tokens for read-only operations
- **File Security Maintained**: Encrypted files remain protected via Lit Protocol - access control moved from query-level to file-level

**Migration Required:**
If you're currently including `X-Service-Token` headers for these queries, you can safely remove them. The queries will work with just `x-api-key`.

```diff
# Before (still works, but unnecessary)
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
- -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{"query": "query { projectsV2 { nodes { ipnftUid } } }"}'

# After (recommended)
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{"query": "query { projectsV2 { nodes { ipnftUid } } }"}'
```

#### 📊 New Pagination for `projectsV2`

The `projectsV2` query now supports pagination and wallet filtering:

```graphql
query ListProjects($walletAddress: String, $page: Int, $perPage: Int) {
  projectsV2(walletAddress: $walletAddress, page: $page, perPage: $perPage) {
    nodes { ... }
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

**New Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| walletAddress | String | null | Filter projects by admin wallet address |
| page | Int | 0 | Page number (0-indexed) |
| perPage | Int | 20 | Results per page (max: 100) |

**Note:** Invalid pagination values are automatically sanitized (negative page → 0, perPage > 100 → 20).

---

### New Features

#### 🔍 New Query: `searchLabs`

Semantic search across all IP-NFT projects, files, and announcements:

- Full-text search with relevance ranking
- Filter by tags, categories, access levels, project UIDs
- Returns unified results (files + announcements)
- Paginated results with totalCount

**Benefits:**

- Discover content across all Labs projects
- Filter by multiple criteria simultaneously
- Unified search experience
- Support for complex queries

See [Searching Labs](#searching-labs) section for complete documentation.

#### 📝 New Mutation: `updateFileMetadataV2`

Update file metadata without creating a new version:

- Modify description, tags, categories, contentText
- Update access level (PUBLIC, HOLDERS, ADMIN)
- No file re-upload required
- Preserves version history

**Benefits:**

- Refine file metadata after upload
- Improve searchability with updated contentText
- Reorganize files with new tags/categories
- Adjust access control as needed

See [Update File Metadata](#update-file-metadata) section for complete documentation.

#### ✨ New Query: `projectAnnouncementsV2`

Added dedicated announcements endpoint with full attachment metadata:

- Fetches ONLY announcements (no file events) for better performance
- Includes complete attachment details (downloadUrl, encryptionMetadata, etc.)
- Efficient pagination with totalCount
- **Use this instead of projectActivityV2 when you only need announcements**

**Benefits over projectActivityV2:**

- Faster response times
- Full attachment metadata available
- Direct pagination support
- No file events to filter through

#### 📊 Pagination Enhancement: `projectActivityV2`

Added `pageInfo` object to `projectActivityV2` results:

```graphql
{
  pageInfo {
    hasNextPage
    hasPreviousPage
    currentPage
    totalPages
  }
  nodes { ... }
}
```

Enables proper client-side pagination with total page count.

### Breaking Changes

#### ⚠️ Announcement Attachments Type Change (IP-1025)

**Affected Queries**: `projectActivityV2`, `activitiesV2`, `searchLabs`, `projectAnnouncementsV2`

**What Changed:**

- Attachments field type: `[String!]!` → `[DataRoomFile!]!`
- Instead of array of DIDs, you now get full file objects

**Before (Old):**

```json
"attachments": ["did:kamu:fed01...", "did:kamu:abc23..."]
```

**After (New):**

```json
"attachments": [
  {
    "did": "did:kamu:fed01...",
    "path": "/data.pdf",
    "contentType": "application/pdf",
    "accessLevel": "PUBLIC"
  }
]
```

**Why This Change:**

- ✅ Eliminates need for separate API calls to resolve file details
- ✅ Improves developer experience
- ✅ Reduces API load and complexity
- ✅ Type-safe file handling

**Migration:**
Update your code to access file properties directly instead of using DIDs to fetch file details:

```javascript
// OLD CODE (no longer works)
announcement.attachments.forEach(async (did) => {
  const file = await fetchFile(did); // Extra API call needed
  renderFile(file);
});

// NEW CODE (current)
announcement.attachments.forEach((file) => {
  renderFile(file); // File object already has all details
});
```

**Field Availability:**

- **Activity queries** (projectActivityV2, activitiesV2, searchLabs): Minimal fields (6 fields for performance)
- **Announcements query** (projectAnnouncementsV2): Full fields (20+ fields including downloadUrl)

### Performance Improvements

#### 🚀 Optimized Activity Query Performance

Reduced attachment fields in activity/search queries to prevent Lambda timeouts:

**Minimal Fields (Activity Queries):**

- Essential: `id`, `did`, `path`, `name`, `contentType`, `accessLevel`
- Removed: `downloadUrl`, `encryptionMetadata`, `contentText`, `version`, `contentHash`, etc.

**Result**: Faster response times, no timeout issues

**Full Fields Available**: Use `projectAnnouncementsV2` when you need downloadUrl, encryptionMetadata, or other detailed fields.

### Internal Improvements

#### 🔧 Path Field Consistency (IP-1352)

Fixed inconsistency where the `path` field differed between queries:

- Now consistently uses Kamu's native `path` field
- Improved `name` field priority: `dataset.name` → `description` → `path-derived`

**Impact**: More consistent file paths across all queries

---

_Last updated: January 2025_
