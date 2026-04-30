---
description: >-
  How Onchain Labs protect confidential research data through client-side
  encryption, on-chain access verification, and a condition-gated key-release
  flow.
icon: fingerprint
---

# Data Privacy & Access

### Why Privacy Matters

Scientific research data is often commercially sensitive, personally identifiable, or competitively valuable. Releasing raw experimental results, proprietary compounds, or patient-derived datasets without control can compromise patent applications, regulatory submissions, and competitive advantage. At the same time, the transparency benefits of on-chain science — provenance, reproducibility, collaboration — require that data _exists_ in a verifiable, shared infrastructure.

Onchain Labs resolve this tension by encrypting data before it enters the public infrastructure. The blockchain records _that_ data exists, _who_ uploaded it, and _who_ can access it — but never the data itself. The underlying content is encrypted client-side, stored as ciphertext, and only decrypted inside an authorised client after access conditions have been verified against live on-chain state.

### Onchain-Verified Envelope Encryption

Molecule uses **Onchain-Verified Envelope Encryption** for every confidential file in an Onchain Lab.

- **Client-side encryption.** Files are AES-256-GCM encrypted inside the client (browser or AI agent) before they leave the device, using a fresh per-file Data Encryption Key (DEK).
- **Decentralised storage.** Ciphertext is pinned to IPFS and persisted to Arweave; access conditions and encryption metadata live alongside the file's provenance record on Kamu (ODF) nodes — decentralised by design.
- **On-chain verification.** Every decryption is gated by a live on-chain check: the stored access conditions are re-evaluated against current chain state (`AccessResolver`, `IPNFT.canRead`, token balances) before the DEK is released. There is no cached permission list.
- **Evolving key custody.** The DEK is wrapped by a protocol-operated key custodian today. Custody moves to a BLS threshold operator network (roadmap) without changes to clients, stored metadata, or the on-chain interface.

Files marked as Public skip encryption entirely. The researcher explicitly chooses to make this data openly accessible. Public files still benefit from content addressing, versioning, and provenance tracking, but they carry no confidentiality guarantees by design.

### Upload Flow

```
1. Client → AppSync: initiateCreateOrUpdateFileV2(ipnftUid, contentType,
                                                  contentLength, encryption: true)
2. Backend: authenticate caller (Privy JWT or service token + role check)
3. Backend: issue a fresh per-file DEK →
            returns { plaintextDEK (one-shot), wrappedDEK, encryptionSystem }
4. Backend: zero its copy of the plaintextDEK after the response is built
5. Client: AES-256-GCM encrypt(file, plaintextDEK) via SubtleCrypto
6. Client: PUT ciphertext to presigned S3 URL
7. Client: build accessControlConditions (createAccessCondition helper)
8. Client → AppSync: finishCreateOrUpdateFileV2(ipnftUid, uploadToken,
                     encryptionMetadata: { encryptionSystem, wrappedDEK,
                                           iv, contentHash, accessControlConditions,
                                           encryptedBy, encryptedAt })
9. Client: wipe plaintextDEK from memory
```

The client only opts **in** to encryption (`encryption: true`). The backend decides which encryption system to use and returns it in `encryptionSystem` — clients must echo this value verbatim, never hardcode it. This keeps the roadmap upgrade to BLS threshold key custody transparent to existing integrations.

### Access Conditions

Who may decrypt a file is determined by on-chain conditions, not by a centralised permission list. When a file is uploaded, the Client SDK attaches an `accessControlConditions` array to the encryption metadata, stored on Kamu (ODF) alongside the file's provenance record. Conditions are **stored but not evaluated** at encrypt time — they're evaluated at decrypt time against live chain state.

Conditions resolve through the [`AccessResolver`](../../references/contracts/accessresolver.md) contract, which exposes three principal predicates:

- **Public** — no condition; anyone can decrypt.
- **Token-Holder** — `isAuthorizedSignerForIpnft(signer, ipnftId)` / `isAuthorizedSignerForTba(signer, account)`: passes for IP-NFT holders and any authorized signer resolved recursively through Safe multisigs, Ownable contracts, and ERC-6551 TBAs.
- **Role-gated** — `hasRole(oclId, signer, ROLE_VIEWER | ROLE_CONTRIBUTOR)`: passes for accounts with an active (non-expired) role grant for the lab. See [Roles & Permissions](../roles-and-permissions.md) for the full role model and grant lifecycle.

