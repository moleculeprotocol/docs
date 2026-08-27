# Tutorials

Four step-by-step tutorials, each ending in something you can verify. They share one config block and two helper functions ([Shared setup](#shared-setup)) and are written against **staging** (Base Sepolia, testnet funds) end to end — see [Running in Production](#running-in-production) for the values to swap.

| Tutorial | What you end up with | Start from |
| -------- | -------------------- | ---------- |
| [**1 — Create a lab and upload a public file**](#tutorial-1-create-a-lab-and-upload-a-public-file) | A lab you own, with a readable file in its data room | Nothing but a credential and a funded wallet |
| [**2 — Upload an encrypted file**](#tutorial-2-upload-an-encrypted-file) | A confidential file only authorised wallets can decrypt | Tutorial 1, or any lab you have a role on |
| [**3 — Give your agent access to a lab you created in the app**](#tutorial-3-give-your-agent-access-to-a-lab-you-created-in-the-app) | An agent wallet writing into a lab a human owns | A lab created in the Labs app |
| [**4 — Announce the dataset**](#tutorial-4-announce-the-dataset) | A public update on the lab's activity feed, attaching your file | Tutorial 1 or 2 |

New here? Read [Getting Started](../getting-started/README.md) first — it covers the two prerequisites, what things cost, and which of the four lanes you should be in. If you are an agent, the [one-pager](../getting-started/for-agents.md) is the same flow with no prose.

## Prerequisites

* A **consumer credential** — `mol_<consumerId>_<secret>`. See [Getting Started](../getting-started/README.md#1-a-mol-consumer-credential-the-one-manual-step) for the request template. **No pre-issued Service Token needed** — every tutorial mints its own from a wallet signature.
* A funded EOA on **Base Sepolia** — testnet ETH from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets). `mintFeeWei()` reads **0** on both Base Sepolia and Base mainnet (verified 2026-08-27), so minting costs gas only — but the tutorials read the live value and forward it, so a future fee needs no code change.
* Node 18+ and `viem` (`npm install viem`). `fetch` and `node:crypto` are built in.

Tutorial 3 additionally needs a lab created in the Labs app by a human, and Tutorial 2's verification step needs the wallet to satisfy the file's own access conditions.

***

## Shared setup

Every environment-specific value lives in this one block; swapping to production is a matter of replacing it with the table in [Running in Production](#running-in-production).

```javascript
import { baseSepolia } from "viem/chains"; // production: `base`

// ---- Staging (Base Sepolia) config — see "Running in Production" to swap ----
const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const CHAIN = baseSepolia;
const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90"; // OnChainLabFactory
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28"; // LabNFT (proxy)
const ACCESS_RESOLVER_ADDRESS = "0x5493F472602C87318EA5Eff753cDD593bf9bF559"; // AccessResolver
const ACCESS_CONDITION_CHAIN = "baseSepolia"; // the `chain` string inside accessControlConditions
const LAB_APP_URL = "https://testnet.labs.molecule.xyz"; // production: https://labs.molecule.xyz

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL; // mol_<id>_<secret> — no "Bearer" prefix
const WALLET_PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY;

// Set once Step 1 exchanges a wallet signature for a token. Every call after
// that automatically starts sending it; public queries (like Step 1's own
// sign-in-message lookup) work fine without it.
let serviceToken;

async function graphql(query, variables) {
  // Authorization is always required. X-Service-Token is added once we have
  // one — omit it entirely rather than sending an empty header.
  const headers = { "Content-Type": "application/json", Authorization: CONSUMER_CREDENTIAL };
  if (serviceToken) headers["X-Service-Token"] = serviceToken;

  const res = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });
  const { data, errors } = await res.json();
  // Queries report failure here: a top-level errors[] entry whose errorType is
  // the catalogue code. Mutations report expected failures in-band instead (see
  // assertOk); a top-level entry on a mutation means a transport/infrastructure
  // failure or an invalid request document.
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}

// Mutations report failure in-band: `error` is null on success. Throw on a
// non-null `error` so a failed step stops the workflow with the catalogue
// `code` and the `requestId` to quote in a bug report.
function assertOk(result, op) {
  if (result.error) {
    // `details` is a JSON-encoded string; JSON.parse(result.error.details ?? "{}").reason
    // carries the specific cause when there is one.
    const { code, message, requestId } = result.error;
    throw new Error(`${op} failed: ${code}: ${message} (requestId ${requestId})`);
  }
  return result;
}
```

***

## Tutorial 1: Create a lab and upload a public file

The default path, and the one to run first. Five steps: get a token, mint the LabNFT, register the lab, upload the file, verify. A public file is stored as-is — no key management, no access conditions.

> **Want the file to be confidential instead?** Steps 1–3 are identical; branch at Step 4 into [Tutorial 2](#tutorial-2-upload-an-encrypted-file).

### Step 1: Get a service token

Prove control of the wallet instead of waiting on a manually issued token — the self-service path for agents, bots and CI/CD. Fetch the deterministic sign-in message, sign it as a plain wallet message (EIP-191 `personal_sign` — **not** typed data), then exchange the signature for a token. Full reference: [Service Tokens](service-tokens.md#obtaining-a-token).

```javascript
import { createPublicClient, createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const SERVICE_NAME = "tutorial-agent";

const account = privateKeyToAccount(WALLET_PRIVATE_KEY);
// publicClient: read-only RPC calls (readContract, waitForTransactionReceipt).
const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
// walletClient: everything that needs the private key — signing and sending.
const walletClient = createWalletClient({ account, chain: CHAIN, transport: http() });

const signInMessage = await graphql(
  `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
    getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) {
      message
    }
  }`,
  { walletAddress: account.address, serviceName: SERVICE_NAME },
);

// Sign the message VERBATIM — the backend recomposes and verifies the same
// string server-side, so re-wording or re-formatting it breaks verification.
const messageSignature = await walletClient.signMessage({
  message: signInMessage.getServiceSignInMessage.message,
});

const tokenResult = await graphql(
  `mutation GenerateServiceToken(
    $serviceName: String!
    $walletAddress: String!
    $messageSignature: String!
  ) {
    generateServiceToken(
      serviceName: $serviceName
      walletAddress: $walletAddress
      messageSignature: $messageSignature
    ) {
      token
      tokenId
      expiresAt
      message
      error { code message requestId retryable details }
    }
  }`,
  { serviceName: SERVICE_NAME, walletAddress: account.address, messageSignature },
);
assertOk(tokenResult.generateServiceToken, "generateServiceToken");
serviceToken = tokenResult.generateServiceToken.token;
```

**Expected response:**

```json
{
  "data": {
    "generateServiceToken": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9…",
      "tokenId": "b3f1c0de-…",
      "expiresAt": "2027-02-23T10:31:07.000Z",
      "message": "Service token generated successfully for tutorial-agent",
      "error": null
    }
  }
}
```

**If it fails:**

| `error.code` | What happened | Fix |
| ------------ | ------------- | --- |
| `UNAUTHENTICATED`, `reason: INVALID_SIGNATURE` | The signed bytes are not the message the backend recomposes | Sign the returned `message` string byte-for-byte. Use `personal_sign` / viem's `signMessage`, not `signTypedData` |
| `UNAUTHENTICATED`, `reason: WALLET_MISMATCH` | `walletAddress` isn't the signer | Pass the same address that signed |
| `VALIDATION_FAILED` | Bad `expiresIn` | Format is `<int><unit>`, unit one of `s m h d w M y`; between 1 hour and 2 years |
| HTTP `401` before GraphQL runs | Consumer credential missing or malformed | Check `Authorization` — no `Bearer` prefix. See [Authentication](../authentication.md) |

`expiresIn` defaults to **`180d`** when omitted. The returned token is **wallet-bound, not lab-bound**: it carries this wallet's identity, and authorisation is resolved per request from that wallet's onchain role on whichever lab you name.

### Step 2: Mint the LabNFT

Mint onchain via `OnChainLabFactory.mintAndCreateAccount` and read `oclId` off the `OclIdentityCreated` event. Reuses `account` / `publicClient` / `walletClient` from Step 1 and `FACTORY_ADDRESS` / `LABNFT_ADDRESS` from the config block. Full detail — the fee call and how `oclId` is derived — is on [Lab Management](lab-management.md#mint-the-labnft).

```javascript
import { parseAbi, parseEventLogs } from "viem";

const factoryAbi = parseAbi([
  "function mintAndCreateAccount(address to) external payable returns (address account, uint256 tokenId)",
]);
const labNftAbi = parseAbi([
  "function mintFeeWei() external view returns (uint256)",
  "event OclIdentityCreated(address indexed account, bytes32 indexed oclId, uint256 indexed tokenId, bytes32 salt, uint256 canonicalChainId)",
]);

// Read the fee live — it is 0 on both chains today, but never hardcode it.
const mintFeeWei = await publicClient.readContract({
  address: LABNFT_ADDRESS,
  abi: labNftAbi,
  functionName: "mintFeeWei",
});

const mintTxHash = await walletClient.writeContract({
  address: FACTORY_ADDRESS,
  abi: factoryAbi,
  functionName: "mintAndCreateAccount",
  args: [account.address],
  value: mintFeeWei,
});
const mintReceipt = await publicClient.waitForTransactionReceipt({ hash: mintTxHash });

const [identity] = parseEventLogs({
  abi: labNftAbi,
  eventName: "OclIdentityCreated",
  logs: mintReceipt.logs.filter((l) => l.address.toLowerCase() === LABNFT_ADDRESS.toLowerCase()),
});
const oclId = identity.args.oclId;
const labAccountAddress = identity.args.account;
```

**Expected result:** `mintReceipt.status === "success"`, and

```
oclId:              0x0101000000000000000000000000abc…  (32-byte hex)
tokenId:            1274
labAccountAddress:  0x… (the ERC-6551 Token Bound Account)
```

**If it fails:**

* **Transaction reverts** — the wallet is unfunded, or `value` didn't match `mintFeeWei()`. Read the fee live and forward it; don't hardcode `0`.
* **`parseEventLogs` returns `[]`** — the logs weren't filtered to the LabNFT. `OclIdentityCreated` fires on the **LabNFT contract**, not the factory; the factory's own `AccountProvisioned` event does not carry `oclId` as a topic.

### Step 3: Register the lab

Register the Kamu-backed data room for the freshly-minted `oclId`. Full reference: [Create Lab](lab-management.md#create-lab).

```javascript
const createLabResult = await graphql(
  `mutation CreateLab($oclId: String!) {
    createLab(input: { oclId: $oclId }) {
      message
      error { code message requestId retryable details }
      lab { oclId shortname labAccountAddress labNftTokenId }
    }
  }`,
  { oclId },
);
assertOk(createLabResult.createLab, "createLab");
```

**Expected response:**

```json
{
  "data": {
    "createLab": {
      "message": "Lab created successfully",
      "error": null,
      "lab": {
        "oclId": "0x0101000000000000000000000000abc…",
        "shortname": null,
        "labAccountAddress": "0x…",
        "labNftTokenId": "1274"
      }
    }
  }
}
```

`shortname` is derived server-side from the lab's name and is `null` until it has been derived — that is expected on a lab that has just been minted and not yet named.

**If it fails:**

| `error.code` | What happened | Fix |
| ------------ | ------------- | --- |
| `CONFLICT`, `reason: PROJECT_CONFLICT` | The lab is already registered | Treat as success and continue — this is what a re-run looks like |
| `NOT_FOUND` | The `oclId` isn't indexed yet | The indexer trails the mint by a few seconds. Retry with backoff |
| `UNAUTHENTICATED`, `reason: NO_AUTH` | No `X-Service-Token` and no Privy session | Step 1 didn't set `serviceToken` |
| `UPSTREAM_UNAVAILABLE` | A dependency is down (`reason: KAMU`) | `retryable: true` — retry with backoff |

DID-linking for the new lab starts automatically in the background; [`getDidLinkStatus`](lab-management.md#get-did-link-status) reports its progress. You do not need to wait for it.

### Step 4: Upload the file

Three calls: get a presigned URL, `PUT` the bytes, finalise with metadata. Full reference: [Files](files.md).

```javascript
import { readFileSync } from "node:fs";
import { basename } from "node:path";

const filePath = "./research-data.csv";
const bytes = readFileSync(filePath);

// 4a. Get a presigned URL
const initiateResult = await graphql(
  `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
    initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
      uploadToken
      uploadUrl
      uploadUrlExpiry
      method
      headers { key value }
      error { code message requestId retryable details }
    }
  }`,
  { oclId, contentType: "text/csv", contentLength: bytes.length },
);
assertOk(initiateResult.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
const { uploadToken, uploadUrl, method, headers } = initiateResult.initiateCreateOrUpdateFile;

// 4b. PUT the bytes with EXACTLY the returned headers
const uploadHeaders = {};
headers.forEach((h) => (uploadHeaders[h.key] = h.value));
const putResponse = await fetch(uploadUrl, {
  method: method || "PUT",
  headers: uploadHeaders,
  body: bytes,
});
if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.status} ${putResponse.statusText}`);

// 4c. Finalise
const finishResult = await graphql(
  `mutation Finish(
    $oclId: String!
    $uploadToken: String!
    $path: String!
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
      error { code message requestId retryable details }
    }
  }`,
  {
    oclId,
    uploadToken,
    path: basename(filePath),
    accessLevel: "PUBLIC", // DataRoomAccessLevel: PUBLIC | HOLDERS | ADMIN
    changeBy: account.address,
    description: "Baseline assay results",
    tags: ["preliminary"],
    categories: ["raw-data"],
    contentText: "assay,replicate,value",
  },
);
assertOk(finishResult.finishCreateOrUpdateFile, "finishCreateOrUpdateFile");
const { datasetId } = finishResult.finishCreateOrUpdateFile;
```

**Expected responses:**

```json
{
  "data": {
    "initiateCreateOrUpdateFile": {
      "uploadToken": "eyJ…",
      "uploadUrl": "https://…s3….amazonaws.com/…?X-Amz-Signature=…",
      "uploadUrlExpiry": "2026-08-27T14:05:00.000Z",
      "method": "PUT",
      "headers": [{ "key": "Content-Type", "value": "text/csv" }],
      "error": null
    }
  }
}
```

The `PUT` returns HTTP `200` with an empty body. Then:

```json
{
  "data": {
    "finishCreateOrUpdateFile": {
      "datasetId": "did:odf:fed01…",
      "contentHash": "f162…",
      "version": 1,
      "message": "…",
      "error": null
    }
  }
}
```

`message` on this result is passed through from the storage layer, so its exact wording varies and is deliberately not shown here — it is **not part of the contract**. Assert on `error == null`, never on `message`.

Keep `datasetId` — Tutorial 4 attaches it to an announcement.

**If it fails:**

| Symptom | Cause | Fix |
| ------- | ----- | --- |
| `initiate` → `UNAUTHORIZED` | The wallet behind the token has no write role on this lab | You must be Owner or Contributor. See [Tutorial 3](#tutorial-3-give-your-agent-access-to-a-lab-you-created-in-the-app) |
| `PUT` → `403` | URL expired (~15 min), or headers altered | Re-run `initiate`; send the returned `headers` verbatim |
| `PUT` → `400`/`411` | Body wasn't sent as raw bytes | Send the buffer, not a JSON wrapper. In curl: `--data-binary` |
| `finish` → `VALIDATION_FAILED`, `details.field: "path"` | `path` contains an underscore, or both `path` and `ref` were sent | Underscores are not allowed in `path`; use `path` for a new file **or** `ref` for a new version, never both |
| `finish` → `VALIDATION_FAILED`, `reason: INVALID_TAGS_OR_CATEGORIES` | Unknown tag or category | Valid values come from the public `fileCategoriesAndTags` query |
| `finish` → `VALIDATION_FAILED`, `reason: INVALID_ACCESS_LEVEL` | Bad `accessLevel` | One of `PUBLIC`, `HOLDERS`, `ADMIN` |

### Step 5: Verify it worked

Two checks. The first needs nothing but your consumer credential:

```javascript
const verify = await graphql(
  `query Verify($oclId: String!) {
    labWithDataRoomAndFiles(oclId: $oclId) {
      oclId
      shortname
      name
      dataRoom {
        id
        files { path contentType accessLevel version createdBy downloadUrl }
      }
    }
  }`,
  { oclId },
);

const file = verify.labWithDataRoomAndFiles.dataRoom.files.find(
  (f) => f.path.endsWith(basename(filePath)),
);
if (!file) throw new Error("File not found in the data room");
console.log("Verified:", file.path, file.accessLevel, "v" + file.version);
```

Your file is in `dataRoom.files` with `accessLevel: "PUBLIC"` and `version: 1`. Because the file is public, `downloadUrl` is a fetchable presigned URL — `fetch` it and compare the bytes to what you uploaded for an end-to-end check.

If `labWithDataRoomAndFiles` comes back `null`, the lab is not registered: Step 3 didn't complete. This is one of only two nullable queries on the Labs API, so a missing lab nulls the field instead of throwing.

The second check is visual. Once `shortname` is populated, the lab has a page at `${LAB_APP_URL}/projects/<shortname>` — `https://testnet.labs.molecule.xyz/projects/<shortname>` on staging.

### Tutorial 1: complete script

All five steps in one file, against staging. No pre-issued service token needed.

```javascript
#!/usr/bin/env node
import { readFileSync } from "node:fs";
import { basename } from "node:path";
import {
  createPublicClient,
  createWalletClient,
  http,
  parseAbi,
  parseEventLogs,
} from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { baseSepolia } from "viem/chains"; // production: `base`

// ---- Staging (Base Sepolia) config — see "Running in Production" to swap ----
const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const CHAIN = baseSepolia;
const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90"; // OnChainLabFactory
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28"; // LabNFT (proxy)
const LAB_APP_URL = "https://testnet.labs.molecule.xyz";
const SERVICE_NAME = "tutorial-agent";

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL; // mol_<id>_<secret> — no "Bearer" prefix
const WALLET_PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY;

let serviceToken;

async function graphql(query, variables) {
  const headers = { "Content-Type": "application/json", Authorization: CONSUMER_CREDENTIAL };
  if (serviceToken) headers["X-Service-Token"] = serviceToken;
  const res = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });
  const { data, errors } = await res.json();
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}

function assertOk(result, op) {
  if (result.error) {
    const { code, message, requestId } = result.error;
    throw new Error(`${op} failed: ${code}: ${message} (requestId ${requestId})`);
  }
  return result;
}

async function main() {
  const filePath = process.argv[2];
  if (!filePath) throw new Error("Usage: node tutorial-1.js <file-to-upload>");

  const account = privateKeyToAccount(WALLET_PRIVATE_KEY);
  const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
  const walletClient = createWalletClient({ account, chain: CHAIN, transport: http() });

  // ---- Step 1: service token ----
  const signInMessage = await graphql(
    `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
      getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message }
    }`,
    { walletAddress: account.address, serviceName: SERVICE_NAME },
  );
  const messageSignature = await walletClient.signMessage({
    message: signInMessage.getServiceSignInMessage.message,
  });
  const tokenResult = await graphql(
    `mutation GenerateServiceToken($serviceName: String!, $walletAddress: String!, $messageSignature: String!) {
      generateServiceToken(serviceName: $serviceName, walletAddress: $walletAddress, messageSignature: $messageSignature) {
        token
        error { code message requestId retryable details }
      }
    }`,
    { serviceName: SERVICE_NAME, walletAddress: account.address, messageSignature },
  );
  assertOk(tokenResult.generateServiceToken, "generateServiceToken");
  serviceToken = tokenResult.generateServiceToken.token;
  console.log("1/5 Got service token");

  // ---- Step 2: mint the LabNFT ----
  const factoryAbi = parseAbi([
    "function mintAndCreateAccount(address to) external payable returns (address account, uint256 tokenId)",
  ]);
  const labNftAbi = parseAbi([
    "function mintFeeWei() external view returns (uint256)",
    "event OclIdentityCreated(address indexed account, bytes32 indexed oclId, uint256 indexed tokenId, bytes32 salt, uint256 canonicalChainId)",
  ]);

  const mintFeeWei = await publicClient.readContract({
    address: LABNFT_ADDRESS,
    abi: labNftAbi,
    functionName: "mintFeeWei",
  });
  const mintTxHash = await walletClient.writeContract({
    address: FACTORY_ADDRESS,
    abi: factoryAbi,
    functionName: "mintAndCreateAccount",
    args: [account.address],
    value: mintFeeWei,
  });
  const mintReceipt = await publicClient.waitForTransactionReceipt({ hash: mintTxHash });
  const [identity] = parseEventLogs({
    abi: labNftAbi,
    eventName: "OclIdentityCreated",
    logs: mintReceipt.logs.filter((l) => l.address.toLowerCase() === LABNFT_ADDRESS.toLowerCase()),
  });
  const oclId = identity.args.oclId;
  console.log("2/5 Minted LabNFT — tx:", mintTxHash, "oclId:", oclId);

  // ---- Step 3: register the lab ----
  const createLabResult = await graphql(
    `mutation CreateLab($oclId: String!) {
      createLab(input: { oclId: $oclId }) {
        message
        error { code message requestId retryable details }
        lab { shortname labAccountAddress labNftTokenId }
      }
    }`,
    { oclId },
  );
  assertOk(createLabResult.createLab, "createLab");
  console.log("3/5 Lab registered — TBA:", createLabResult.createLab.lab.labAccountAddress);

  // ---- Step 4: upload the file ----
  const bytes = readFileSync(filePath);
  const initiateResult = await graphql(
    `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
      initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
        uploadToken uploadUrl method headers { key value }
        error { code message requestId retryable details }
      }
    }`,
    { oclId, contentType: "application/octet-stream", contentLength: bytes.length },
  );
  assertOk(initiateResult.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
  const { uploadToken, uploadUrl, method, headers } = initiateResult.initiateCreateOrUpdateFile;

  const uploadHeaders = {};
  headers.forEach((h) => (uploadHeaders[h.key] = h.value));
  const putResponse = await fetch(uploadUrl, {
    method: method || "PUT",
    headers: uploadHeaders,
    body: bytes,
  });
  if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.status} ${putResponse.statusText}`);

  const finishResult = await graphql(
    `mutation Finish($oclId: String!, $uploadToken: String!, $path: String!, $accessLevel: String!, $changeBy: String!) {
      finishCreateOrUpdateFile(oclId: $oclId, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel, changeBy: $changeBy) {
        datasetId version
        error { code message requestId retryable details }
      }
    }`,
    {
      oclId,
      uploadToken,
      path: basename(filePath),
      accessLevel: "PUBLIC",
      changeBy: account.address,
    },
  );
  assertOk(finishResult.finishCreateOrUpdateFile, "finishCreateOrUpdateFile");
  const { datasetId } = finishResult.finishCreateOrUpdateFile;
  console.log("4/5 Uploaded — datasetId:", datasetId);

  // ---- Step 5: verify ----
  const verify = await graphql(
    `query Verify($oclId: String!) {
      labWithDataRoomAndFiles(oclId: $oclId) {
        shortname
        dataRoom { files { path accessLevel version } }
      }
    }`,
    { oclId },
  );
  const lab = verify.labWithDataRoomAndFiles;
  if (!lab) throw new Error("Lab not found — createLab did not complete");
  const file = lab.dataRoom.files.find((f) => f.path.endsWith(basename(filePath)));
  if (!file) throw new Error("File not found in the data room");
  console.log("5/5 Verified:", file.path, file.accessLevel, "v" + file.version);
  if (lab.shortname) console.log("Lab page:", `${LAB_APP_URL}/projects/${lab.shortname}`);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**Usage:**

```bash
WALLET_PRIVATE_KEY="0x..." \
CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret" \
node tutorial-1.js ./research-data.csv
```

***

## Tutorial 2: Upload an encrypted file

Same lab, same three-call upload — but the bytes are AES-256-GCM encrypted locally before they leave your machine, and decryption is gated on live onchain state. Encryption is a first-class flow, not an appendix: reach for it whenever the file is confidential and access should follow the lab's roles.

**Steps 1–3 are identical to [Tutorial 1](#tutorial-1-create-a-lab-and-upload-a-public-file)** — get a service token, mint, register. Pick up here with `oclId`, `labAccountAddress`, `account` and `serviceToken` already in hand (or with any lab you already hold a role on).

Conceptually: the backend hands you a one-shot data encryption key (DEK) in two forms — plaintext, and wrapped by the key custodian. You encrypt with the plaintext copy, throw it away, and store the wrapped copy in the file's metadata alongside the conditions under which the custodian may unwrap it again. Full model: [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md).

### Step 4a: Get a DEK

```javascript
const dekResult = await graphql(`
  mutation {
    generateDataEncryptionKey {
      plaintextDEK
      encryptedDek
      encryptionSystem
      error { code message requestId retryable details }
    }
  }
`);
assertOk(dekResult.generateDataEncryptionKey, "generateDataEncryptionKey");
const { plaintextDEK, encryptedDek, encryptionSystem } = dekResult.generateDataEncryptionKey;
```

**Expected response:**

```json
{
  "data": {
    "generateDataEncryptionKey": {
      "plaintextDEK": "3q2+7wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
      "encryptedDek": "AQIDAHjR…",
      "encryptionSystem": "kms",
      "error": null
    }
  }
}
```

`encryptionSystem` is **backend-set** — echo it verbatim in Step 4d, never hardcode `"kms"`. Key custody is on a path to threshold cryptography, and echoing the value is what keeps your integration working across that change.

Requires authentication (service token or Privy session), so Step 1 must have run.

### Step 4b: Encrypt locally

```javascript
import { webcrypto, randomBytes, createHash } from "node:crypto";
import { readFileSync } from "node:fs";
import { basename } from "node:path";

const filePath = "./confidential-results.csv";
const plaintext = readFileSync(filePath);

const cryptoKey = await webcrypto.subtle.importKey(
  "raw",
  Buffer.from(plaintextDEK, "base64"),
  "AES-GCM",
  false, // not extractable — the key cannot be read back out
  ["encrypt"],
);
const iv = randomBytes(12); // 96-bit IV, fresh per file — never reuse one
const ciphertext = Buffer.from(
  await webcrypto.subtle.encrypt({ name: "AES-GCM", iv }, cryptoKey, plaintext),
);
// Hash of the PLAINTEXT — this is what a reader checks after decrypting.
const contentHashHex = "sha256-" + createHash("sha256").update(plaintext).digest("hex");
```

The plaintext DEK is now unreferenced; don't log it, don't persist it, don't send it anywhere. Web Crypto's `AES-GCM` appends the 16-byte authentication tag to the ciphertext, which is what the decrypt side expects.

### Step 4c: Write the access conditions

`accessControlConditions` is a JSON-stringified array of predicates the backend re-evaluates against live chain state every time someone asks to decrypt. Two recipes cover almost everything.

**Owner only** — only the LabNFT owner (and authorised signers of its Token Bound Account, so a Safe's signers resolve through) can decrypt:

```javascript
const ownerOnlyConditions = JSON.stringify([
  {
    conditionType: "evmContract",
    contractAddress: ACCESS_RESOLVER_ADDRESS,
    chain: ACCESS_CONDITION_CHAIN,
    functionName: "isAuthorizedSignerForTba",
    functionParams: [":userAddress", labAccountAddress],
    functionAbi: {
      name: "isAuthorizedSignerForTba",
      inputs: [
        { name: "signer", type: "address" },
        { name: "account", type: "address" },
      ],
      outputs: [{ name: "", type: "bool" }],
      stateMutability: "view",
      type: "function",
    },
    returnValueTest: { key: "", comparator: "=", value: "true" },
  },
]);
```

**Owner or Contributor or Viewer** — the recipe to use when the lab has a team, and the one Tutorial 3's agent needs. `hasRole` is hierarchical (`ROLE_VIEWER = 1`; a Contributor and the Owner both pass a Viewer check), and the explicit `isAuthorizedSignerForTba` branch keeps the Owner covered even if conditions are ever evaluated against a non-canonical chain:

```javascript
const accessResolverAbi = {
  isAuthorizedSignerForTba: {
    name: "isAuthorizedSignerForTba",
    inputs: [
      { name: "signer", type: "address" },
      { name: "account", type: "address" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "view",
    type: "function",
  },
  hasRole: {
    name: "hasRole",
    inputs: [
      { name: "oclId", type: "bytes32" },
      { name: "account", type: "address" },
      { name: "role", type: "uint8" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "view",
    type: "function",
  },
};

const teamConditions = JSON.stringify([
  {
    conditionType: "evmContract",
    contractAddress: ACCESS_RESOLVER_ADDRESS,
    chain: ACCESS_CONDITION_CHAIN,
    functionName: "isAuthorizedSignerForTba",
    functionParams: [":userAddress", labAccountAddress],
    functionAbi: accessResolverAbi.isAuthorizedSignerForTba,
    returnValueTest: { key: "", comparator: "=", value: "true" },
  },
  { operator: "or" },
  {
    conditionType: "evmContract",
    contractAddress: ACCESS_RESOLVER_ADDRESS,
    chain: ACCESS_CONDITION_CHAIN,
    functionName: "hasRole",
    functionParams: [oclId, ":userAddress", "1"], // "1" = ROLE_VIEWER; "2" = ROLE_CONTRIBUTOR and up
    functionAbi: accessResolverAbi.hasRole,
    returnValueTest: { key: "", comparator: "=", value: "true" },
  },
]);
```

`:userAddress` is substituted with the authenticated caller's wallet at evaluation time. Pass `"2"` instead of `"1"` to exclude Viewers. Evaluation walks the array left to right and short-circuits (`or` stops at the first true); **any RPC error fails closed** and the DEK is not released. The full condition grammar, including `EvmBasicCondition`, is on [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md#worked-example-encrypt-for-owner-or-contributor-or-viewer).

### Step 4d: Upload the ciphertext

The same three calls as Tutorial 1, with the ciphertext in place of the raw file and `encryptionMetadata` attached on finish.

```javascript
const initiateResult = await graphql(
  `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
    initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
      uploadToken uploadUrl method headers { key value }
      error { code message requestId retryable details }
    }
  }`,
  // Content-length is the CIPHERTEXT length, and the type is opaque bytes.
  { oclId, contentType: "application/octet-stream", contentLength: ciphertext.length },
);
assertOk(initiateResult.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
const { uploadToken, uploadUrl, method, headers } = initiateResult.initiateCreateOrUpdateFile;

const uploadHeaders = {};
headers.forEach((h) => (uploadHeaders[h.key] = h.value));
const putResponse = await fetch(uploadUrl, {
  method: method || "PUT",
  headers: uploadHeaders,
  body: ciphertext,
});
if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.status} ${putResponse.statusText}`);

const finishResult = await graphql(
  `mutation Finish(
    $oclId: String!
    $uploadToken: String!
    $path: String!
    $accessLevel: String!
    $changeBy: String!
    $encryptionMetadata: EncryptionMetadataInput
  ) {
    finishCreateOrUpdateFile(
      oclId: $oclId
      uploadToken: $uploadToken
      path: $path
      accessLevel: $accessLevel
      changeBy: $changeBy
      encryptionMetadata: $encryptionMetadata
    ) {
      datasetId
      version
      message
      error { code message requestId retryable details }
    }
  }`,
  {
    oclId,
    uploadToken,
    path: basename(filePath),
    accessLevel: "HOLDERS", // encrypted files use HOLDERS or ADMIN, not PUBLIC
    changeBy: account.address,
    encryptionMetadata: {
      encryptionSystem, // echo verbatim — never hardcode
      encryptedDek,
      iv: iv.toString("base64"),
      contentHash: contentHashHex,
      accessControlConditions: teamConditions,
      encryptedBy: account.address.toLowerCase(),
      encryptedAt: new Date().toISOString(),
    },
  },
);
assertOk(finishResult.finishCreateOrUpdateFile, "finishCreateOrUpdateFile");
```

**If it fails:**

| `error.code` | What happened | Fix |
| ------------ | ------------- | --- |
| `VALIDATION_FAILED`, `reason: INVALID_CONDITIONS` | `accessControlConditions` isn't a valid stringified condition array | It is a **JSON string**, not an object. Check `functionAbi` is complete and `returnValueTest` present |
| `VALIDATION_FAILED`, `reason: INVALID_ACCESS_LEVEL` | `PUBLIC` on an encrypted file | Use `HOLDERS` or `ADMIN` |
| `UNAUTHORIZED` on `generateDataEncryptionKey` | No write role on the lab | Owner or Contributor required |

### Step 5: Verify by decrypting it

The real test is a round trip: ask the backend to unwrap the DEK, decrypt, and compare hashes. `decryptDataKey` re-evaluates the file's conditions against **live** chain state and your token's wallet, so a success here proves the gate works.

```javascript
const decryptResult = await graphql(
  `mutation DecryptDataKey($oclId: String!, $filePath: String!) {
    decryptDataKey(oclId: $oclId, filePath: $filePath) {
      plaintextDEK
      iv
      error { code message requestId retryable details }
    }
  }`,
  { oclId, filePath: basename(filePath) },
);
assertOk(decryptResult.decryptDataKey, "decryptDataKey");
const { plaintextDEK: unwrappedDEK, iv: returnedIv } = decryptResult.decryptDataKey;

// Fetch the ciphertext back from the data room
const fileQuery = await graphql(
  `query GetFile($oclId: String!, $path: String!) {
    dataRoomFile(oclId: $oclId, path: $path) {
      path
      accessLevel
      downloadUrl
      encryptionMetadata { encryptionSystem contentHash encryptedBy encryptedAt }
    }
  }`,
  { oclId, path: basename(filePath) },
);
const downloaded = Buffer.from(
  await (await fetch(fileQuery.dataRoomFile.downloadUrl)).arrayBuffer(),
);

const decryptKey = await webcrypto.subtle.importKey(
  "raw",
  Buffer.from(unwrappedDEK, "base64"),
  "AES-GCM",
  false,
  ["decrypt"],
);
const recovered = Buffer.from(
  await webcrypto.subtle.decrypt(
    { name: "AES-GCM", iv: Buffer.from(returnedIv, "base64") },
    decryptKey,
    downloaded,
  ),
);

const recoveredHash = "sha256-" + createHash("sha256").update(recovered).digest("hex");
if (recoveredHash !== fileQuery.dataRoomFile.encryptionMetadata.contentHash) {
  throw new Error("Content hash mismatch after decryption");
}
console.log("Round trip verified —", recovered.length, "bytes recovered");
```

**Expected:** the hashes match and `recovered` equals your original file byte-for-byte.

**If it fails:**

| `error.code` | What happened | Fix |
| ------------ | ------------- | --- |
| `UNAUTHORIZED` | Your wallet does not satisfy the file's conditions | Check the role with the public `listLabMembers(oclId)` query. After a fresh grant, allow for indexer/chain lag and retry |
| `FAILED_PRECONDITION`, `reason: LEGACY_ENCRYPTION` | The file predates onchain-verified envelope encryption | Not decryptable through this mutation; use the original encryption client |
| `FAILED_PRECONDITION`, `reason: NOT_ENCRYPTED` / `MISSING_DEK` | The file has no `encryptionMetadata` | You're pointing at a public file |
| `UPSTREAM_UNAVAILABLE` | Condition evaluation could not reach the chain RPC | Fails closed by design. `retryable: true` |
| Web Crypto throws `OperationError` | Wrong IV, or ciphertext truncated | Use the `iv` **returned by `decryptDataKey`**, and pass the whole downloaded body including the trailing GCM tag |

### Tutorial 2: complete script

Steps 1–3 are Tutorial 1's verbatim; this script carries them so it runs standalone.

```javascript
#!/usr/bin/env node
import { webcrypto, randomBytes, createHash } from "node:crypto";
import { readFileSync } from "node:fs";
import { basename } from "node:path";
import {
  createPublicClient,
  createWalletClient,
  http,
  parseAbi,
  parseEventLogs,
} from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { baseSepolia } from "viem/chains"; // production: `base`

const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const CHAIN = baseSepolia;
const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90";
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28";
const ACCESS_RESOLVER_ADDRESS = "0x5493F472602C87318EA5Eff753cDD593bf9bF559";
const ACCESS_CONDITION_CHAIN = "baseSepolia";
const SERVICE_NAME = "tutorial-agent";

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL;
const WALLET_PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY;
// Optional: reuse an existing lab instead of minting a new one.
const EXISTING_OCL_ID = process.env.OCL_ID;

let serviceToken;

async function graphql(query, variables) {
  const headers = { "Content-Type": "application/json", Authorization: CONSUMER_CREDENTIAL };
  if (serviceToken) headers["X-Service-Token"] = serviceToken;
  const res = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });
  const { data, errors } = await res.json();
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}

