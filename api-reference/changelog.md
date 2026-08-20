---
icon: clock-rotate-left
---

# API Changelog & Migration

This page tracks breaking changes, deprecations, and additions across the Molecule APIs. Use it when upgrading an existing integration. Each API's most recent changes are listed first.

---

## Authentication

### `x-api-key` replaced by consumer credentials

All Molecule APIs (Labs, Tokenization, and IPNFT (Deprecated) — they share one GraphQL endpoint) now authenticate with a consumer credential instead of an `x-api-key` header. A consumer credential has the shape `mol_<consumerId>_<secret>` and is sent directly as the `Authorization` header value, with **no `Bearer` prefix**.

```diff
- x-api-key: YOUR_API_KEY
+ Authorization: mol_<consumerId>_<secret>
```

**Migration:** Contact the Molecule team for a consumer credential and send it as `Authorization: mol_<consumerId>_<secret>` instead of `x-api-key` — do not prefix it with `Bearer`, which is reserved for Privy user tokens and will fail authentication. Nothing else changes: `X-Service-Token` for machine-authorized mutations, and `Authorization: Bearer <Privy token>` + `x-wallet-address` for user-authorized mutations, work exactly as before. See [Authentication](authentication.md) for the full header reference.

---

## Labs API

### Lab access policies — permissionless & condition-gated labs

Labs can now open individual data-room capabilities beyond their onchain members. This is **purely additive**: a lab created without a policy behaves exactly as before, including its error responses, and a role grant always wins over a policy, so no existing integration changes.

#### New mutation

| Mutation                | Description                                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `updateLabAccessPolicy` | Set a lab's contribution-access policy — open it up, gate it back down (`preset: GATED`), or configure per-capability rules. Owner-only |

#### New input field

`CreateLabInput` gained an optional `accessPolicy: LabAccessPolicyInput`, which creates the lab already open (or condition-gated). Omitting it keeps the default role-gated lab. Passing it restricts `createLab` to the lab owner.

#### New output field

`Lab.accessPolicy` and `LabRef.accessPolicy` expose the stored policy as `LabAccessPolicy` — public, since policies gate access rather than being secrets. Labs without a stored policy return the synthesized `GATED` default. Render from `capabilities`; `preset` is a provenance hint and is `null` for custom configurations.

#### New enums and inputs

`LabAccessPreset` (`GATED` | `OPEN`), `LabCapability` (`ADD_FILES` | `MODIFY_FILES` | `DELETE_FILES` | `CREATE_ANNOUNCEMENTS` | `DECRYPT_FILES`), `LabPolicyRuleKind` (`ROLES` | `ANYONE` | `CONDITIONS`), `LabAccessPolicyInput`, `LabCapabilityPolicyInput`, and the output types `LabAccessPolicy` / `LabCapabilityPolicy`.

#### What to watch for when integrating

* Writes to an open lab still require authentication — an API Key plus a Privy session or a Service Token. Permissionless is not unauthenticated.
* On a policy-granted write, `changeBy` is pinned to the authenticated caller; a spoofed value is ignored.
* `DECRYPT_FILES` defaults to viewer-or-above (not contributor-or-above like the write capabilities) and is **not** included in the `OPEN` preset — an open lab's encrypted files stay member-readable-only until the owner adds an explicit rule.
* Denials add second-level causes on `details.reason`: `CAPABILITY_DENIED`, `INVALID_ACCESS_POLICY`, `POLICY_CHECK_UNAVAILABLE` (retryable), `LAB_ACCESS_CHECK_FAILED`. No new top-level `error.code` values were introduced.

Full reference: [Access Policies](labs-api/access-policies.md).

---

### Breaking: `generateDataEncryptionKey` now requires `oclId` and `accessControlConditions`

Minting a data encryption key is now lab-scoped and bound to the file's condition array up front, closing a gap where an encrypted file's DEK could be re-published under different access conditions after the fact.