Conditions compose via boolean operators — a Lab can, for example, require both Contributor role AND a license NFT before granting access. Future releases will introduce additional composable conditions: credential-gating by minimum IPT holdings, access-list gating by specific wallet addresses, payment-gated unlocks, license-gated access via time-bound license NFTs (ERC-4907), and time-locked conditions that auto-release data at a specified date or block.

#### Condition Shape

Each entry in `accessControlConditions` is one of three TypeScript shapes — `EvmContractCondition` for arbitrary `view` calls, `EvmBasicCondition` for standard ERC reads, and `BooleanCondition` as a separator between predicates:

```ts
interface EvmContractCondition {
  conditionType: "evmContract";
  contractAddress: string;
  chain: string;
  functionName: string;
  functionParams: string[];
  functionAbi: {
    name: string;
    inputs:  Array<{ name: string; type: string; internalType?: string }>;
    outputs: Array<{ name: string; type: string; internalType?: string }>;
    stateMutability: string;
    type: string;
  };
  returnValueTest: { key: string; comparator: string; value: string };
}

interface BooleanCondition {
  operator: "and" | "or";
}
```

The placeholder `:userAddress` inside `functionParams` is substituted with the authenticated caller's wallet at evaluate time. For boolean predicates (`hasRole`, `isAuthorizedSigner*`) the `returnValueTest` is the literal `{ key: "", comparator: "=", value: "true" }`. The full array is JSON-stringified into [`encryptionMetadata.accessControlConditions`](../../api-reference/labs-api.md) — typed as `String!` in the GraphQL schema and parsed back into an array on the backend.

#### Worked Example: Encrypt for Owner OR Contributor OR Viewer

To encrypt a file so the LabNFT owner, any active Contributor, and any active Viewer can all decrypt it, target the `AccessResolver` deployment on the chain whose RPC the backend evaluator uses. Substitute `<accessresolver-address>` below with the right deployment for that chain — see the [deployments table](../../references/contracts/accessresolver.md) — and use `"chain": "base"` (canonical), `"ethereum"`, or `"sepolia"` to match.

`hasRole` already collapses the role hierarchy on the canonical chain — the LabNFT owner passes the admin path inside the contract, a Contributor passes because `ROLE_CONTRIBUTOR ≥ ROLE_VIEWER`, and a Viewer passes directly. So when conditions are evaluated against Base a **single condition** is enough:

```json
[
  {
    "conditionType": "evmContract",
    "contractAddress": "<accessresolver-address>",
    "chain": "base",
    "functionName": "hasRole",
    "functionParams": [
      "0x0101<20hex-tokenId><40hex-tba>",
      ":userAddress",
      "1"
    ],
    "functionAbi": {
      "name": "hasRole",
      "inputs": [
        { "name": "oclId",   "type": "bytes32" },
        { "name": "account", "type": "address" },
        { "name": "role",    "type": "uint8"   }
      ],
      "outputs": [{ "name": "", "type": "bool" }],
      "stateMutability": "view",
      "type": "function"
    },
    "returnValueTest": { "key": "", "comparator": "=", "value": "true" }
  }
]
```

The first `functionParams` entry is the lab's `oclId` — a packed `bytes32` of `0x01` (version) ‖ `0x01` (EVM namespace) ‖ 10-byte big-endian `tokenId` ‖ 20-byte TBA address. See [Onchain Lab](../onchain-lab.md) for how this identifier is derived. The third entry, `"1"`, is `ROLE_VIEWER` — Contributor and Owner pass the same check thanks to hierarchy.

The **explicit OR-composite form** is recommended as the cross-chain-safe default. The contract's owner-check (`_isLabOwner`) returns `false` off the canonical chain (Mainnet / Sepolia) because the OCL TBA's `owner()` returns `address(0)` there, so the role-only condition above will not cover the LabNFT owner if conditions are ever evaluated against a non-Base RPC. OR'ing in `isAuthorizedSignerForTba` keeps the Owner branch explicit:

