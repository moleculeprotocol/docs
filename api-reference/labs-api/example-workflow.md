# Example Workflow: Authenticate, Mint, Create, Sign, Encrypt, Upload

A complete, runnable walkthrough that takes a wallet with **no prior credentials and no onchain lab** all the way to an encrypted file living in its dataroom: prove control of the wallet to mint a service token, mint the LabNFT, register the dataroom, sign the assignment agreement, then encrypt and upload a file. Each step below links back to its full reference; the [Complete Script](#complete-script) at the end wires all five together.

This is the workflow an autonomous agent needs to run end-to-end without any browser-based user interaction or manually provisioned Service Token — the only thing it needs ahead of time is a consumer credential and a funded wallet. It's written against **staging** (Base Sepolia, testnet ETH) end to end; see [Running in Production](#running-in-production) at the bottom for the values to swap.

## Prerequisites

* A funded EOA on **Base Sepolia** — get testnet ETH from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets)
* A **consumer credential** — see [Authentication](../authentication.md). No pre-issued Service Token needed; the workflow mints its own in Step 1.
* `viem` and `node-fetch` (`npm install viem node-fetch`)

Every environment-specific value used below — the GraphQL endpoint, contract addresses, and the viem chain — lives in this one block. Swapping to production later is a matter of replacing this block with the table in [Running in Production](#running-in-production).

```javascript
import { baseSepolia } from "viem/chains"; // production: `base`

// ---- Staging (Base Sepolia) config — see "Running in Production" to swap ----
const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const CHAIN = baseSepolia;
const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90"; // OnChainLabFactory
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28"; // LabNFT (proxy)
const ACCESS_RESOLVER_ADDRESS = "0x5493F472602C87318EA5Eff753cDD593bf9bF559"; // AccessResolver
const ACCESS_CONDITION_CHAIN = "baseSepolia"; // the `chain` string inside accessControlConditions

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
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}
```

## Step 1: Get a Service Token

Prove control of the wallet instead of waiting on a manually issued token — the self-service path for agents, bots, and CI/CD. Fetch the deterministic sign-in message, sign it as a plain wallet message (EIP-191 `personal_sign` — not typed data), then exchange the signature for a token. Full reference: [Service Tokens — Obtaining Tokens](service-tokens.md#obtaining-tokens).

```javascript
import { createPublicClient, createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const SERVICE_NAME = "example-workflow-agent";

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
      isSuccess
      message
    }
  }`,
  { serviceName: SERVICE_NAME, walletAddress: account.address, messageSignature },
);
if (!tokenResult.generateServiceToken.isSuccess) {
  throw new Error(tokenResult.generateServiceToken.message ?? "generateServiceToken failed");
}
serviceToken = tokenResult.generateServiceToken.token;
```

`generateServiceToken`'s result has no `error` field — check `isSuccess` and fall back to `message` on failure. The returned `token` authorizes this wallet's onchain-resolved role for whatever lab it acts on; it isn't scoped to a single `oclId` up front.

## Step 2: Mint the LabNFT

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
```

## Step 3: Create the Lab

Register the Kamu-backed dataroom for the freshly-minted `oclId`. Full reference: [Create Lab](lab-management.md#create-lab).

```javascript
const createLabResult = await graphql(
  `mutation CreateLab($oclId: String!) {
    createLab(input: { oclId: $oclId }) {
      isSuccess
      message
      error { message code retryable }
      lab { oclId shortname labAccountAddress labNftTokenId }
    }
  }`,
  { oclId },
);
if (!createLabResult.createLab.isSuccess) {
  throw new Error(createLabResult.createLab.error?.message ?? "createLab failed");
}
const { labAccountAddress } = createLabResult.createLab.lab;
```

## Step 4: Sign the Assignment Agreement

Fetch the populated agreement, sign the `LegalAgreementAcceptance` EIP-712 payload, then submit it. Full schema and a self-test vector: [Legal Agreements — EIP-712 Envelope](legal-agreements.md#eip-712-envelope).

```javascript
const template = await graphql(
  `query Template($oclId: String!, $walletAddress: String!) {
    legalAgreementTemplate(
      oclId: $oclId
      type: ASSIGNMENT_AGREEMENT
      walletAddress: $walletAddress
    ) {
      isSuccess
      contentHash
      templateVersion
      issuedAt
      error { message code retryable }
    }
  }`,
  { oclId, walletAddress: account.address },
);
const { contentHash, templateVersion, issuedAt } = template.legalAgreementTemplate;

const signature = await walletClient.signTypedData({
  domain: {
    name: "MoleculeOcl",
    version: "1",
    chainId: CHAIN.id,
    verifyingContract: LABNFT_ADDRESS.toLowerCase(),
  },
  types: {
    LegalAgreementAcceptance: [
      { name: "oclId", type: "bytes32" },
      { name: "agreementType", type: "string" },
      { name: "contentHash", type: "bytes32" },
      { name: "templateVersion", type: "string" },
      { name: "signer", type: "address" },
      { name: "issuedAt", type: "uint64" },
    ],
  },
  primaryType: "LegalAgreementAcceptance",
  message: {
    oclId,
    agreementType: "assignment-agreement", // registry slug, NOT the "ASSIGNMENT_AGREEMENT" enum value
    contentHash,
    templateVersion,
    signer: account.address.toLowerCase(),
    issuedAt: BigInt(issuedAt),
  },
});

const signResult = await graphql(
  `mutation Sign($input: SignLegalAgreementInput!) {
    signLegalAgreement(input: $input) {
      isSuccess
      path
      message
      error { message code retryable }
    }
  }`,
  {
    input: {
      oclId,
      type: "ASSIGNMENT_AGREEMENT",
      walletAddress: account.address,
      signature,
      issuedAt,
    },
  },
);
if (!signResult.signLegalAgreement.isSuccess) {
  throw new Error(signResult.signLegalAgreement.error?.message ?? "signLegalAgreement failed");
}
```

## Step 5: Encrypt and Upload a File

Request a data encryption key, AES-256-GCM encrypt the file locally via Web Crypto, then run the standard three-step upload with `encryptionMetadata` attached. Uses `ACCESS_RESOLVER_ADDRESS` / `ACCESS_CONDITION_CHAIN` from the config block. Full reference: [Files — Advanced: Encrypted File Upload](files.md#advanced-encrypted-file-upload) and [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md).

```javascript
import { webcrypto, randomBytes, createHash } from "node:crypto";
import { readFileSync } from "node:fs";
import { basename } from "node:path";

const filePath = "./research-data.csv";
const plaintext = readFileSync(filePath);

// 5a. Get a DEK
const dekResult = await graphql(`
  mutation {
    generateDataEncryptionKey {
      isSuccess
      plaintextDEK
      encryptedDek
      encryptionSystem
      error { message code retryable }
    }
  }
`);
const { plaintextDEK, encryptedDek, encryptionSystem } = dekResult.generateDataEncryptionKey;

// 5b. Encrypt locally (Web Crypto SubtleCrypto), then wipe the plaintext key
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

// 5c. Gate decryption to the lab owner (LabNFT owner / authorized TBA signer).
// See the Data Privacy & Access worked example for OR-composing in
// Contributor/Viewer roles too.
const accessControlConditions = JSON.stringify([
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

// 5d. Standard three-step upload, ciphertext in place of the raw file
const initiateResult = await graphql(
  `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
    initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
      uploadToken
      uploadUrl
      method
      headers { key value }
      isSuccess
      error { message code retryable }
    }
  }`,
  { oclId, contentType: "application/octet-stream", contentLength: ciphertext.length },
);
const { uploadToken, uploadUrl, headers } = initiateResult.initiateCreateOrUpdateFile;

const uploadHeaders = {};
headers.forEach((h) => (uploadHeaders[h.key] = h.value));
const putResponse = await fetch(uploadUrl, { method: "PUT", headers: uploadHeaders, body: ciphertext });
if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.statusText}`);

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
      isSuccess
      message
      error { message code retryable }
    }
  }`,
  {
    oclId,
    uploadToken,
    path: basename(filePath),
    accessLevel: "HOLDERS", // DataRoomAccessLevel: PUBLIC | HOLDERS | ADMIN — encrypted files use HOLDERS or ADMIN
    changeBy: account.address,
    encryptionMetadata: {
      encryptionSystem, // echo verbatim — never hardcode
      encryptedDek,
      iv: iv.toString("base64"),
      contentHash: contentHashHex,
      accessControlConditions,
      encryptedBy: account.address.toLowerCase(),
      encryptedAt: new Date().toISOString(),
    },
  },
);
if (!finishResult.finishCreateOrUpdateFile.isSuccess) {
  throw new Error(finishResult.finishCreateOrUpdateFile.error?.message ?? "finishCreateOrUpdateFile failed");
}
console.log("Uploaded. datasetId:", finishResult.finishCreateOrUpdateFile.datasetId);
```

