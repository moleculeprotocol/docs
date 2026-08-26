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

### GraphQL introspection disabled and query depth capped in production

The production endpoint (shared by all Molecule APIs — see [API Overview](README.md)) no longer serves `__schema` / `__type` introspection queries: they now return a validation error. `__typename` still resolves. Selection-set depth is also capped at 10 in production, with scalar leaves counted as a level (`{ root { child { name } } }` is depth 3). A query beyond that limit fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data — a plain GraphQL error, not the catalogued error shape used elsewhere, so handle both.

**Migration:** If your codegen or tooling discovers the schema by introspecting the production endpoint, that now fails — request a current copy of the schema from the Molecule team (see [Getting Support](README.md)) rather than introspecting production. If you see `QueryDepthLimitReached`, flatten the query to 10 levels of nesting or fewer; this limit was not previously enforced.

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

On an in-band mutation error, `details` arrives as a JSON-encoded string (AppSync `AWSJSON`), e.g. `"details": "{\"reason\":\"NOT_LAB_OWNER\"}"` — read it with `JSON.parse(error.details ?? "{}")`. On a thrown query error, `errorInfo.details` is a plain object. Ignore keys you do not recognise, and never match on `message` — its wording may change without notice.

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
+   const { reason } = JSON.parse(result.error.details ?? "{}");
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
