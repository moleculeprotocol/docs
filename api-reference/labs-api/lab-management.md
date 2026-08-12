# Lab Management

Operations for creating and administering a Lab: creating the dataroom, managing its LabNFT display metadata, managing members, and linking its decentralised identifier (DID). Who may contribute to the dataroom is covered in [Access Policies](access-policies.md).

***

## Create Lab

Register a Kamu-backed lab (data room) for an onchain lab (OCL) that already exists onchain. The lab is identified by its canonical `oclId` (a 32-byte hex string, 0x-prefixed).

> **Prerequisite — the LabNFT must be minted first.** `createLab` does not mint anything; it attaches a dataroom to an OCL that already exists. Minting happens onchain via `OnChainLabFactory.mintAndCreateAccount`, which mints the LabNFT and deploys its bound account in one transaction — see [Lab Creation](../../technical-deep-dive/architecture.md#lab-creation) for the contract-level flow, or [Molecule Labs](../../technical-deep-dive/onchain-lab.md) for what a Lab is and how `oclId` is derived from the minted token. If you'd rather not touch the contracts directly, the Molecule app does this for you in [Step 1: Create Your Onchain Lab](../../user-guides/scientists-researchers.md#step-1-create-your-onchain-lab).

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

| Field        | Type                 | Required | Description                                                                                                     |
| ------------ | -------------------- | -------- | --------------------------------------------------------------------------------------------------------------- |
| oclId        | String               | Yes      | Canonical 32-byte oclId (lowercase 0x-hex) of the onchain lab                                                    |
| accessPolicy | LabAccessPolicyInput | No       | Contribution-access policy. Omit for the default role-gated lab — see [Access Policies](access-policies.md) |

> **Creating a lab open to contributions**: pass `accessPolicy: { preset: OPEN }` to let any authenticated caller add files, or configure per-capability rules and onchain conditions. The policy can be changed later with `updateLabAccessPolicy`. Both are owner-only — full details in [Access Policies](access-policies.md).

**Prerequisites:**

1. **LabNFT Ownership**: You must own the LabNFT for the `oclId` or be an authorized signer for it
   * For individual wallets: You must be the owner
   * For multisig/Safe wallets: You must be one of the Safe owners
   * For ERC-4337 accounts: You must be an authorized account owner
2. **Authentication**: One of the following:
   * **Service Token** (recommended for automation): Obtain from Molecule team via Discord
   * **Privy Token** (for user-initiated requests): Use your authenticated Privy session
3. **LabNFT Must Be Minted**: The onchain lab (LabNFT / `oclId`) must already exist onchain before registering the lab

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
  "errors": [
    {
      "message": "Admin authorization required. Please contact Molecule team for service token access.",
      "extensions": { "code": "UNAUTHORIZED" }
    }
  ]
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
        "message": "Onchain verification failed: wallet address is not owner or authorized signer",
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
2. **Onchain Verification**: Verifies you own or are an authorized signer for the LabNFT (`oclId`)
3. **Lab Creation**: Registers the Kamu-backed lab and its data room for the `oclId`
4. **Whitelist Update**: Automatically adds your wallet address to the lab whitelist
5. **Returns Result**: Lab details if successful, error details if failed

**Use Cases:**

* **Automate Lab Creation**: Register labs programmatically after minting LabNFTs
* **CI/CD Integration**: Automatically set up data rooms for new research labs
* **Batch Operations**: Register multiple labs for a portfolio of onchain labs
* **User Self-Service**: Allow users to create their own lab data rooms

**Getting Service Token Access:**

To obtain a service token for automated lab creation:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team
3. Provide:
   * Your wallet address
   * Use case description
   * Intended automation workflow
4. You'll receive:
   * API Key (for all APIs)
   * Service Token (JWT for lab creation)
   * Token expiration date

***

## Get Single Project with Files

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

> The optional CMS-enriched fields `trlValue`, `trlRationale`, and `isVerified` (see [List All Projects](browse-and-search.md#list-all-projects)) are also available on this query and are hydrated only when requested.

***

## LabNFT Metadata

### Update LabNFT Metadata

Partial update of the LabNFT display metadata (`name`, `description`, `image`, `externalUrl`). Omitted fields are left unchanged; an explicit `null` clears a field.

> **Authorization**: Restricted to the OCL admin (LabNFT owner + multisig signers).

```graphql
mutation UpdateLabNftMetadata(
  $oclId: String!
  $input: UpdateLabNftMetadataInput!
) {
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

| Parameter | Type                      | Required | Description                        |
| --------- | ------------------------- | -------- | ---------------------------------- |
| oclId     | String                    | Yes      | Canonical 32-byte oclId of the lab |
| input     | UpdateLabNftMetadataInput | Yes      | Patch object (all fields optional) |

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

***

***

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

| Field         | Type            | Description                                                                                                            |
| ------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------- |
| walletAddress | String          | Lowercased wallet address of the member                                                                                |
| role          | LabMemberRole   | Effective role: `OWNER`, `CONTRIBUTOR`, or `VIEWER`                                                                    |
| source        | LabMemberSource | Row that defines the membership: `ONCHAIN_EVENT`, `MULTISIG_RESOLUTION`, `ACCESS_CONTRACT`, or `ACCESS_RESOLVER_EVENT` |
| expiry        | String          | Unix-seconds expiry as a decimal string; `null` means the grant is permanent                                           |
| isAgent       | Boolean         | True if the member is an agent identity (surfaced for UI; not used for authorization)                                  |
| grantedAt     | String          | ISO-8601 timestamp the row was first persisted                                                                         |

***

***

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

`status` is a `DidLinkingStatus`: `PENDING`, `SUBMITTED`, `LINKED`, or `FAILED` (`null` before the first linking attempt). `linkedDidCount` reflects the number of active onchain DID links observed by the event indexer.

***
