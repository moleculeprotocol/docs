---
icon: clock-rotate-left
---

# API Changelog & Migration

This page tracks breaking changes, deprecations, and additions across the Molecule APIs. Use it when upgrading an existing integration. Each API's most recent changes are listed first.

---

## Authentication

### The service-token sign-in message is now single-use and expires

`getServiceSignInMessage` used to return a deterministic string — a pure function of `(walletAddress, serviceName)` — which meant one captured signature could mint fresh tokens indefinitely. The message now embeds a **server-issued single-use nonce** and its expiry, and `generateServiceToken` verifies and consumes that nonce:

- The message is **valid for 10 minutes** from issuance. The new `expiresAt` field on `getServiceSignInMessage` reports the deadline.
- The nonce is **consumed on first successful redemption**. Issuing a second token requires a fresh message and a fresh signature.
- There is **one outstanding nonce per `(walletAddress, serviceName)`**, last-write-wins — calling the query again invalidates an unredeemed message.
- Signatures over the older nonce-free message format no longer verify.

Failures come back as `UNAUTHENTICATED` with `details.reason` one of `NONCE_NOT_FOUND` (never requested, or already consumed), `NONCE_EXPIRED`, `INVALID_SIGNATURE` (altered text, or a message superseded by a later call) or `WALLET_MISMATCH`.