```json
[
  {
    "conditionType": "evmContract",
    "contractAddress": "<accessresolver-address>",
    "chain": "base",
    "functionName": "isAuthorizedSignerForTba",
    "functionParams": [":userAddress", "0x<40hex-tba>"],
    "functionAbi": {
      "name": "isAuthorizedSignerForTba",
      "inputs": [
        { "name": "signer",  "type": "address" },
        { "name": "account", "type": "address" }
      ],
      "outputs": [{ "name": "", "type": "bool" }],
      "stateMutability": "view",
      "type": "function"
    },
    "returnValueTest": { "key": "", "comparator": "=", "value": "true" }
  },
  { "operator": "or" },
  {
    "conditionType": "evmContract",
    "contractAddress": "<accessresolver-address>",
    "chain": "base",
    "functionName": "hasRole",
    "functionParams": [
      "0x0101<20hex-tokenId><40hex-tba>",
      ":userAddress",
      "1"
    ],
    "functionAbi": {
      "name": "hasRole",
      "inputs": [
        { "name": "oclId",   "type": "bytes32" },
        { "name": "account", "type": "address" },
        { "name": "role",    "type": "uint8"   }
      ],
      "outputs": [{ "name": "", "type": "bool" }],
      "stateMutability": "view",
      "type": "function"
    },
    "returnValueTest": { "key": "", "comparator": "=", "value": "true" }
  }
]
```

The TBA address (`<40hex-tba>`) is the lower 20 bytes of `oclId`; `tokenId` is 10 bytes big-endian sitting between the version/namespace prefix and the TBA. Substitute the AccessResolver address from the [deployments table](../../references/contracts/accessresolver.md) when targeting a non-Base chain. To restrict access to Contributors-and-up only (excluding Viewers), pass `"2"` (`ROLE_CONTRIBUTOR`) instead of `"1"` for the role parameter.

#### How Conditions Are Evaluated

At decrypt time the backend walks the array left-to-right: each `EvmContractCondition` is dispatched as a viem `readContract` call against the configured RPC, the result is compared to `returnValueTest`, and `BooleanCondition` separators short-circuit the chain (`and` stops at the first false, `or` stops at the first true). Any RPC error fails closed — the DEK is not released. A `hasRole` call whose `oclId` does not match the lab's canonical binding reverts with `InvalidOclId` inside the contract and is treated as "condition not met".

### Decryption Flow

