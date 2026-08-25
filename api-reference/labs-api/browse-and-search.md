# Browse & Search

Cross-cutting read operations that aren't scoped to a single lab you administer: browsing and reading labs and their files, full-text search, and activity feeds. Reads tied to a specific resource live with that resource — e.g. members and DID-link status in [Lab Management](lab-management.md), and legal-agreement status in [Legal Agreements](legal-agreements.md).

## Listing Labs & Activity

Query operations for listing all labs and reading their activity feeds. To read a single lab and its data-room files, see [Get Single Project with Files](lab-management.md#get-single-project-with-files); to read one file, see [Get File by Path](files.md#get-file-by-path).

### List All Projects

Get all labs. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `labs` query does not require authentication. You only need a consumer credential (`Authorization: Bearer`) - no Service Token is needed.

**GraphQL Query:**

```graphql
query ListProjects($walletAddress: String, $page: Int, $perPage: Int) {
  labs(walletAddress: $walletAddress, page: $page, perPage: $perPage) {
    nodes {
      oclId
      shortname
      name
      description
      labAccountAddress
      labNftTokenId
      latestContributionAt
      trlValue
      trlRationale
      isVerified
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

> The `labs` list returns lightweight `LabRef` objects. Data-room contents and account details are not part of `LabRef` — fetch them per lab via [`labWithDataRoomAndFiles`](lab-management.md#get-single-project-with-files).

**Parameters:**

| Parameter     | Type          | Required | Description                                                                                                                          |
| ------------- | ------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| walletAddress | String        | No       | Filter to labs where this wallet holds any active role (owner/contributor/viewer). Omit to return all labs.                          |
| role          | LabMemberRole | No       | Only meaningful with `walletAddress`: restrict to labs where the wallet holds this specific role (`OWNER`, `CONTRIBUTOR`, `VIEWER`). |
| page          | Int           | No       | Page number (0-indexed, default: 0)                                                                                                  |
| perPage       | Int           | No       | Results per page (default: 20, max: 100)                                                                                             |

**CMS-enriched fields (optional):**

These fields are sourced from the Molecule CMS and hydrated only when requested in the selection set. They are `null` when the project has no corresponding CMS entry.

| Field               | Type     | Description                                                                                                                        |
| ------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| trlValue            | String   | Technology Readiness Level (TRL) assessment for the project                                                                        |
| trlRationale        | String   | Explanation supporting the assigned TRL value                                                                                      |
| trlLastUpdated      | DateTime | Timestamp of the last change to `trlValue`                                                                                         |
| weightedScore       | Float    | AI-generated weighted project score derived from the `trlValue`                                                                    |
| scoreInterpretation | String   | Human-readable summary of the overall assessment behind `weightedScore`                                                            |
| criterionScores     | [JSON]   | Per-criterion breakdown behind `weightedScore`. Each entry is a JSON object shaped like `{ "criterion": String, "score": Number }` |
| scoredAt            | DateTime | Timestamp the project scoring behind `weightedScore` was last computed                                                             |
| todos               | [JSON]   | AI-generated action items for the lab, each a JSON object shaped like `{ "todo": String, "completed": Boolean }`                   |
| isVerified          | Boolean  | Whether the project has been verified by Molecule                                                                                  |

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query ListProjects($page: Int, $perPage: Int) { labs(page: $page, perPage: $perPage) { nodes { oclId shortname name labAccountAddress trlValue } totalCount pageInfo { hasNextPage currentPage totalPages } } }",
    "variables": {
      "page": 0,
      "perPage": 20
    }
  }'
```

### Project Activity Feed

Get activity timeline for a specific project including file events and announcements. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `labActivity` query does not require authentication. You only need a consumer credential (`Authorization: Bearer`) - no Service Token is needed.

> **Filtering**: By default, returns all activity types (file events and announcements). Use the optional `filter` parameter (`ANNOUNCEMENT` or `FILE`) to retrieve only a specific type.

**GraphQL Query:**

```graphql
query GetProjectActivity(
  $id: String!
  $page: Int!
  $perPage: Int!
  $filter: LabActivityFilter
) {
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
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
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
- Encrypted file access (Onchain-Verified Envelope Encryption for new files)
- Projects with many announcements (efficient pagination)

### Global Activity Feed

Get all activity across all projects. This is a **public endpoint** - no authentication required.

> **🔓 Public Endpoint**: The `activities` query does not require authentication. You only need a consumer credential (`Authorization: Bearer`) - no Service Token is needed.

> **Filtering**: By default, returns all activity types (file events and announcements). Use the optional `filter` parameter (`ANNOUNCEMENT` or `FILE`) to retrieve only a specific type.

**GraphQL Query:**

```graphql
query GetActivities($page: Int, $perPage: Int, $filter: LabActivityFilter) {
  activities(page: $page, perPage: $perPage, filter: $filter) {
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
  }
}
```

> **Errors**: a failed `activities` query returns `data: null` with a top-level GraphQL `errors[]` entry whose `errorType` is the error code — see [Error Handling](README.md#error-handling).

---

## Searching Labs

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

| Filter         | Type       | Description                                           |
| -------------- | ---------- | ----------------------------------------------------- |
| byOclIds       | \[String!] | Filter by specific lab oclIds                         |
| byTags         | \[String!] | Filter files by tags                                  |
| byCategories   | \[String!] | Filter files by categories                            |
| byKinds        | \[String!] | Filter by result type                                 |

**Example - Basic Search:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
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
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
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
    "Authorization": process.env.CONSUMER_CREDENTIAL,
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

## Onchain Activity

### Onchain Activity Feed

Return the onchain event feed for an OCL or a wallet. Exactly one of `oclId` / `wallet` must be supplied. Paginate with a cursor of the form `"<block_number>:<log_index>"` — pass the last row's `id` to fetch the next page.

```graphql
query OnChainActivity(
  $oclId: String
  $wallet: String
  $limit: Int
  $cursor: String
) {
  onChainActivity(
    oclId: $oclId
    wallet: $wallet
    limit: $limit
    cursor: $cursor
  ) {
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

| Parameter | Type   | Required | Description                                                        |
| --------- | ------ | -------- | ------------------------------------------------------------------ |
| oclId     | String | No\*     | Canonical 32-byte oclId of the lab                                 |
| wallet    | String | No\*     | Wallet address to filter events by                                 |
| limit     | Int    | No       | Max rows to return (default: 50)                                   |
| cursor    | String | No       | Pagination cursor `"<block_number>:<log_index>"` (last row's `id`) |

\* Provide exactly one of `oclId` or `wallet`. `contractName` is one of `accessresolver`, `ocl`, `ipnft` or `ipt`. `args` is a JSON object of the decoded event arguments (BigInts as decimal strings, addresses lowercased).

