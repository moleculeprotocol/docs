---
description: >-
  The whole default Labs flow on one page, no prose — the page to paste into a
  system prompt.
icon: robot
---

# 🤖 For Agents: One-Pager

The complete default flow — self-issue a token, mint a lab, upload a public file, verify — with nothing else on the page. Copy it into a system prompt or a context file. For the same flow with expected responses, failure modes and the encrypted variant, use [the tutorials](README.md).

## Constants

Staging (Base Sepolia) — everything on this page runs against these:

```
GRAPHQL_URL              https://staging.graphql.api.molecule.xyz/graphql
CHAIN                    baseSepolia (84532)
ONCHAIN_LAB_FACTORY      0xd629FE2310b4309a212495F10A47f8436dcEfD90
LABNFT                   0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28
ACCESS_RESOLVER          0x5493F472602C87318EA5Eff753cDD593bf9bF559
ACCESS_CONDITION_CHAIN   "baseSepolia"
LAB_PAGE                 https://testnet.labs.molecule.xyz/projects/<shortname>
```

Production (Base) — swap these in, nothing else changes:

```
GRAPHQL_URL              https://production.graphql.api.molecule.xyz/graphql
CHAIN                    base (8453)
ONCHAIN_LAB_FACTORY      0xECdF4f05384056507485C90aeAb0a83268760D6E
LABNFT                   0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92
ACCESS_RESOLVER          0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B
ACCESS_CONDITION_CHAIN   "base"
LAB_PAGE                 https://labs.molecule.xyz/projects/<shortname>
```

## Headers

```
Content-Type:    application/json
Authorization:   mol_<consumerId>_<secret>      # NEVER prefixed with "Bearer"
X-Service-Token: <JWT>                          # mutations only; omit the header entirely if you have none
```

Public queries take `Authorization` alone. Sending `X-Service-Token` on a public query is unnecessary; sending an empty one is worse than omitting it.

## Error contract

* **Queries throw.** Failure lands in top-level `errors[]`. Branch on `errors[i].errorType`. `errors[i].errorInfo` is `{ requestId, retryable, details }` and `details` is already an object.
* **Mutations return errors in-band.** Every `*Result` has `error: ApiError`. **Success ⇔ `error == null`.** Select `error { code message requestId retryable details }` on every mutation.
* **Parse `details` tolerantly.** It is an object on thrown query errors, a JSON string in-band, and currently a **doubly-encoded** JSON string in-band — one `JSON.parse` there returns a string, and `.reason` on it is silently `undefined`. Loop until it is not a string:

```javascript
function parseDetails(d) {
  let v = d;
  for (let i = 0; i < 3 && typeof v === "string"; i++) { try { v = JSON.parse(v); } catch { break } }
  return v && typeof v === "object" ? v : {};
}
```
* Branch on `code`, never on `message`. Retry only when `retryable` is `true`, with exponential backoff. Quote `requestId` in any bug report.
* Codes: `UNAUTHENTICATED`, `UNAUTHORIZED`, `NOT_FOUND`, `VALIDATION_FAILED`, `CONFLICT`, `FAILED_PRECONDITION`, `COMPLEXITY_LIMIT_EXCEEDED`, `RATE_LIMITED`*, `TIMEOUT`*, `UPSTREAM_UNAVAILABLE`*, `INTERNAL_ERROR`* (`*` = retryable). An unrecognised code: treat as non-retryable, preserve it, surface it.

## Step 1 — Self-issue a service token

```graphql
query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
  getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message expiresAt }
}
```

Sign `message` **verbatim** with the wallet as a plain personal message (EIP-191 `personal_sign`, **not** typed data). `message` embeds a **single-use nonce valid for 10 minutes** — fetch it immediately before signing, never cache it or the signature, never rebuild the string yourself, and fetch a fresh one for every token. Then:

```graphql
mutation GenerateServiceToken($serviceName: String!, $walletAddress: String!, $messageSignature: String!, $expiresIn: String) {
  generateServiceToken(serviceName: $serviceName, walletAddress: $walletAddress, messageSignature: $messageSignature, expiresIn: $expiresIn) {
    token tokenId expiresAt
    error { code message requestId retryable details }
  }
}
```

`token` → `X-Service-Token` on everything after this. `expiresIn` defaults to `180d`; format `<int><unit>` with unit one of `s m h d w M y`; bounds 1 hour to 2 years. The token is **wallet-bound, not lab-bound**: authorisation is resolved per request from that wallet's onchain role on the lab you name.

## Step 2 — Mint the LabNFT (onchain)

```
OnChainLabFactory.mintAndCreateAccount(address to) payable returns (address account, uint256 tokenId)
LabNFT.mintFeeWei() view returns (uint256)                     // send as `value`; reads 0 on both chains today
LabNFT event OclIdentityCreated(address indexed account, bytes32 indexed oclId, uint256 indexed tokenId, bytes32 salt, uint256 canonicalChainId)
```

Read `oclId` off `OclIdentityCreated`. The event fires on **LabNFT**, not the factory — filter receipt logs to the LabNFT address before decoding.

## Step 3 — Register the lab

```graphql
mutation CreateLab($oclId: String!) {
  createLab(input: { oclId: $oclId }) {
    message
    lab { oclId shortname labAccountAddress labNftTokenId }
    error { code message requestId retryable details }
  }
}
```