function assertOk(result, op) {
  if (result.error) {
    const { code, message, requestId } = result.error;
    throw new Error(`${op} failed: ${code}: ${message} (requestId ${requestId})`);
  }
  return result;
}

const accessResolverAbi = {
  isAuthorizedSignerForTba: {
    name: "isAuthorizedSignerForTba",
    inputs: [
      { name: "signer", type: "address" },
      { name: "account", type: "address" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "view",
    type: "function",
  },
  hasRole: {
    name: "hasRole",
    inputs: [
      { name: "oclId", type: "bytes32" },
      { name: "account", type: "address" },
      { name: "role", type: "uint8" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "view",
    type: "function",
  },
};

function buildTeamConditions(oclId, labAccountAddress) {
  return JSON.stringify([
    {
      conditionType: "evmContract",
      contractAddress: ACCESS_RESOLVER_ADDRESS,
      chain: ACCESS_CONDITION_CHAIN,
      functionName: "isAuthorizedSignerForTba",
      functionParams: [":userAddress", labAccountAddress],
      functionAbi: accessResolverAbi.isAuthorizedSignerForTba,
      returnValueTest: { key: "", comparator: "=", value: "true" },
    },
    { operator: "or" },
    {
      conditionType: "evmContract",
      contractAddress: ACCESS_RESOLVER_ADDRESS,
      chain: ACCESS_CONDITION_CHAIN,
      functionName: "hasRole",
      functionParams: [oclId, ":userAddress", "1"],
      functionAbi: accessResolverAbi.hasRole,
      returnValueTest: { key: "", comparator: "=", value: "true" },
    },
  ]);
}

async function main() {
  const filePath = process.argv[2];
  if (!filePath) throw new Error("Usage: node tutorial-2.js <file-to-encrypt-and-upload>");

  const account = privateKeyToAccount(WALLET_PRIVATE_KEY);
  const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
  const walletClient = createWalletClient({ account, chain: CHAIN, transport: http() });

  // ---- Step 1: service token ----
  const signInMessage = await graphql(
    `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
      getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message }
    }`,
    { walletAddress: account.address, serviceName: SERVICE_NAME },
  );
  const messageSignature = await walletClient.signMessage({
    message: signInMessage.getServiceSignInMessage.message,
  });
  const tokenResult = await graphql(
    `mutation GenerateServiceToken($serviceName: String!, $walletAddress: String!, $messageSignature: String!) {
      generateServiceToken(serviceName: $serviceName, walletAddress: $walletAddress, messageSignature: $messageSignature) {
        token
        error { code message requestId retryable details }
      }
    }`,
    { serviceName: SERVICE_NAME, walletAddress: account.address, messageSignature },
  );
  assertOk(tokenResult.generateServiceToken, "generateServiceToken");
  serviceToken = tokenResult.generateServiceToken.token;
  console.log("1/6 Got service token");

  // ---- Steps 2 & 3: mint + register (skipped when OCL_ID is provided) ----
  let oclId = EXISTING_OCL_ID;
  let labAccountAddress;

  if (oclId) {
    const existing = await graphql(
      `query($oclId: String!) { labWithDataRoomAndFiles(oclId: $oclId) { labAccountAddress } }`,
      { oclId },
    );
    if (!existing.labWithDataRoomAndFiles) throw new Error(`Lab ${oclId} not found`);
    labAccountAddress = existing.labWithDataRoomAndFiles.labAccountAddress;
    console.log("2-3/6 Reusing lab", oclId);
  } else {
    const factoryAbi = parseAbi([
      "function mintAndCreateAccount(address to) external payable returns (address account, uint256 tokenId)",
    ]);
    const labNftAbi = parseAbi([
      "function mintFeeWei() external view returns (uint256)",
      "event OclIdentityCreated(address indexed account, bytes32 indexed oclId, uint256 indexed tokenId, bytes32 salt, uint256 canonicalChainId)",
    ]);
    const mintFeeWei = await publicClient.readContract({
      address: LABNFT_ADDRESS,
      abi: labNftAbi,
      functionName: "mintFeeWei",
    });
    const mintTxHash = await walletClient.writeContract({
      address: FACTORY_ADDRESS,
      abi: factoryAbi,
      functionName: "mintAndCreateAccount",
      args: [account.address],
      value: mintFeeWei,
    });
    const mintReceipt = await publicClient.waitForTransactionReceipt({ hash: mintTxHash });
    const [identity] = parseEventLogs({
      abi: labNftAbi,
      eventName: "OclIdentityCreated",
      logs: mintReceipt.logs.filter((l) => l.address.toLowerCase() === LABNFT_ADDRESS.toLowerCase()),
    });
    oclId = identity.args.oclId;
    labAccountAddress = identity.args.account;
    console.log("2/6 Minted LabNFT — oclId:", oclId);

    const createLabResult = await graphql(
      `mutation CreateLab($oclId: String!) {
        createLab(input: { oclId: $oclId }) {
          error { code message requestId retryable details }
          lab { labAccountAddress }
        }
      }`,
      { oclId },
    );
    assertOk(createLabResult.createLab, "createLab");
    console.log("3/6 Lab registered");
  }

  // ---- Step 4a: DEK ----
  const dekResult = await graphql(`
    mutation {
      generateDataEncryptionKey {
        plaintextDEK encryptedDek encryptionSystem
        error { code message requestId retryable details }
      }
    }
  `);
  assertOk(dekResult.generateDataEncryptionKey, "generateDataEncryptionKey");
  const { plaintextDEK, encryptedDek, encryptionSystem } = dekResult.generateDataEncryptionKey;

  // ---- Step 4b: encrypt locally ----
  const plaintext = readFileSync(filePath);
  const cryptoKey = await webcrypto.subtle.importKey(
    "raw",
    Buffer.from(plaintextDEK, "base64"),
    "AES-GCM",
    false,
    ["encrypt"],
  );
  const iv = randomBytes(12);
  const ciphertext = Buffer.from(
    await webcrypto.subtle.encrypt({ name: "AES-GCM", iv }, cryptoKey, plaintext),
  );
  const contentHashHex = "sha256-" + createHash("sha256").update(plaintext).digest("hex");
  console.log("4/6 Encrypted locally —", plaintext.length, "→", ciphertext.length, "bytes");

  // ---- Step 4c + 4d: conditions + upload ----
  const initiateResult = await graphql(
    `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
      initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
        uploadToken uploadUrl method headers { key value }
        error { code message requestId retryable details }
      }
    }`,
    { oclId, contentType: "application/octet-stream", contentLength: ciphertext.length },
  );
  assertOk(initiateResult.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
  const { uploadToken, uploadUrl, method, headers } = initiateResult.initiateCreateOrUpdateFile;

  const uploadHeaders = {};
  headers.forEach((h) => (uploadHeaders[h.key] = h.value));
  const putResponse = await fetch(uploadUrl, {
    method: method || "PUT",
    headers: uploadHeaders,
    body: ciphertext,
  });
  if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.status} ${putResponse.statusText}`);

  const finishResult = await graphql(
    `mutation Finish($oclId: String!, $uploadToken: String!, $path: String!, $accessLevel: String!, $changeBy: String!, $encryptionMetadata: EncryptionMetadataInput) {
      finishCreateOrUpdateFile(oclId: $oclId, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel, changeBy: $changeBy, encryptionMetadata: $encryptionMetadata) {
        datasetId version
        error { code message requestId retryable details }
      }
    }`,
    {
      oclId,
      uploadToken,
      path: basename(filePath),
      accessLevel: "HOLDERS",
      changeBy: account.address,
      encryptionMetadata: {
        encryptionSystem,
        encryptedDek,
        iv: iv.toString("base64"),
        contentHash: contentHashHex,
        accessControlConditions: buildTeamConditions(oclId, labAccountAddress),
        encryptedBy: account.address.toLowerCase(),
        encryptedAt: new Date().toISOString(),
      },
    },
  );
  assertOk(finishResult.finishCreateOrUpdateFile, "finishCreateOrUpdateFile");
  console.log("5/6 Uploaded — datasetId:", finishResult.finishCreateOrUpdateFile.datasetId);

  // ---- Step 5: verify by decrypting ----
  const decryptResult = await graphql(
    `mutation DecryptDataKey($oclId: String!, $filePath: String!) {
      decryptDataKey(oclId: $oclId, filePath: $filePath) {
        plaintextDEK iv
        error { code message requestId retryable details }
      }
    }`,
    { oclId, filePath: basename(filePath) },
  );
  assertOk(decryptResult.decryptDataKey, "decryptDataKey");

  const fileQuery = await graphql(
    `query GetFile($oclId: String!, $path: String!) {
      dataRoomFile(oclId: $oclId, path: $path) {
        downloadUrl
        encryptionMetadata { contentHash }
      }
    }`,
    { oclId, path: basename(filePath) },
  );
  const downloaded = Buffer.from(
    await (await fetch(fileQuery.dataRoomFile.downloadUrl)).arrayBuffer(),
  );
  const decryptKey = await webcrypto.subtle.importKey(
    "raw",
    Buffer.from(decryptResult.decryptDataKey.plaintextDEK, "base64"),
    "AES-GCM",
    false,
    ["decrypt"],
  );
  const recovered = Buffer.from(
    await webcrypto.subtle.decrypt(
      { name: "AES-GCM", iv: Buffer.from(decryptResult.decryptDataKey.iv, "base64") },
      decryptKey,
      downloaded,
    ),
  );
  const recoveredHash = "sha256-" + createHash("sha256").update(recovered).digest("hex");
  if (recoveredHash !== fileQuery.dataRoomFile.encryptionMetadata.contentHash) {
    throw new Error("Content hash mismatch after decryption");
  }
  console.log("6/6 Round trip verified —", recovered.length, "bytes recovered");
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**Usage:**

```bash
WALLET_PRIVATE_KEY="0x..." \
CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret" \
node tutorial-2.js ./confidential-results.csv

# or against a lab you already have
OCL_ID="0x0101…" WALLET_PRIVATE_KEY="0x..." CONSUMER_CREDENTIAL="mol_…" \
node tutorial-2.js ./confidential-results.csv
```

***

## Tutorial 3: Give your agent access to a lab you created in the app

The most common real-world shape: a researcher created their Lab in the Labs app with an email address — no wallet, no code — and now wants an agent contributing to it. The agent gets its **own** identity rather than borrowing the human's; the human grants it a role; the agent authenticates itself from then on.

**Who does what:**

| # | Actor | Action |
| - | ----- | ------ |
| 1 | Agent | Generate a wallet and report its address |
| 2 | Human | Add that address to the lab as **Contributor**, flagged as an agent, with an expiry |
| 3 | Agent | Self-issue a service token by signing the sign-in message |
| 4 | Agent | Upload files and announce; the human sees the result in the app |

The human never hands over a private key, a token, or their session. Revoking the agent is one onchain revoke, and it does not touch anything else.

### Step 1: The agent reports its address

With the plugin: run `wallet_address`. With viem:

```javascript
import { privateKeyToAccount, generatePrivateKey } from "viem/accounts";

// Generate once, store it as the agent's own secret — never the human's key.
const agentPrivateKey = process.env.AGENT_PRIVATE_KEY ?? generatePrivateKey();
const agentAccount = privateKeyToAccount(agentPrivateKey);
console.log("Agent wallet address:", agentAccount.address);
```

Give that address to the lab owner. The agent needs **no gas** for this tutorial — it never sends a transaction, only signs a message. (Persist `agentPrivateKey` if you generated it, or the next run is a different agent with no role.)

### Step 2: The human grants Contributor

In the Labs app, the lab owner adds the agent's address to the lab's members and grants it the **Contributor** role, setting:

* **`isAgent = true`** — informational metadata that marks the member as an agent identity in the members list and UI. It does not change authorisation.
* **an expiry** — typically the agent's session lifetime. When it lapses the agent loses access until it is re-granted; a permanent grant is possible but not the default you want for an agent.

Members can be invited by wallet address, ENS name or email, and the app sponsors the gas for the grant. Under the hood this is one onchain call on the `AccessResolver`, which the owner (or an existing Contributor, for Viewer grants) can also make directly:

```solidity
function grantRole(bytes32 oclId, address account, uint8 role, uint64 expiry, bool isAgent) external;
// role: 2 = ROLE_CONTRIBUTOR, 1 = ROLE_VIEWER
```

Only the **Owner** may grant Contributor. Contributors can grant Viewer, but not Contributor. Full capability matrix: [Roles & Permissions](../../technical-deep-dive/roles-and-permissions.md).

**Why Contributor and not Viewer:** a Viewer can decrypt and read but cannot write. Uploading files and posting announcements needs Contributor.

Both parties can confirm the grant landed with a public query — no authentication beyond the consumer credential:

```javascript
const members = await graphql(
  `query ListLabMembers($oclId: String!) {
    listLabMembers(oclId: $oclId) {
      members { walletAddress role isAgent expiry grantedAt }
    }
  }`,
  { oclId },
);
const grant = members.listLabMembers.members.find(
  (m) => m.walletAddress.toLowerCase() === agentAccount.address.toLowerCase(),
);
console.log("Agent role:", grant?.role, "expiry:", grant?.expiry ?? "permanent");
```

Expect `role: "CONTRIBUTOR"` and `isAgent: true`. `expiry` is unix seconds as a decimal string, or `null` for a permanent grant. Expired grants are excluded from this list entirely, so a missing entry after a while means the grant lapsed.

### Step 3: The agent self-issues a service token

Identical to Tutorial 1 Step 1, signed by the **agent's** wallet:

```javascript
const AGENT_SERVICE_NAME = "research-agent-1";

const signInMessage = await graphql(
  `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
    getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message }
  }`,
  { walletAddress: agentAccount.address, serviceName: AGENT_SERVICE_NAME },
);

const messageSignature = await agentAccount.signMessage({
  message: signInMessage.getServiceSignInMessage.message,
});

const tokenResult = await graphql(
  `mutation GenerateServiceToken(
    $serviceName: String!
    $walletAddress: String!
    $messageSignature: String!
    $expiresIn: String
  ) {
    generateServiceToken(
      serviceName: $serviceName
      walletAddress: $walletAddress
      messageSignature: $messageSignature
      expiresIn: $expiresIn
    ) {
      token tokenId expiresAt
      error { code message requestId retryable details }
    }
  }`,
  {
    serviceName: AGENT_SERVICE_NAME,
    walletAddress: agentAccount.address,
    messageSignature,
    expiresIn: "30d", // match the role grant's expiry rather than taking the 180d default
  },
);
assertOk(tokenResult.generateServiceToken, "generateServiceToken");
serviceToken = tokenResult.generateServiceToken.token;
```

Issuance is **not** gated on holding a role — any wallet can mint a token for itself. The role is what makes the token *useful*: authorisation is resolved per request from the token's wallet against the lab you name. So a token issued before the grant lands keeps working once it does; you do not need to re-issue it.

### Step 4: The agent uploads and announces

From here the agent is an ordinary caller. Run [Tutorial 1 Step 4](#step-4-upload-the-file) with `changeBy: agentAccount.address`, or [Tutorial 2](#tutorial-2-upload-an-encrypted-file) for a confidential file — the `hasRole` branch of the team conditions is exactly what lets the agent decrypt too. Then [Tutorial 4](#tutorial-4-announce-the-dataset) to surface it on the lab's feed.

Writes by a Contributor service token are gated per mutation, matching the Privy user path: `initiateCreateOrUpdateFile`, `finishCreateOrUpdateFile`, `createAnnouncement`, `deleteDataRoomFile`, `updateFileMetadata` and `moveEntry` all accept Contributor. A few surfaces remain **Owner-only** and an agent Contributor cannot reach them: `updateLabNftMetadata`, `generateLabImageUploadUrl` and the legal-agreement mutations.

{% hint style="warning" %}
**Retry on `UNAUTHORIZED` right after the grant.** Role state reaches the API through an event indexer, so there is a short window after `grantRole` confirms onchain in which a write still returns `UNAUTHORIZED` (`reason: NOT_CONTRIBUTOR`). It is not a permissions problem and re-issuing the token will not help — wait and retry:

```javascript
async function withRoleGrantRetry(fn, { attempts = 5, baseMs = 2000 } = {}) {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      const isAuthzLag = /UNAUTHORIZED/.test(String(err));
      if (!isAuthzLag || i === attempts - 1) throw err;
      await new Promise((r) => setTimeout(r, baseMs * 2 ** i)); // 2s, 4s, 8s, 16s
    }
  }
}

await withRoleGrantRetry(() => uploadFile(oclId, "./findings.csv"));
```
{% endhint %}

### Step 5: Verify from both sides

**The agent** verifies as in Tutorial 1 Step 5 — the file is in `dataRoom.files` with `createdBy` set to the agent's address:

```javascript
const verify = await graphql(
  `query Verify($oclId: String!) {
    labWithDataRoomAndFiles(oclId: $oclId) {
      shortname
      dataRoom { files { path accessLevel version createdBy } }
      announcements { headline changeBy }
    }
  }`,
  { oclId },
);
```

**The human** verifies in the app: the file appears in the lab's data room and the announcement on its activity feed, both attributed to the agent's address, which the members list shows flagged as an agent.

### Revoking the agent

One onchain call, and the agent's writes stop:

```solidity
function revokeRole(bytes32 oclId, address account) external;   // Owner only, for a Contributor
```

Or let the grant's `expiry` lapse. Independently, the agent's token can be killed with `revokeServiceToken(tokenId)` — a token can only revoke or extend **its own** record, so one agent cannot interfere with another's.

***

## Tutorial 4: Announce the dataset

An announcement is the lab's public update stream: a headline, a body, and optionally the datasets it is about. Attaching the file makes the announcement the discoverable surface for it — announcements are indexed by `searchLabs` alongside files, and they appear on the lab's activity feed and public page.

Requires Owner or Contributor (a Viewer cannot announce). Pick up with `oclId`, `serviceToken` and the `datasetId` returned by `finishCreateOrUpdateFile`.

```javascript
const announcementResult = await graphql(
  `mutation CreateAnnouncement(
    $oclId: String!
    $headline: String!
    $body: String!
    $attachments: [String!]
  ) {
    createAnnouncement(oclId: $oclId, headline: $headline, body: $body, attachments: $attachments) {
      message
      error { code message requestId retryable details }
    }
  }`,
  {
    oclId,
    headline: "Baseline assay results published",
    body: "First replicate set for the ApoB series. 240 samples, three conditions. Raw CSV attached.",
    attachments: [datasetId], // the datasetId from finishCreateOrUpdateFile
  },
);
assertOk(announcementResult.createAnnouncement, "createAnnouncement");
```

**Expected response:**

```json
{
  "data": {
    "createAnnouncement": {
      "message": "…",
      "error": null
    }
  }
}
```

`createAnnouncement` returns no announcement object, and its `message` is passed through from the storage layer rather than being a fixed string — so success is `error == null` and nothing else. Verify by reading the announcement back.

**If it fails:**

| `error.code` | What happened | Fix |
| ------------ | ------------- | --- |
| `UNAUTHORIZED` | Caller is a Viewer, or has no role | Contributor or Owner required. Fresh grant? See the [retry note](#step-4-the-agent-uploads-and-announces) |
| `NOT_FOUND` | An `attachments` entry isn't a dataset in this lab | Pass the exact `datasetId` strings from `finishCreateOrUpdateFile`, from this lab |
| `VALIDATION_FAILED` | Empty `headline` or `body` | Both are required and non-empty |

### Verify it worked

```javascript
const feed = await graphql(
  `query LabActivity($oclId: String!) {
    labActivity(oclId: $oclId, page: 0, perPage: 10, filter: ANNOUNCEMENT) {
      nodes {
        __typename
        ... on LabEventAnnouncement {
          announcement {
            id
            headline
            body
            changeBy
            eventTime
            attachments { path contentType accessLevel }
          }
        }
      }
      pageInfo { totalPages currentPage }
    }
  }`,
  { oclId },
);
console.log(JSON.stringify(feed.labActivity.nodes[0], null, 2));
```

Your announcement is the newest node, with `attachments` resolved to the full file objects — not just ids — and `changeBy` set to the wallet that posted it. It is also on the lab's public page at `${LAB_APP_URL}/projects/<shortname>`.

`labActivity` is a **public** query: anyone with a consumer credential can read the feed, which is the point of an announcement.

***

## Running in Production

Everything above runs against staging (Base Sepolia, testnet funds). To run the same scripts against production, replace the values in the config block — nothing else changes, since every step reads from these constants:

| Constant | Staging (these tutorials) | Production |
| -------- | ------------------------- | ---------- |
| `GRAPHQL_URL` | `https://staging.graphql.api.molecule.xyz/graphql` | `https://production.graphql.api.molecule.xyz/graphql` |
| `CHAIN` (viem import) | `baseSepolia` from `viem/chains` | `base` from `viem/chains` |
| `FACTORY_ADDRESS` | `0xd629FE2310b4309a212495F10A47f8436dcEfD90` | `0xECdF4f05384056507485C90aeAb0a83268760D6E` |
| `LABNFT_ADDRESS` | `0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28` | `0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92` |
| `ACCESS_RESOLVER_ADDRESS` | `0x5493F472602C87318EA5Eff753cDD593bf9bF559` | `0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B` |
| `ACCESS_CONDITION_CHAIN` | `"baseSepolia"` | `"base"` |
| `LAB_APP_URL` | `https://testnet.labs.molecule.xyz` | `https://labs.molecule.xyz` |

```javascript
import { base } from "viem/chains"; // instead of baseSepolia

const GRAPHQL_URL = "https://production.graphql.api.molecule.xyz/graphql";
const CHAIN = base;
const FACTORY_ADDRESS = "0xECdF4f05384056507485C90aeAb0a83268760D6E";
const LABNFT_ADDRESS = "0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92";
const ACCESS_RESOLVER_ADDRESS = "0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B";
const ACCESS_CONDITION_CHAIN = "base";
const LAB_APP_URL = "https://labs.molecule.xyz";
```

A few things follow automatically from that swap:

* **Headers and the `graphql()` helper** are identical — `Authorization` (consumer credential, no `Bearer`) and the self-issued `X-Service-Token` work the same against both endpoints. Note that credentials are **per environment**: a staging `mol_` credential does not authenticate against production.
* **The `mintFeeWei()` read** already queries the live contract, so it picks up whatever fee production has configured with no code change. It reads `0` today on both chains.
* **The condition ABIs** are unchanged; only `contractAddress` and `chain` differ, and both come from the config block.

What doesn't follow automatically, and is on you:

* **Real funds.** Minting on `base` spends real ETH. Test on staging first.
* **Introspection is off in production** and query depth is capped at 10. Generate types against staging — see [Getting the schema](../getting-started/README.md#getting-the-schema).
* **`SERVICE_NAME`** should identify the real integration; it is echoed into the sign-in message and stored against the issued token.
* Full deployment list, including every other OCL contract on both chains: [Contracts reference](../../references/contracts/README.md).