* `generateDataEncryptionKey` gained two **required** arguments: `oclId: String!` and `accessControlConditions: String!`. Calls with no arguments now fail `VALIDATION_FAILED`.
* The returned `encryptedDek` carries a `v1:` bound-marker prefix ahead of the base64 ciphertext — pass it through to `finishCreateOrUpdateFile` **verbatim**. A new `dekContextVersion` field (`"v1"`, or `null` for legacy pre-cutover DEKs) surfaces the binding state on both the mutation result and `EncryptionMetadata`.
* The DEK is cryptographically bound (KMS `EncryptionContext`) to `{oclId, sha256(canonicalized accessControlConditions)}`. Passing a different condition array to `finishCreateOrUpdateFile` than the one used to generate the key produces a permanently undecryptable file.
* Minting is gated by the lab's `ADD_FILES` capability (falling back to `MODIFY_FILES`) — the same rule an open lab's contributions follow.

Full reference: [Data Encryption Keys](labs-api/files.md#data-encryption-keys) and [DEK Binding](../technical-deep-dive/data/data-privacy-and-access.md#dek-binding).

---

### New capability: `DECRYPT_FILES`, and a lab-level gate on `decryptDataKey`

`decryptDataKey` called with an `oclId` is now gated by the new `DECRYPT_FILES` access-policy capability *before* the file's own `accessControlConditions` are evaluated. The default behavior is unchanged (viewer-or-above membership, for both a Privy session and a service token) — what's new is that a lab owner can widen it via `updateLabAccessPolicy`, so a non-member who contributed to an open lab can read their own encrypted submission back. The `tokenUri`-only path (IPFS agreement documents, no `oclId`) is unaffected and keeps its existing paid-access carve-out, but now also goes through the same x402 `allowedMutations` scope check as every other gated mutation (previously bypassed on that branch).

Full reference: [Decrypt Authorization](../technical-deep-dive/data/data-privacy-and-access.md#decrypt-authorization).

---

### Breaking: service-token wallet sign-in moves to EIP-712 + single-use nonces

`getServiceSignInMessage` now returns a stateful, single-use challenge instead of a deterministic string, closing a replay gap where one captured signature could mint tokens indefinitely.

* `message` is now JSON-serialized **EIP-712 typed data** — sign it with `eth_signTypedData_v4` (viem `signTypedData`). Signatures over the old plain-text message (`personal_sign`) are **no longer accepted**.
* The query response gained `nonce`, `issuedAt`, and `expiresAt`. The nonce is single-use, expires after roughly 10 minutes, and must be passed to `generateServiceToken`'s new required-with-signature `nonce` argument. A reused or expired nonce fails `UNAUTHENTICATED` / `details.reason: "INVALID_NONCE"`.
* `expiresIn` on both `generateServiceToken` and `extendServiceToken` is now clamped to **\[1 hour, 2 years]**; out-of-range or malformed values fail `VALIDATION_FAILED` instead of a masked internal error.

Full reference: [Obtaining Tokens](labs-api/service-tokens.md#obtaining-tokens).

---

### GraphQL introspection disabled and query depth capped in production

The production endpoint (shared by all Molecule APIs — see [API Overview](README.md)) no longer serves `__schema` / `__type` introspection queries: they now return a validation error. `__typename` still resolves. Selection-set depth is also capped at 10 in production, with scalar leaves counted as a level (`{ root { child { name } } }` is depth 3). A query beyond that limit fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data — a plain GraphQL error, not the catalogued error shape used elsewhere, so handle both.

**Migration:** If your codegen or tooling discovers the schema by introspecting the production endpoint, that now fails — request a current copy of the schema from the Molecule team (see [Getting Support](README.md)) rather than introspecting production. If you see `QueryDepthLimitReached`, flatten the query to 10 levels of nesting or fewer; this limit was not previously enforced.

---

### `*V2` operations and pre-OCL naming removed

The legacy `*V2` operations and the pre-OCL naming have been **removed**. The current API is `oclId`-based. If you are migrating from an older integration, use the current names below.

#### Renamed queries

| Legacy (removed)                                   | Current                      | Notes                                                     |
| -------------------------------------------------- | ---------------------------- | --------------------------------------------------------- |
| `projects` / `projectsV2`                          | `labs`                       | Same paginated shape; adds an optional `role` filter      |
| `projectWithDataRoomAndFiles` / `…V2`              | `labWithDataRoomAndFiles`    | Look up by `oclId` (or `shortname`) instead of `ipnftUid` |
| `dataRoomFileV2`                                   | `dataRoomFile`               | Identified by `oclId` + `path`                            |
| `projectActivity` / `projectActivityV2`            | `labActivity`                | —                                                         |
| `activitiesV2`                                     | `activities`                 | —                                                         |
| `projectAnnouncementsV2` / `projectAnnouncementV2` | `labActivity` / `activities` | Removed — use the `filter: ANNOUNCEMENT` argument         |

#### Renamed mutations

| Legacy (removed)               | Current                      | Notes                                                                  |
| ------------------------------ | ---------------------------- | ---------------------------------------------------------------------- |
| `createProject`                | `createLab`                  | Now takes `input: { oclId }` instead of `ipnftSymbol` / `ipnftTokenId` |
| `initiateCreateOrUpdateFileV2` | `initiateCreateOrUpdateFile` | —                                                                      |
| `finishCreateOrUpdateFileV2`   | `finishCreateOrUpdateFile`   | —                                                                      |
| `createAnnouncementV2`         | `createAnnouncement`         | Takes `oclId`; the legacy `moleculeAccessLevel` param was removed      |
| `updateFileMetadataV2`         | `updateFileMetadata`         | —                                                                      |
| `deleteDataRoomFileV2`         | `deleteDataRoomFile`         | —                                                                      |

#### Renamed fields

Top-level identifiers on `Lab` / `LabRef` were renamed away from the legacy IP-NFT naming:

| Legacy field   | Current field       | Notes                                                       |
| -------------- | ------------------- | ----------------------------------------------------------- |
| `ipnftUid`     | `oclId`             | Now a 32-byte hex string (`0x…`), not `<address>_<tokenId>` |
| `ipnftSymbol`  | `shortname`         | Human-readable slug derived from the lab name               |
| `ipnftAddress` | `labAccountAddress` | ERC-6551 token-bound account address                        |
| `ipnftTokenId` | `labNftTokenId`     | LabNFT tokenId (decimal string)                             |

> The linked legacy IP-NFT (for labs migrated from one) is still available as the nested `ipnft` object on `Lab` / `LabRef`; `LabRef` additionally exposes a scalar `ipnftId` field.

---

## IPNFT API (Deprecated)

> The IPNFT API is deprecated. The changes below are preserved for integrations that have not yet migrated. See the [IPNFT API reference](ipnft-api-deprecated.md).

### February 2026

#### Breaking Changes

##### Schema Flattening — `metadata` wrapper removed from IPNFT and IPT

The intermediate `metadata` wrapper objects (`IPNFTMetadata`, `IPTMetadata`) have been removed. All metadata fields now live directly on the `IPNFT` and `IPT` types.

```diff
# IPNFT queries — before
- ipnft {
-   metadata {
-     name
-     description
-     topic
-   }
- }

# IPNFT queries — after
+ ipnft {
+   name
+   description
+   topic
+ }

# IPT queries — before
- ipt {
-   metadata {
-     symbol
-     name
-     totalIssued
-   }
- }

# IPT queries — after
+ ipt {
+   symbol
+   name
+   totalIssued
+ }
```

**Migration:** Remove all `metadata { ... }` wrappers and access fields directly on the parent type.

##### `fundingAmount` JSON field decomposed into 4 typed fields

The `fundingAmount` JSON field on IPNFT has been replaced with 4 strongly-typed fields:

```diff
- ipnft { metadata { fundingAmount } }  // JSON object
+ ipnft {
+   fundingAmountCurrency      // e.g., "USDC"
+   fundingAmountValue         // e.g., "1000000"
+   fundingAmountDecimals      // e.g., 6
+   fundingAmountCurrencyType  // e.g., "ERC20"
+ }
```

**Migration example:**

```javascript
// OLD
const amount = ipnft.metadata.fundingAmount; // JSON object

// NEW
const amount = {
  currency: ipnft.fundingAmountCurrency,
  value: ipnft.fundingAmountValue,
  decimals: ipnft.fundingAmountDecimals,
  currencyType: ipnft.fundingAmountCurrencyType
};
```

##### `agreements` changed from JSON array to typed relation

The `agreements` field on IPNFT has changed from a JSON array to a queryable typed relation with sub-field selection:

```diff
- ipnft { metadata { agreements } }  // JSON array
+ ipnft {
+   agreements {  // Typed relation
+     id
+     contentHash
+     mimeType
+     type
+     url
+     ipnftId
+   }
+ }
```

**Migration:** Update your queries to select specific agreement fields instead of receiving a raw JSON array.

##### Filter changes — nested metadata filters removed

All filter paths that previously went through `metadata` are now direct fields:

```diff
# Filtering IP-NFTs by topic
- filterBy: { metadata: { topic: "Oncology" } }
+ filterBy: { topic: "Oncology" }

# Filtering IP-NFTs by organization
- filterBy: { metadata: { organization: "University Lab" } }
+ filterBy: { organization: "University Lab" }

# Filtering IPTs by symbol
- filterBy: { metadata: { symbol: "VITA" } }
+ filterBy: { symbol: "VITA" }
```

##### `researchLead` and `originalOwner` filter paths changed

These relation filters are now direct on the parent type instead of nested inside metadata context:

```diff
# Filter IPNFT by research lead (now direct on IPNFTFilterBy)
- filterBy: { metadata: { researchLead: { email: "..." } } }
+ filterBy: { researchLead: { email: "..." } }

# Filter IPT by original owner (now direct on IPTFilterBy)
- filterBy: { metadata: { originalOwner: { address: "..." } } }
+ filterBy: { originalOwner: { address: "..." } }
```

#### New Features

##### New query types

The following new top-level queries are now available:

| Query | Description |
|-------|-------------|
| `user(id)` / `users(...)` | Query users by address, list associated IP-NFTs and IPTs |
| `researchLead(id)` / `researchLeads(...)` | Query research leads and their associated IP-NFTs |
| `chain(id)` / `chains(...)` | Query blockchain networks and their markets |
| `agreement(id)` / `agreements(...)` | Query legal agreements associated with IP-NFTs |

All new queries support `limit`, `skip`, `sortBy`, `sortOrder`, and `filterBy` parameters.

##### New fields on IPNFT

- `updatedAt` — Last update timestamp
- `tokenUri` — Token metadata URI
- `symbol` — Token symbol
- `schemaVersion` — Metadata schema version
- `userId` — Owner user ID (for direct filtering)
- `researchLeadId` — Research lead ID (for direct filtering)
- `fundingAmountCurrency`, `fundingAmountValue`, `fundingAmountDecimals`, `fundingAmountCurrencyType` — Decomposed funding amount fields

##### New fields on IPT

- `updatedAt` — Last update timestamp
- `mintedAt` — Minting timestamp
- `agreementMimeType` — Agreement file MIME type
- `originalOwner` — Full `User` type (with `id`, `address`)
- `originalOwnerId` — Original owner user ID (for direct filtering)
- `ipnftId` — Parent IP-NFT ID (for direct filtering)
- `links` — Related links
- `capped` — Whether token issuance is capped

##### New fields on Market

- `createdAt`, `updatedAt` — Timestamps
- `inverted` — Whether the pair is inverted
- `iptId` — Associated IPT ID (for direct filtering)
- `chain.logoUrl` — Chain logo URL now available

##### `agreements` queryable as typed relation

Agreements on IP-NFTs are now a fully queryable relation with sub-field selection, filtering, sorting, and pagination:

```graphql
ipnft {
  agreements(limit: 10, sortBy: type, sortOrder: asc) {
    id
    contentHash
    mimeType
    type
    url
  }
}
```
