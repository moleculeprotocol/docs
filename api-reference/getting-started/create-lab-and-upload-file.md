---
description: >-
  The default path: self-issue a token, mint a LabNFT, register the lab, upload
  a public file, and verify it landed.
icon: file-arrow-up
---

# Create a lab and upload a public file

The default path, and the one to run first. Five steps: get a token, mint the LabNFT, register the lab, upload the file, verify. A public file is stored as-is — no key management, no access conditions.

> **Want the file to be confidential instead?** Steps 1–3 are identical; branch at Step 4 into [Upload an encrypted file](upload-encrypted-file.md).

{% hint style="info" %}
**Before you start:** you need the [two prerequisites](README.md#prerequisites) — a `mol_` consumer credential and a funded Base Sepolia wallet — plus the [shared setup block](shared-setup.md), which defines the config constants and the `graphql()` / `assertOk()` helpers every snippet below uses. The [complete script](#complete-script) at the end of this page carries all of it inline and runs standalone. Unfamiliar with a term used here? See the [Glossary](../../references/glossary.md).
{% endhint %}

## Step 1: Get a service token

Prove control of the wallet instead of waiting on a manually issued token — the self-service path for agents, bots and CI/CD. Fetch a sign-in message, sign it as a plain wallet message (EIP-191 `personal_sign` — **not** typed data), then exchange the signature for a token. Full reference: [Service Tokens](../labs-api/service-tokens.md#obtaining-a-token).

The message carries a server-issued single-use nonce and is valid for **10 minutes**, so these two calls belong together: fetch, sign, redeem. Neither the message nor the signature can be cached or reused.

```javascript
import { createPublicClient, createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const SERVICE_NAME = "tutorial-agent";

const account = privateKeyToAccount(WALLET_PRIVATE_KEY);
// publicClient: read-only RPC calls (readContract, waitForTransactionReceipt).
const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
// walletClient: everything that needs the private key — signing and sending.
const walletClient = createWalletClient({ account, chain: CHAIN, transport: http() });

// Fetch this immediately before signing: the message embeds a single-use
// nonce that expires 10 minutes after issuance, and requesting a new one
// invalidates any previous message for this wallet + service.
const signInMessage = await graphql(
  `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
    getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) {
      message
      expiresAt
    }
  }`,
  { walletAddress: account.address, serviceName: SERVICE_NAME },
);

// Sign the message VERBATIM — the backend recomposes the same string from the
// stored nonce and verifies it, so re-wording, re-formatting or rebuilding it
// client-side breaks verification.
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
| `UNAUTHENTICATED`, `reason: INVALID_SIGNATURE` | The signed bytes are not the message the backend recomposes — altered text, or a message superseded by a later `getServiceSignInMessage` call | Sign the most recent `message` byte-for-byte. Use `personal_sign` / viem's `signMessage`, not `signTypedData` |
| `UNAUTHENTICATED`, `reason: NONCE_NOT_FOUND` | No nonce on file — never requested for this wallet + service, or already redeemed by an earlier token | Re-run the query and sign the new `message`. Retrying the same signature never works |
| `UNAUTHENTICATED`, `reason: NONCE_EXPIRED` | More than 10 minutes passed between fetching the message and redeeming it | Re-run the query and sign the new `message` |
| `UNAUTHENTICATED`, `reason: WALLET_MISMATCH` | `walletAddress` isn't the signer | Pass the same address that signed |
| `VALIDATION_FAILED` | Bad `expiresIn` | Format is `<int><unit>`, unit one of `s m h d w M y`; between 1 hour and 2 years |
| HTTP `401` before GraphQL runs | Consumer credential missing or malformed | Check `Authorization` — no `Bearer` prefix. See [Authentication](../authentication.md) |

`expiresIn` defaults to **`180d`** when omitted. The returned token is **wallet-bound, not lab-bound**: it carries this wallet's identity, and authorisation is resolved per request from that wallet's onchain role on whichever lab you name.

## Step 2: Mint the LabNFT

Mint onchain via `OnChainLabFactory.mintAndCreateAccount` and read `oclId` off the `OclIdentityCreated` event. Reuses `account` / `publicClient` / `walletClient` from Step 1 and `FACTORY_ADDRESS` / `LABNFT_ADDRESS` from the config block. Full detail — the fee call and how `oclId` is derived — is on [Lab Management](../labs-api/lab-management.md#mint-the-labnft).

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

## Step 3: Register the lab

Register the Kamu-backed data room for the freshly-minted `oclId`. Full reference: [Create Lab](../labs-api/lab-management.md#create-lab).

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

DID-linking for the new lab starts automatically in the background; [`getDidLinkStatus`](../labs-api/lab-management.md#get-did-link-status) reports its progress. You do not need to wait for it.

## Step 4: Upload the file

Three calls: get a presigned URL, `PUT` the bytes, finalise with metadata. Full reference: [Files](../labs-api/files.md).

{% hint style="warning" %}
**This is the step that most often fails on a lab you have just created.**

Molecule runs onchain and offchain systems side by side, and the offchain side learns about onchain events through an indexer. Keeping the two in step takes a moment, so a call that succeeded does not mean every read has caught up yet.

That is exactly what happens here. `createLab` can succeed before your mint has been indexed, because it falls back to checking ownership onchain. The file mutations have no such fallback — they read the indexed record, and return `NOT_FOUND` until it arrives.

So wrap the first call in [`withIndexerLagRetry`](shared-setup.md). On staging the record usually appears within seconds, but one mint took **over four minutes**, which is why the helper keeps retrying that long instead of giving up after a few seconds.
{% endhint %}

```javascript
import { readFileSync } from "node:fs";
import { basename } from "node:path";

