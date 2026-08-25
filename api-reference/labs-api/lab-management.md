# Lab Management

Operations for creating and administering a Lab: creating the dataroom, managing its LabNFT display metadata, managing members, and linking its decentralised identifier (DID).

***

## Mint the LabNFT

Before `createLab` can attach a dataroom, an onchain lab (OCL) has to exist: a LabNFT minted to your wallet with its ERC-6551 account (Token Bound Account) deployed. This step is **onchain only** — there is no Labs API mutation for it. See [Lab Creation](../../technical-deep-dive/architecture.md#lab-creation) for the contract-level flow and [Molecule Labs](../../technical-deep-dive/onchain-lab.md) for what a Lab is. If you'd rather not touch contracts directly, the Molecule app does this for you in [Step 1: Create Your Onchain Lab](../../user-guides/scientists-researchers.md#step-1-create-your-onchain-lab).

### Contract Addresses

| Chain                  | `OnChainLabFactory`                         | `LabNFT` (proxy)                            |
| ----------------------- | -------------------------------------------- | -------------------------------------------- |
| Base Mainnet (`8453`)   | `0xECdF4f05384056507485C90aeAb0a83268760D6E` | `0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92` |
| Base Sepolia (`84532`)  | `0xd629FE2310b4309a212495F10A47f8436dcEfD90` | `0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28` |

Full deployment list, including every other OCL contract: [Contracts reference](../../references/contracts/README.md).

### Mint

Call `mintAndCreateAccount` on the factory. It mints the LabNFT to `to` and deploys the bound account in the same transaction, returning `(account, tokenId)`. The call is `payable` — read the current fee off the LabNFT's `mintFeeWei()` and send it as `value`, or the transaction reverts.

```solidity
function mintAndCreateAccount(address to) external payable returns (address account, uint256 tokenId);
```

**Example (viem):**

```javascript
import {
  createPublicClient,
  createWalletClient,
  http,
  parseAbi,
  parseEventLogs,
} from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { baseSepolia } from "viem/chains"; // use `base` + the mainnet addresses in production

const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90";
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28";

const factoryAbi = parseAbi([
  "function mintAndCreateAccount(address to) external payable returns (address account, uint256 tokenId)",
]);
const labNftAbi = parseAbi([
  "function mintFeeWei() external view returns (uint256)",
  "event OclIdentityCreated(address indexed account, bytes32 indexed oclId, uint256 indexed tokenId, bytes32 salt, uint256 canonicalChainId)",
]);

const account = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY);
const publicClient = createPublicClient({ chain: baseSepolia, transport: http() });
const walletClient = createWalletClient({ account, chain: baseSepolia, transport: http() });

const mintFeeWei = await publicClient.readContract({
  address: LABNFT_ADDRESS,
  abi: labNftAbi,
  functionName: "mintFeeWei",
});

const txHash = await walletClient.writeContract({
  address: FACTORY_ADDRESS,
  abi: factoryAbi,
  functionName: "mintAndCreateAccount",
  args: [account.address],
  value: mintFeeWei,
});

const receipt = await publicClient.waitForTransactionReceipt({ hash: txHash });

// The canonical oclId comes straight off the LabNFT's OclIdentityCreated
// event — no manual packing needed for the happy path.
const [identity] = parseEventLogs({
  abi: labNftAbi,
  eventName: "OclIdentityCreated",
  logs: receipt.logs.filter(
    (l) => l.address.toLowerCase() === LABNFT_ADDRESS.toLowerCase(),
  ),
});

console.log("oclId:", identity.args.oclId);
console.log("tokenId:", identity.args.tokenId.toString());
console.log("labAccountAddress:", identity.args.account);
```

`OclIdentityCreated` fires on the LabNFT contract itself (not the factory); filter receipt logs to `LABNFT_ADDRESS` before decoding. The factory's own `AccountProvisioned` event confirms the account was deployed but does not carry `oclId` as a topic.

### How `oclId` Is Derived

`identity.args.oclId` above is already the value `createLab` expects — normalize to lowercase and you're done. For reference, or to cross-check the emitted value, `oclId` is a bit-packed `bytes32`:

| Bytes    | Field     | Value                                          |
| -------- | --------- | ----------------------------------------------- |
| 1 (MSB)  | version   | `0x01`                                          |
| 1        | namespace | `0x01` (EVM)                                    |
| 10       | tokenId   | big-endian `uint80`                             |
| 20 (LSB) | account   | the ERC-6551 Token Bound Account, lowercased    |

```javascript
function computeOclId(tokenId, accountAddress) {
  const packed =
    (0x01n << 248n) |
    (0x01n << 240n) |
    (tokenId << 160n) |
    BigInt(accountAddress.toLowerCase());
  return "0x" + packed.toString(16).padStart(64, "0");
}
```

Note the chain id is **not** encoded in `oclId` — it's implied by which `LabNFT` deployment minted the token.

Once you have `oclId`, continue to [Create Lab](#create-lab) below.

***

## Create Lab

Register a Kamu-backed lab (data room) for an onchain lab (OCL) that already exists onchain. The lab is identified by its canonical `oclId` (a 32-byte hex string, 0x-prefixed).

> **Prerequisite — the LabNFT must be minted first.** `createLab` does not mint anything; it attaches a dataroom to an OCL that already exists onchain. See [Mint the LabNFT](#mint-the-labnft) above for the contract call and how to derive `oclId` from the result. If you'd rather not touch the contracts directly, the Molecule app does this for you in [Step 1: Create Your Onchain Lab](../../user-guides/scientists-researchers.md#step-1-create-your-onchain-lab).

> **Admin Authorization Required**: This mutation requires either a service token (JWT) from the Molecule team OR a valid Privy authentication token. The caller must be the LabNFT owner (or an authorized multisig signer) for the given `oclId`.

**GraphQL Mutation:**

```graphql
mutation CreateLab($oclId: String!) {
  createLab(input: { oclId: $oclId }) {
    message
    error {
      code
      message
      requestId
      retryable
      details
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

| Field | Type   | Required | Description                                                   |
| ----- | ------ | -------- | ------------------------------------------------------------- |
| oclId | String | Yes      | Canonical 32-byte oclId (lowercase 0x-hex) of the onchain lab |

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
Authorization: YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

**Option 2: Privy Token (User-Initiated)**

```bash
Authorization: Bearer YOUR_PRIVY_TOKEN
x-wallet-address: YOUR_WALLET_ADDRESS
```

**Example Request (Service Token):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_SERVICE_TOKEN' \
  -d '{
    "query": "mutation CreateLab($oclId: String!) { createLab(input: { oclId: $oclId }) { message error { code message requestId retryable details } lab { oclId shortname labAccountAddress labNftTokenId } } }",
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

`createLab` reports failures in-band: `error` is `null` on success and a full `ApiError` on failure. Branch on `error.code` (and, where documented, the `reason` key inside `error.details`, a JSON-encoded string) — never on message text. The top-level `message` mirrors `error.message` on failure.

**Not Authenticated (No Token):**

```json
{
  "data": {
    "createLab": {
      "message": "Authentication required. Please provide either a service token (x-service-token header) or Privy authentication token (Authorization header + x-wallet-address header). Contact Molecule tech team to obtain a service token.",
      "error": {
        "code": "UNAUTHENTICATED",
        "message": "Authentication required. Please provide either a service token (x-service-token header) or Privy authentication token (Authorization header + x-wallet-address header). Contact Molecule tech team to obtain a service token.",
        "requestId": "8f1e4c9a-2b7d-4e10-9c3a-5d6f7a8b9c0d",
        "retryable": false,
        "details": "{\"reason\":\"NO_AUTH\"}"
      },
      "lab": null
    }
  }
}
```

**Not the LabNFT Owner (Privy user token):**

```json
{
  "data": {
    "createLab": {
      "message": "You are not allowed to perform this operation.",
      "error": {
        "code": "UNAUTHORIZED",
        "message": "You are not allowed to perform this operation.",
        "requestId": "2b9c11d0-6f3e-4a71-8d52-c4e9b0a1f7d3",
        "retryable": false,
        "details": "{\"reason\":\"UNAUTHORIZED\"}"
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
      "message": "Project already exists",
      "error": {
        "code": "CONFLICT",
        "message": "Project already exists",
        "requestId": "5c7d2e81-9a4b-4f06-b3e8-1d0f6a2c8e94",
        "retryable": false,
        "details": "{\"reason\":\"PROJECT_CONFLICT\"}"
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
   * Consumer credential (for all APIs)
   * Service Token (JWT for lab creation)
   * Token expiration date

***

## Get Single Project with Files

Retrieve complete details for a specific lab including all files. This is a **public endpoint** - no authentication required. Look up a lab by its `oclId` or, alternatively, by its human-readable `shortname` — provide exactly one.

> **🔓 Public Endpoint**: The `labWithDataRoomAndFiles` query does not require authentication. You only need a consumer credential (`Authorization: Bearer`) - no Service Token is needed. File-level access control is handled via encryption rather than query-level authentication.

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
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
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
    oclId
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

| Parameter   | Type   | Required | Description                                                                             |
| ----------- | ------ | -------- | --------------------------------------------------------------------------------------- |
| oclId       | String | Yes      | Canonical 32-byte oclId of the lab                                                      |
| contentType | String | Yes      | Image MIME type (`image/jpeg`, `image/png`, `image/webp`, `image/gif`, `image/svg+xml`) |

***

***

## Lab Members

### List Lab Members

Return the active members of a lab (owner, contributors, viewers), sourced from the indexed `ocl_user` table. Expired grants are excluded.

> **Public query** — only a consumer credential is required. The same data is also exposed on the public `Lab` / `LabRef.members` field.

```graphql
query ListLabMembers($oclId: String!) {
  listLabMembers(oclId: $oclId) {
    message
    members {
      walletAddress
      role
      source
      expiry
      isAgent
      grantedAt
    }
  }
}
```

Failures throw: they arrive as top-level GraphQL `errors[]` entries with `errorType` set to the catalogue code (an unknown `oclId` throws `NOT_FOUND`). The response carries `"data": null` (the field is non-nullable, so the error propagates to the root) and `errors[0].path` names `listLabMembers`.

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
  }
}
```

Failures throw: they arrive as top-level GraphQL `errors[]` entries with `errorType` set to the catalogue code. The response carries `"data": null` (the field is non-nullable, so the error propagates to the root) and `errors[0].path` names `getDidLinkStatus`.

**Parameters:**

| Parameter | Type   | Required | Description                        |
| --------- | ------ | -------- | ---------------------------------- |
| oclId     | String | Yes      | Canonical 32-byte oclId of the lab |

`status` is a `DidLinkingStatus`: `PENDING`, `SUBMITTED`, `LINKED`, or `FAILED` (`null` before the first linking attempt). `linkedDidCount` reflects the number of active onchain DID links observed by the event indexer.

***