**Migration:** Fetch the message immediately before signing, and treat every one of those reasons as "request a new message and sign it again" rather than as a retryable call — re-submitting the same signature can never succeed. Callers that already ran `getServiceSignInMessage` → sign → `generateServiceToken` back to back need no change; callers that cached the message, cached a signature, or reconstructed the string client-side must stop doing so. See [Obtaining a Token](labs-api/service-tokens.md#obtaining-a-token).

### Service tokens are self-issued, and scoped to their own lifecycle

Two clarifications and one hardening, all now reflected across the API docs:

- **Nobody provisions a service token for you.** `generateServiceToken` accepts a wallet signature, so any caller mints its own: `getServiceSignInMessage` → sign the message verbatim (EIP-191 `personal_sign`) → `generateServiceToken`. The only credential that still comes from the Molecule team is the `mol_` consumer credential. Earlier pages described service tokens as team-issued; that was never the only path and is no longer the documented one.
- **A service token is bound to a wallet, not to a lab.** Docs that described it as identifying "which lab you have write access to" were wrong. It carries a wallet identity; what it may do on a given lab is resolved per request from that wallet's live onchain role. One token therefore works across every lab the wallet has a role on, and a role granted *after* issuance takes effect without re-issuing.
- **`extendServiceToken` and `revokeServiceToken` are scoped to the caller's own tokens.** The presented token must own the `tokenId` it names; a `tokenId` belonging to another wallet returns the byte-identical `NOT_FOUND` of one that does not exist, so token existence cannot be enumerated.

**Migration:** None required if you already self-issue. If you hold a team-provisioned token, it keeps working — but you can mint and rotate your own. If you built per-lab token issuance, you can collapse it to one token per wallet. If any code called `extendServiceToken` / `revokeServiceToken` for a `tokenId` issued to a different wallet, it now receives `NOT_FOUND`. See [Authentication](authentication.md#obtaining-a-service-token) and [Service Tokens](labs-api/service-tokens.md#obtaining-a-token).

### Contributor role parity for service-token content writes

The six content-write mutations — `initiateCreateOrUpdateFile`, `finishCreateOrUpdateFile`, `createAnnouncement`, `deleteDataRoomFile`, `updateFileMetadata`, `moveEntry` — now gate a service token on the **Contributor** role, matching the Privy user path per mutation. Previously the service path required lab ownership for these, which meant a wallet granted Contributor on a lab could act through a user session but not through its own service token.

This is what unblocks the "human owns the lab, agent contributes to it" flow: the owner grants the agent's wallet Contributor, and the agent's self-issued token can write. Owner-gated surfaces are unchanged — `createLab`, `updateLabNftMetadata` and `generateLabImageUploadUrl` still require ownership.

**Migration:** None — this is a widening. Note that role state reaches the API through an event indexer, so a write can still return `UNAUTHORIZED` for a few seconds after a grant confirms onchain; retry with backoff rather than re-issuing the token. Walkthrough: [Tutorial 3](getting-started/tutorial-3-agent-access.md).

### Backend credential stores confined to the platform network

The data stores behind API authentication — the consumer credential registry, machine service tokens, and the access whitelist — are now network-confined to Molecule's private cloud network. They are unreachable from outside it, even with valid cloud-account credentials; only the API's own backend can read or write them. This is a hardening change with **no effect on any API, header, token format, or SDK** — consumer credentials, `X-Service-Token`, and Privy user tokens all work exactly as before.

**Migration:** None required.

### `x-api-key` replaced by consumer credentials

All Molecule APIs (Labs, Tokenization, and IPNFT (Deprecated) — they share one GraphQL endpoint) now authenticate with a consumer credential instead of an `x-api-key` header. A consumer credential has the shape `mol_<consumerId>_<secret>` and is sent directly as the `Authorization` header value, with **no `Bearer` prefix**.

```diff
- x-api-key: YOUR_API_KEY
+ Authorization: mol_<consumerId>_<secret>
```

**Migration:** Request a consumer credential from the Molecule team ([template](getting-started/README.md#1-a-mol-consumer-credential-the-one-manual-step)) and send it as `Authorization: mol_<consumerId>_<secret>` instead of `x-api-key` — do not prefix it with `Bearer`, which is reserved for Privy user tokens and will fail authentication. Nothing else changes: `X-Service-Token` for machine-authorized mutations, and `Authorization: Bearer <Privy token>` + `x-wallet-address` for user-authorized mutations, work exactly as before. See [Authentication](authentication.md) for the full header reference.

---

## Labs API

### Assignment Agreement is no longer a gate, and is out of the API docs

Signing the assignment agreement is **not** a precondition for `createLab`, for uploading files, or for any other Labs API operation. It was previously presented as a required onboarding step, and the `<AGREEMENT_TYPE>_NOT_SIGNED` / `AGREEMENT_CHECK_UNAVAILABLE` failure causes are now dormant — reserved in the catalogue, but not emitted.

The `legalAgreement*` operations remain in the schema and are unchanged, but they have been removed from every onboarding and reference flow, and [Legal Agreements](labs-api/legal-agreements.md) is out of the site navigation. Do not build a new integration around them.

**Migration:** Delete any agreement-signing step from your workflow — it does nothing. If you branch on an agreement status before writing, remove the branch. Nothing in the API changed; only what is required of you did. The onboarding tutorials no longer include the step: [Tutorials](getting-started/README.md).

### Staging introspection is the supported way to get the schema

Production has introspection disabled (below), but **staging has it enabled** — point codegen, a playground or an SDK generator at `https://staging.graphql.api.molecule.xyz/graphql` with your consumer credential and generate normally. Both environments serve the same schema, so generate against staging and point the generated client at production.

This supersedes the earlier guidance to request a copy of the schema from the Molecule team.

**Migration:** If your codegen currently fails against production, repoint it at staging. See [Getting the schema](getting-started/README.md#getting-the-schema).

### GraphQL introspection disabled and query depth capped in production

The production endpoint (shared by all Molecule APIs — see [API Overview](README.md)) no longer serves `__schema` / `__type` introspection queries: they now return a validation error. `__typename` still resolves. Selection-set depth is also capped at 10 in production, with scalar leaves counted as a level (`{ root { child { name } } }` is depth 3). A query beyond that limit fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data — a plain GraphQL error, not the catalogued error shape used elsewhere, so handle both.

**Migration:** If your codegen or tooling discovers the schema by introspecting the production endpoint, that now fails — introspect **staging** instead, where it is enabled, and point the generated client at production (see [Getting the schema](getting-started/README.md#getting-the-schema)). If you see `QueryDepthLimitReached`, flatten the query to 10 levels of nesting or fewer; this limit was not previously enforced.

### `isSuccess` removed (unified error contract)

The Labs API now reports failure one way per operation class, and the `isSuccess` envelope is gone:

- **Queries throw.** A failed query surfaces as an entry in the top-level GraphQL `errors[]` array; the failed field is `null`, and because most Labs query fields are non-null the null propagates and `data` itself comes back `null` (only `labWithDataRoomAndFiles` and `dataRoomFile` null just their own field). `errorType` carries the error code (table below) — the only value to branch on — and `errorInfo { requestId, retryable, details }` carries the correlation id and structured context. Query result types (`ActivitiesResult`, `FileCategoriesAndTagsResult`, `ListLabMembersResult`, `DidLinkStatusResult`, `LegalAgreementTemplateResult`, `ServiceSignInMessageResult`) no longer have an `isSuccess` or an `error` field.
- **Mutations return errors in-band.** Every mutation `*Result` carries a nullable `error: ApiError`. **Success ⇔ `error == null`.** A top-level `errors[]` entry on a mutation means a transport or infrastructure failure (or an invalid request document), not a business outcome. Where a result keeps a top-level `message`, it mirrors `error.message` on failure and is never empty.

`isSuccess` has been **removed** from every Labs result type, and the `error` field on mutation results is now typed `ApiError` (the previous error type no longer exists). A document that still selects `isSuccess` is rejected at validation (`Field 'isSuccess' in type '…Result' is undefined`) and never executes.

Behaviour that changed alongside the envelope:

- **Auth denials are code-accurate.** The former `AUTH_FAILED` catch-all is split into `UNAUTHENTICATED` (missing, invalid or expired credentials), `UNAUTHORIZED` (authenticated but lacking the role or membership), `NOT_FOUND` (the lab does not exist) and `VALIDATION_FAILED` (malformed input).
- **Legacy codes live on as `details.reason`.** Codes such as `PROJECT_NOT_FOUND`, `LAB_NOT_FOUND`, `INVALID_OCL_ID` or `SHORTNAME_TAKEN` are no longer top-level codes; they survive as the `reason` key inside `details`, beneath the catalogue code (for example `CONFLICT` with `reason: "SHORTNAME_TAKEN"`). Branch on `code` first and on `reason` only where a page documents it.
- **Service tokens.** `ServiceTokenResult` and `ServiceTokenRevocationResult` gained `error: ApiError`; `token`, `tokenId`, `serviceName`, `expiresAt`, `createdAt` and `revokedAt` are `null` on failure instead of `""`.
- **`InitiateFileUploadResult.method`** is nullable and `null` on failure.
- **`LegalAgreementStatusResult` is dual-surface.** As the `legalAgreementStatus(oclId, type)` query, failures throw and `error` is always `null`. As the `legalAgreementStatus` field on `Lab` / `LabRef`, an upstream failure returns the payload with `error` set (typically `UPSTREAM_UNAVAILABLE`) instead of nulling the lab: `error != null` means the status could not be determined, and only when `error == null` does `signed: false` mean "not signed".
- `listLabMembers` on an unknown lab throws `NOT_FOUND` (previously a `PROJECT_NOT_FOUND` envelope). `createLab` conflicts return `CONFLICT` with `details.reason` `PROJECT_CONFLICT` (lab already registered) or `ACCOUNT_NAME_CONFLICT`.

> **The Tokenization API is unaffected.** Its operations keep their previous result envelope (a success flag alongside an `error` object), exactly as documented in the [Tokenization API reference](tokenization-api.md). This section applies to the Labs API only.

#### `ApiError`

```graphql
type ApiError {
  code: String!       # error code from the table below — the only field to branch on
  message: String!    # human-readable, never empty; not part of the contract
  requestId: String!  # correlation id — include it in bug reports
  retryable: Boolean! # whether an unchanged retry can plausibly succeed
  details: AWSJSON    # optional structured context — keys: field, reason, hint, docs
}
```

On an in-band mutation error, `details` arrives as a JSON-encoded string (AppSync `AWSJSON`), e.g. `"details": "{\"reason\":\"NOT_LAB_OWNER\"}"`; on a thrown query error, `errorInfo.details` is a plain object. The in-band string is currently encoded twice, so read it with the tolerant [`parseDetails`](labs-api/README.md#error-handling) rather than a single `JSON.parse`, which returns another string and makes `.reason` silently `undefined`. Ignore keys you do not recognise, and never match on `message` — its wording may change without notice.

#### Error codes

| Code                        | `retryable` | Meaning                                                                  | Typical `details.reason`                                                         |
| --------------------------- | ----------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| `UNAUTHENTICATED`           | false       | Missing, invalid or expired credentials.                                 | `TOKEN_EXPIRED`, `INVALID_SIGNATURE`, `WALLET_MISMATCH`                          |
| `UNAUTHORIZED`              | false       | Authenticated but not allowed (role or membership).                      | `NOT_LAB_OWNER`, `NOT_CONTRIBUTOR`, `SERVICE_NOT_WHITELISTED`                    |
| `NOT_FOUND`                 | false       | The referenced resource does not exist.                                  | `LAB_NOT_FOUND`, `PROJECT_NOT_FOUND`, …                                          |
| `VALIDATION_FAILED`         | false       | Input failed validation; `details.field` names the offending field.      | `INVALID_OCL_ID`, …                                                              |
| `CONFLICT`                  | false       | A valid request conflicts with current state.                            | `SHORTNAME_TAKEN`, `ALREADY_SIGNED`, `PROJECT_CONFLICT`, `ACCOUNT_NAME_CONFLICT` |
| `FAILED_PRECONDITION`       | false       | Resource state makes the operation impossible until that state changes.  | `TEMPLATE_EXPIRED`, `LEGACY_ENCRYPTION`, `NOT_ENCRYPTED`                         |
| `COMPLEXITY_LIMIT_EXCEEDED` | false       | Query shape or result size is over the limit.                            | `FILTER_COMPLEXITY_LIMIT`, `RESULT_CARDINALITY_LIMIT`                            |
| `RATE_LIMITED`              | **true**    | Throttled — retry with backoff.                                          | —                                                                                |
| `TIMEOUT`                   | **true**    | Execution exceeded the request budget.                                   | —                                                                                |
| `UPSTREAM_UNAVAILABLE`      | **true**    | A dependency failed — retry with backoff.                                | `KAMU`, `CMS`, `IPFS`                                                            |
| `INTERNAL_ERROR`            | **true**    | Unexpected failure; quote `requestId` when reporting it.                 | —                                                                                |

Codes may be added over time, and each addition is announced on this page. Treat a code you do not recognise as non-retryable, keep the raw value for diagnostics and surface it to a human. `PAYMENT_REQUIRED` is reserved for the x402 gateway and is not emitted by the GraphQL API. `details.reason` values are diagnostic refinement, not a contract surface — they may be extended without notice.

#### Before / after

```diff
# Query selection (listLabMembers) — the envelope field is gone; failures
# arrive as top-level errors[] with errorType
  query ListLabMembers($oclId: String!) {
    listLabMembers(oclId: $oclId) {
-     isSuccess
      message
      members {
        walletAddress
        role
      }
    }
  }
```

```diff
# Mutation selection (createAnnouncement) — select `error` instead of `isSuccess`
  mutation CreateAnnouncement($oclId: String!, $headline: String!, $body: String!) {
    createAnnouncement(oclId: $oclId, headline: $headline, body: $body) {
-     isSuccess
      message
-     error { message code retryable }
+     error { code message requestId retryable details }
    }
  }
```

```diff
- if (!result.isSuccess) {
-   handle(result.error?.code);
- }
+ if (result.error) {
+   const { reason } = parseDetails(result.error.details); // tolerant parse, see Error Handling
+   handle(result.error.code, reason);
+ }
```

**Migration:** Remove `isSuccess` (and, except on `legalAgreementStatus`, any `error { … }` selection) from every Labs query document — a document that still selects it fails validation and the query never runs — and handle failures from the top-level `errors[]` array, keyed on `errorType`. On mutations, replace `isSuccess` with `error { code message requestId retryable details }`, treat `error == null` as success, and branch on `error.code` (plus `details.reason` where documented), never on `message` text. Replace any check on the former `AUTH_FAILED` catch-all with the split codes listed above, and any `token === ""` / `tokenId === ""` checks on service-token results with `null` checks. If you read `legalAgreementStatus` off a lab object, select `error { code message }` and check it before trusting `signed`. Leave your Tokenization API handling as it is. See [Labs API › Error Handling](labs-api/README.md#error-handling) for the full reference.

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

## x402 Gateway

### Gateway base URLs published, and the 402 challenge is a header

The staging and production gateway base URLs are now published on the [x402 Gateway](x402-gateway.md#gateway-base-urls) page — they no longer have to be requested.

Along with them, one correction that matters for anyone implementing the handshake: the payment requirements arrive as **base64-encoded JSON in the `payment-required` response header**, not in the `402` response body. The body is only `{"isSuccess":false,"message":"Payment required"}`. Client code that parsed the body for `accepts` never saw a price.

**Migration:** Read the challenge from the `payment-required` header and base64-decode it; take `amount`, `asset`, `network` and `payTo` from `accepts[0]`. `amount` is in the asset's smallest unit (USDC has 6 decimals, so `"10000"` is $0.01). Worked example: [Reading the 402 challenge](x402-gateway.md#reading-the-402-challenge).

Also worth knowing: payment buys a short-lived service token for the payer wallet, **not** a role. A mutation the payer is not authorized for returns `200` with `error.code: "UNAUTHORIZED"` and is still settled — check the target lab and your role on it (both free, public queries) before signing.

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
query GetAgreements($id: ID!) {
  ipnft(id: $id) {
    agreements(limit: 10, sortBy: type, sortOrder: asc) {
      id
      contentHash
      mimeType
      type
      url
    }
  }
}
```