***

## Complete Script

All five steps combined into one file, against **staging**. Run with `WALLET_PRIVATE_KEY` and `CONSUMER_CREDENTIAL` set, and a file at the path passed on the command line — no pre-issued Service Token needed. See [Running in Production](#running-in-production) below to point this at mainnet instead.

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

// ---- Staging (Base Sepolia) config — see "Running in Production" to swap ----
const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const CHAIN = baseSepolia;
const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90"; // OnChainLabFactory
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28"; // LabNFT (proxy)
const ACCESS_RESOLVER_ADDRESS = "0x5493F472602C87318EA5Eff753cDD593bf9bF559"; // AccessResolver
const ACCESS_CONDITION_CHAIN = "baseSepolia"; // the `chain` string inside accessControlConditions
const SERVICE_NAME = "example-workflow-agent";

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL; // mol_<id>_<secret> — no "Bearer" prefix
const WALLET_PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY;

// Set once Step 1 exchanges a wallet signature for a token.
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
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}

async function main() {
  const filePath = process.argv[2];
  if (!filePath) throw new Error("Usage: node workflow.js <file-to-upload>");

  const account = privateKeyToAccount(WALLET_PRIVATE_KEY);
  // publicClient: read-only RPC calls. walletClient: signing and sending.
  const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
  const walletClient = createWalletClient({ account, chain: CHAIN, transport: http() });

  // ---- Step 1: Get a Service Token ----
  const signInMessage = await graphql(
    `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
      getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) {
        message
      }
    }`,
    { walletAddress: account.address, serviceName: SERVICE_NAME },
  );
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
        isSuccess
        message
      }
    }`,
    { serviceName: SERVICE_NAME, walletAddress: account.address, messageSignature },
  );
  if (!tokenResult.generateServiceToken.isSuccess) {
    throw new Error(tokenResult.generateServiceToken.message ?? "generateServiceToken failed");
  }
  serviceToken = tokenResult.generateServiceToken.token;
  console.log("1/5 Got service token");

  // ---- Step 2: Mint the LabNFT ----
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
  console.log("2/5 Minted LabNFT — oclId:", oclId);

  // ---- Step 3: Create the Lab ----
  const createLabResult = await graphql(
    `mutation CreateLab($oclId: String!) {
      createLab(input: { oclId: $oclId }) {
        isSuccess
        error { message code retryable }
        lab { labAccountAddress }
      }
    }`,
    { oclId },
  );
  if (!createLabResult.createLab.isSuccess) {
    throw new Error(createLabResult.createLab.error?.message ?? "createLab failed");
  }
  const { labAccountAddress } = createLabResult.createLab.lab;
  console.log("3/5 Lab created — TBA:", labAccountAddress);

  // ---- Step 4: Sign the Assignment Agreement ----
  const template = await graphql(
    `query Template($oclId: String!, $walletAddress: String!) {
      legalAgreementTemplate(oclId: $oclId, type: ASSIGNMENT_AGREEMENT, walletAddress: $walletAddress) {
        isSuccess
        contentHash
        templateVersion
        issuedAt
        error { message code retryable }
      }
    }`,
    { oclId, walletAddress: account.address },
  );
  const { contentHash, templateVersion, issuedAt } = template.legalAgreementTemplate;

  const signature = await walletClient.signTypedData({
    domain: {
      name: "MoleculeOcl",
      version: "1",
      chainId: CHAIN.id,
      verifyingContract: LABNFT_ADDRESS.toLowerCase(),
    },
    types: {
      LegalAgreementAcceptance: [
        { name: "oclId", type: "bytes32" },
        { name: "agreementType", type: "string" },
        { name: "contentHash", type: "bytes32" },
        { name: "templateVersion", type: "string" },
        { name: "signer", type: "address" },
        { name: "issuedAt", type: "uint64" },
      ],
    },
    primaryType: "LegalAgreementAcceptance",
    message: {
      oclId,
      agreementType: "assignment-agreement",
      contentHash,
      templateVersion,
      signer: account.address.toLowerCase(),
      issuedAt: BigInt(issuedAt),
    },
  });

  const signResult = await graphql(
    `mutation Sign($input: SignLegalAgreementInput!) {
      signLegalAgreement(input: $input) {
        isSuccess
        path
        error { message code retryable }
      }
    }`,
    { input: { oclId, type: "ASSIGNMENT_AGREEMENT", walletAddress: account.address, signature, issuedAt } },
  );
  if (!signResult.signLegalAgreement.isSuccess) {
    throw new Error(signResult.signLegalAgreement.error?.message ?? "signLegalAgreement failed");
  }
  console.log("4/5 Agreement signed —", signResult.signLegalAgreement.path);

  // ---- Step 5: Encrypt and Upload a File ----
  const plaintext = readFileSync(filePath);

  const dekResult = await graphql(`
    mutation {
      generateDataEncryptionKey {
        isSuccess
        plaintextDEK
        encryptedDek
        encryptionSystem
        error { message code retryable }
      }
    }
  `);
  const { plaintextDEK, encryptedDek, encryptionSystem } = dekResult.generateDataEncryptionKey;

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

  const accessControlConditions = JSON.stringify([
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

  const initiateResult = await graphql(
    `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
      initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
        uploadToken
        uploadUrl
        headers { key value }
        isSuccess
        error { message code retryable }
      }
    }`,
    { oclId, contentType: "application/octet-stream", contentLength: ciphertext.length },
  );
  const { uploadToken, uploadUrl, headers } = initiateResult.initiateCreateOrUpdateFile;

  const uploadHeaders = {};
  headers.forEach((h) => (uploadHeaders[h.key] = h.value));
  const putResponse = await fetch(uploadUrl, { method: "PUT", headers: uploadHeaders, body: ciphertext });
  if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.statusText}`);

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
        isSuccess
        error { message code retryable }
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
        accessControlConditions,
        encryptedBy: account.address.toLowerCase(),
        encryptedAt: new Date().toISOString(),
      },
    },
  );
  if (!finishResult.finishCreateOrUpdateFile.isSuccess) {
    throw new Error(finishResult.finishCreateOrUpdateFile.error?.message ?? "finishCreateOrUpdateFile failed");
  }
  console.log("5/5 File uploaded — datasetId:", finishResult.finishCreateOrUpdateFile.datasetId);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**Usage:**