const filePath = "./research-data.csv";
const bytes = readFileSync(filePath);

// 4a. Get a presigned URL. Retried: the mint may not be indexed yet, even
// though createLab already returned success.
const initiateResult = await withIndexerLagRetry(async () => {
  const result = await graphql(
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
  return assertOk(result.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
});
const { uploadToken, uploadUrl, method, headers } = initiateResult;

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

Keep `datasetId` — it is the file's stable identifier for later reads and updates.

**If it fails:**

| Symptom | Cause | Fix |
| ------- | ----- | --- |
| `initiate` → `NOT_FOUND`, "Project 0x… does not exist" | The mint is not indexed yet. `createLab` can succeed before this is true, so a successful Step 3 is no guarantee | Retry with backoff — `withIndexerLagRetry` above. Usually seconds; observed up to ~4 minutes under indexer backlog. Do **not** re-run `createLab`, which returns `CONFLICT` once registered |
| `initiate` → `UNAUTHORIZED` | The wallet behind the token has no write role on this lab | You must be Owner or Contributor. See [Agent as a lab contributor](agent-as-a-lab-contributor.md) |
| `PUT` → `403` | URL expired (~15 min), or headers altered | Re-run `initiate`; send the returned `headers` verbatim |
| `PUT` → `400`/`411` | Body wasn't sent as raw bytes | Send the buffer, not a JSON wrapper. In curl: `--data-binary` |
| `finish` → `VALIDATION_FAILED`, `details.field: "path"` | `path` contains an underscore, or both `path` and `ref` were sent | Underscores are not allowed in `path`; use `path` for a new file **or** `ref` for a new version, never both |
| `finish` → `VALIDATION_FAILED`, `reason: INVALID_TAGS_OR_CATEGORIES` | Unknown tag or category | Valid values come from the public `fileCategoriesAndTags` query |
| `finish` → `VALIDATION_FAILED`, `reason: INVALID_ACCESS_LEVEL` | Bad `accessLevel` | One of `PUBLIC`, `HOLDERS`, `ADMIN` |
| `finish` → `UPSTREAM_UNAVAILABLE`, "Path is occupied" | A file already lives at that `path` — the usual cause is re-running this tutorial against the same lab | **Not retryable despite the code**: retrying sends the identical request and fails identically. Either pick a new `path`, or send `ref` (the previous `datasetId`) instead of `path` to add a version to the existing file |

## Step 5: Verify it worked

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

## Complete script

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

// `details` arrives as an object (thrown queries), a JSON string (in-band), or
// a doubly-encoded JSON string (in-band today) — parse until it is not a string.
function parseDetails(details) {
  let value = details;
  for (let i = 0; i < 3 && typeof value === "string"; i++) {
    try {
      value = JSON.parse(value);
    } catch {
      break;
    }
  }
  return value && typeof value === "object" ? value : {};
}

function assertOk(result, op) {
  if (result.error) {
    const { code, message, requestId } = result.error;
    const { reason } = parseDetails(result.error.details);
    throw new Error(
      `${op} failed: ${code}${reason ? `/${reason}` : ""}: ${message} (requestId ${requestId})`,
    );
  }
  return result;
}

// The mint reaches the API through an event indexer, so the lab's first write
// can return NOT_FOUND for a few seconds after createLab already succeeded.
async function withIndexerLagRetry(
  fn,
  { codes = ["NOT_FOUND"], attempts = 12, baseMs = 2000, capMs = 30000 } = {},
) {
  const laggy = new RegExp(codes.join("|"));
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      if (!laggy.test(String(err)) || i === attempts - 1) throw err;
      const delay = Math.min(baseMs * 2 ** i, capMs); // 2s, 4s, 8s, 16s, then 30s
      console.warn(`indexer not caught up (attempt ${i + 1}/${attempts}); retrying in ${delay / 1000}s`);
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}

async function main() {
  const filePath = process.argv[2];
  if (!filePath) throw new Error("Usage: node create-lab-and-upload-file.js <file-to-upload>");

  const account = privateKeyToAccount(WALLET_PRIVATE_KEY);
  const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
  const walletClient = createWalletClient({ account, chain: CHAIN, transport: http() });

  // ---- Step 1: service token ----
  // Fetch → sign → redeem, back to back: the message holds a single-use nonce
  // that expires 10 minutes after issuance.
  const signInMessage = await graphql(
    `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
      getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message expiresAt }
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
  // Retried: createLab returning success does not mean the mint is indexed yet.
  const { uploadToken, uploadUrl, method, headers } = await withIndexerLagRetry(async () => {
    const result = await graphql(
      `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
        initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
          uploadToken uploadUrl method headers { key value }
          error { code message requestId retryable details }
        }
      }`,
      { oclId, contentType: "application/octet-stream", contentLength: bytes.length },
    );
    return assertOk(result.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
  });

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
node create-lab-and-upload-file.js ./research-data.csv
```

***

## Next

| | |
| --- | --- |
| Make the next file confidential | [Upload an encrypted file](upload-encrypted-file.md) |
| Let an agent write into this lab | [Agent as a lab contributor](agent-as-a-lab-contributor.md) |
| Run it against mainnet | [Running in Production](README.md#running-in-production) |
| Per-operation reference | [Files](../labs-api/files.md) · [Lab Management](../labs-api/lab-management.md) |