`CONFLICT` / `reason: PROJECT_CONFLICT` means the lab is already registered — treat as success and continue.

## Step 4 — Upload a public file (3 calls)

```graphql
mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
  initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
    uploadToken uploadUrl method headers { key value } uploadUrlExpiry
    error { code message requestId retryable details }
  }
}
```

`PUT` the raw bytes to `uploadUrl` with **exactly** the returned `headers` (key/value pairs) and the returned `method`. Presigned URLs expire in ~15 minutes.

```graphql
mutation Finish($oclId: String!, $uploadToken: String!, $path: String!, $accessLevel: String!, $changeBy: String!,
                $description: String, $tags: [String!], $categories: [String!], $contentText: String) {
  finishCreateOrUpdateFile(oclId: $oclId, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel,
                           changeBy: $changeBy, description: $description, tags: $tags,
                           categories: $categories, contentText: $contentText) {
    datasetId contentHash version
    error { code message requestId retryable details }
  }
}
```

* `accessLevel`: `"PUBLIC"` | `"HOLDERS"` | `"ADMIN"`.
* `changeBy`: the caller's wallet address.
* `path` for a **new** file (no underscores), `ref` for a **new version** of an existing one. Use one or the other, never both.
* Valid `tags` / `categories` come from the public `fileCategoriesAndTags` query.

## Step 5 — Verify

```graphql
query Verify($oclId: String!) {
  labWithDataRoomAndFiles(oclId: $oclId) {
    oclId shortname name
    dataRoom { id files { path contentType accessLevel version createdBy } }
  }
}
```

Public query, `Authorization` only. Your `path` is in `dataRoom.files`. A `null` result means the lab is not registered — this query is nullable and does not throw for a missing lab.

## Encrypted files, in four lines

1. `generateDataEncryptionKey` → `{ plaintextDEK, encryptedDek, encryptionSystem }`.
2. AES-256-GCM the bytes locally with `plaintextDEK` and a fresh 12-byte IV; discard the plaintext key.
3. `finishCreateOrUpdateFile` with `accessLevel: "HOLDERS"` (or `"ADMIN"`) and `encryptionMetadata: { encryptionSystem, encryptedDek, iv, contentHash, accessControlConditions, encryptedBy, encryptedAt }` — echo `encryptionSystem` verbatim, never hardcode it.
4. To read it back: `decryptDataKey(oclId:, filePath:)` → `{ plaintextDEK, iv }` after the backend re-evaluates the file's onchain access conditions against your wallet.

Full recipe including `accessControlConditions`: [Upload an encrypted file](upload-encrypted-file.md).

## Rules that break runs when ignored

1. No `Bearer` in front of a `mol_` credential.
2. Sign the sign-in message **verbatim**; any reformatting fails verification. It is single-use and expires after 10 minutes, so a cached message or signature returns `UNAUTHENTICATED` — `NONCE_NOT_FOUND` once redeemed, `NONCE_EXPIRED` past the window, `INVALID_SIGNATURE` if a later fetch superseded it. The fix is always a fresh `getServiceSignInMessage`, never a retry of the same signature.
3. Success on a mutation is `error == null` — never a truthy payload field, and never a `message` string.
4. Read `error.details` with the tolerant `parseDetails` above — never a bare `JSON.parse`. In-band it is a JSON string (currently doubly encoded); on thrown query errors it is already an object.
5. Filter mint receipt logs to the **LabNFT** address before decoding `OclIdentityCreated`.
6. Send the presigned `PUT` with the returned headers unchanged, and the raw bytes as the body.
7. Writing into a lab you do not own needs a **Contributor** role on it — see [Agent as a lab contributor](agent-as-a-lab-contributor.md). After a role grant, an indexer lag of a few seconds can still return `UNAUTHORIZED`; retry with backoff.
8. **A successful `createLab` does not mean the lab is writable yet.** Step 4's first call can return `NOT_FOUND` ("Project 0x… does not exist") for a few seconds, because `createLab` falls back to an onchain ownership check while the file mutations read the indexed record. Retry `NOT_FOUND` with backoff on the first write after a mint; do not re-run `createLab`, which then returns `CONFLICT`.
9. **Three addresses, not interchangeable**: your own wallet (`walletAddress`, `changeBy`), the human owner's (`x-wallet-address`, their path only), and the Lab's OCL account (`labAccountAddress`, and the `account` argument in access conditions). Putting an owner's address where `labAccountAddress` belongs uploads fine and then locks everyone out of the file, with no error saying why. `oclId` is none of them — it is a lab id whose trailing 40 hex chars happen to be the OCL account address. See [the three wallets](../authentication.md#the-three-wallets-side-by-side).
10. Production has introspection off and a depth limit of 10. Generate types against staging.

## Related

* [The three wallets](../authentication.md#the-three-wallets-side-by-side) — owner vs agent vs OCL account, and which field each address goes in
* [Getting Started](README.md) — how to interact with our products, prerequisites, costs
* [Tutorials](README.md) — the same flow with responses and failure handling
* [Labs API](../labs-api/README.md) — full operation reference
* [Molecule Skill](../../ai-tooling/molecule-skill.md) — the same workflow as MCP tool calls
* [x402 Gateway](../x402-gateway.md) — pay per call, no service token