Decryption is **condition-authoritative**: the backend reads the stored conditions from their immutable source (Kamu for data-room files, the IPNFT's on-chain `tokenURI` for agreements), verifies them against live chain state, and only then releases the plaintext DEK. A compromised client cannot substitute weaker conditions.

```
1. Client → AppSync: decryptDataKey(ipnftUid, filePath | tokenUri + agreementUrl)
2. Backend: authenticate caller (Privy JWT or service token)
3. Backend: fetch stored encryptionMetadata
            • filePath → Kamu (ODF): file's accessControlConditions + wrappedDEK
            • tokenUri → IPFS: IPNFT JSON → matching agreement's encryption block
4. Backend: evaluate accessControlConditions against live chain state (EVM RPC)
5. Backend: unwrap the DEK via the protocol key custodian → plaintextDEK
6. Backend: zero plaintextDEK buffer after the response is built
7. Client: download ciphertext from S3 (data room) or IPFS (agreement)
8. Client: AES-256-GCM decrypt(ciphertext, plaintextDEK, iv) via SubtleCrypto
9. Client: wipe plaintextDEK from memory
```

The GraphQL interface:

```graphql
mutation {
  decryptDataKey(
    ipnftUid: "0xcaD8...Fc1_42"
    filePath: "raw/experiment-01.csv"   # OR tokenUri + agreementUrl for IPFS agreements
  ) {
    isSuccess
    plaintextDEK   # base64, client-only, wipe after use
    iv             # base64 AES-GCM IV from stored metadata
    message
    error { message code retryable }
  }
}
```

For IPFS-pinned agreement files (immutable once minted), the client passes `tokenUri` (the IPNFT's on-chain `tokenURI`) and `agreementUrl` — the backend fetches the IPNFT JSON, locates the matching agreement in `properties.agreements[]`, and extracts its encryption block. Because the `tokenURI` is on-chain, conditions cannot be tampered with after minting.

### Agentic Encryption

AI agents encrypt and decrypt lab files through the same GraphQL interface, using a service-token auth path instead of a user Privy JWT:

- **Auth** — The agent authenticates with an `X-Service-Token` JWT. For short-lived access, the [x402 Gateway](../../api-reference/x402-gateway.md) mints a per-request token scoped to one mutation after verifying a USDC payment. For long-lived agents, the Molecule team provisions a service token tied to a wallet and an `allowedMutations` list.
- **Role grant** — The Lab owner grants the agent's wallet a Contributor (or Viewer) role via `AccessResolver.grantRole` with `isAgent = true` and a bounded `expiry`. The `isAgent` flag is surfaced in the team-members UI so agent session keys are clearly distinguished from human collaborators.
- **Encrypt** — The agent calls `initiateCreateOrUpdateFileV2(encryption: true)`, receives a plaintext DEK, encrypts the file locally (Node.js `crypto` / Web Crypto), uploads the ciphertext, then calls `finishCreateOrUpdateFileV2` with the encryption metadata.
- **Decrypt** — The agent calls `decryptDataKey(ipnftUid, filePath)`. The backend evaluates the stored conditions against live chain state; a valid Viewer/Contributor grant satisfies the `hasRole` predicate. The backend returns the plaintext DEK over TLS; the agent decrypts locally.
- **Expiry** — When the role grant expires (`block.timestamp >= expiry`), `hasRole` returns `false` and `decryptDataKey` starts failing with a conditions-not-met error. The agent must request a fresh grant — typically from an owner-controlled orchestrator — before it can continue.

See the [Developers / AI Agents guide](../../user-guides/developers-ai-agents.md) for end-to-end agent integration patterns and the [MCP Tools reference](../../references/mcp-tools.md) for the read-side agent toolset.

### Privacy Summary

The net result of this architecture is that no single party has unilateral access to confidential research data. The file content is encrypted before it leaves the client, transmitted as ciphertext, stored as ciphertext across IPFS, Arweave, and S3, and only ever decrypted inside an authorised client after access conditions have been re-verified against live on-chain state. Every action against the data — uploads, version changes, access events — is recorded with the author's decentralised identifier, creating a tamper-evident provenance trail.

<table><thead><tr><th width="177.015625">Layer</th><th>Protection</th><th>Mechanism</th></tr></thead><tbody><tr><td>At rest</td><td>File content encrypted before leaving client</td><td>Client-side AES-256-GCM with a per-file wrapped DEK</td></tr><tr><td>In transit</td><td>All communications over HTTPS; payload is ciphertext</td><td>TLS + pre-encryption</td></tr><tr><td>Key storage</td><td>Plaintext DEK is never persisted; the wrapped DEK is useless without the custodian</td><td>Protocol-operated key custodian today; BLS threshold operator network on roadmap</td></tr><tr><td>Access control</td><td>Decryption gated by a live on-chain verification of stored conditions</td><td><code>AccessResolver</code> (<code>hasRole</code>, <code>isAuthorizedSigner*</code>), <code>IPNFT.canRead</code></td></tr><tr><td>During decryption</td><td>Plaintext DEK only exists inside the authorised client for the session</td><td>Client-side key assembly and decryption; backend zeroes its copy</td></tr><tr><td>Provenance</td><td>Every file action tracked with author's DID</td><td>Kamu (ODF) version records with <code>did:ethr:{wallet_address}</code></td></tr><tr><td>Permanence</td><td>Encrypted content persists even if file record is removed</td><td>IPFS + Arweave store ciphertext; keys are separate</td></tr></tbody></table>

### Roadmap

Key custody evolves from a single protocol-operated custodian to a **BLS threshold operator network**. In the target design the DEK is split across the operator set using threshold cryptography, so no single party — including Molecule — can unwrap it alone. Clients, stored metadata shape, and the on-chain interface stay the same; the `encryptionSystem` value on new files rolls forward to indicate threshold custody, and the `decryptDataKey` flow continues to work transparently.

### Legacy: Lit Protocol _(deprecated for new files)_

> **Lit Protocol is retained read-only for files encrypted before the migration to Onchain-Verified Envelope Encryption.** New uploads go through the current flow; the backend no longer generates Lit-encrypted files. Existing Lit-encrypted files continue to decrypt through the Lit SDK until they are migrated. Calling `decryptDataKey` on a Lit-encrypted file returns an error directing the caller to the Lit SDK.

The legacy model uses Lit Protocol's threshold-cryptography network: a symmetric key is generated client-side, sharded across Lit's decentralised nodes, and only reassembled on the client when a threshold of nodes independently verifies the same on-chain conditions exposed through `AccessResolver`. Files carry a distinct metadata shape (`dataToEncryptHash`, `litSdkVersion`, `litNetwork`, `templateName`, `contractVersion`) and are discriminated by the absence of the `encryptionSystem` field.

Integrators with Lit-encrypted files in their data rooms should continue using `@moleculexyz/storage`'s Lit flow for those files, and the current flow for everything new. Re-encryption tooling to migrate legacy files is tracked separately.