```bash
WALLET_PRIVATE_KEY="0x..." CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret" node workflow.js ./research-data.csv
```

***

## Running in Production

Everything above runs against staging (Base Sepolia, testnet ETH). To run the same script against production, replace the six values in the config block — nothing else in the script changes, since every step reads from these constants:

| Constant                    | Staging (this walkthrough)                    | Production                                    |
| ---------------------------- | ---------------------------------------------- | ----------------------------------------------- |
| `GRAPHQL_URL`                 | `https://staging.graphql.api.molecule.xyz/graphql` | `https://production.graphql.api.molecule.xyz/graphql` |
| `CHAIN` (viem import)         | `baseSepolia` from `viem/chains`               | `base` from `viem/chains`                       |
| `FACTORY_ADDRESS`             | `0xd629FE2310b4309a212495F10A47f8436dcEfD90`   | `0xECdF4f05384056507485C90aeAb0a83268760D6E`     |
| `LABNFT_ADDRESS`              | `0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28`   | `0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92`     |
| `ACCESS_RESOLVER_ADDRESS`     | `0x5493F472602C87318EA5Eff753cDD593bf9bF559`   | `0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B`     |
| `ACCESS_CONDITION_CHAIN`      | `"baseSepolia"`                                | `"base"`                                        |

