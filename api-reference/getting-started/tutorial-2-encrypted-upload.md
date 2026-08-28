---
description: >-
  Encrypt a file locally with AES-256-GCM, gate decryption on live onchain
  roles, and verify with a full decrypt round trip.
icon: lock
---

# Tutorial 2: Upload an encrypted file

Same lab, same three-call upload — but the bytes are AES-256-GCM encrypted locally before they leave your machine, and decryption is gated on live onchain state. Encryption is a first-class flow, not an appendix: reach for it whenever the file is confidential and access should follow the lab's roles.

**Steps 1–3 are identical to [Tutorial 1](tutorial-1-public-upload.md)** — get a service token, mint, register. Pick up here with `oclId`, `labAccountAddress`, `account` and `serviceToken` already in hand (or with any lab you already hold a role on).

Conceptually: the backend hands you a one-shot data encryption key (DEK) in two forms — plaintext, and wrapped by the key custodian. You encrypt with the plaintext copy, throw it away, and store the wrapped copy in the file's metadata alongside the conditions under which the custodian may unwrap it again. Full model: [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md).

{% hint style="info" %}
**Before you start:** you need the [two prerequisites](README.md#prerequisites) — a `mol_` consumer credential and a funded Base Sepolia wallet — plus the [shared setup block](README.md#shared-setup), which defines the config constants and the `graphql()` / `assertOk()` helpers every snippet below uses. The [complete script](#complete-script) at the end of this page carries all of it inline and runs standalone.
{% endhint %}

## Step 4a: Get a DEK

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

## Step 4b: Encrypt locally

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

## Step 4c: Write the access conditions

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

## Step 4d: Upload the ciphertext

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

## Step 5: Verify by decrypting it

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

## Complete script

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

## Next

| | |
| --- | --- |
| Let an agent decrypt and contribute too | [Tutorial 3 — Give your agent access](tutorial-3-agent-access.md) |
| Publish an update attaching the file | [Tutorial 4 — Announce the dataset](tutorial-4-announce.md) |
| Run it against mainnet | [Running in Production](README.md#running-in-production) |
| How conditions are evaluated, in depth | [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md) |