```javascript
import { base } from "viem/chains"; // instead of baseSepolia

const GRAPHQL_URL = "https://production.graphql.api.molecule.xyz/graphql";
const CHAIN = base;
const FACTORY_ADDRESS = "0xECdF4f05384056507485C90aeAb0a83268760D6E";
const LABNFT_ADDRESS = "0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92";
const ACCESS_RESOLVER_ADDRESS = "0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B";
const ACCESS_CONDITION_CHAIN = "base";
```

A few things that follow automatically from that swap and don't need separate handling:

* **EIP-712 `chainId`** in Step 4 is read as `CHAIN.id` (`8453` for `base`, `84532` for `baseSepolia`) — it tracks `CHAIN` and needs no separate edit. See [Legal Agreements — EIP-712 Envelope](legal-agreements.md#eip-712-envelope) for why this must match the LabNFT's actual deployment chain.
* **Headers and the `graphql()` helper** are identical in both environments — `Authorization` (consumer credential, no `Bearer` prefix) and the self-issued `X-Service-Token` from Step 1 work the same way against both endpoints. See [Authentication](../authentication.md).
* **The `mintFeeWei()` read** in Step 2 already queries the live contract, so it picks up whatever fee production has configured without a code change.

What doesn't follow automatically, and is on you to handle:

* **Real funds.** Minting on `base` spends real ETH from the wallet behind `WALLET_PRIVATE_KEY`, and the assignment agreement you sign is a real one. Test the full flow on staging first.
* **`SERVICE_NAME`** should identify the real integration once you're not just testing — it's echoed into the sign-in message and stored against the issued token.
* Full deployment list, including every other OCL contract on both chains: [Contracts reference](../../references/contracts/README.md).
