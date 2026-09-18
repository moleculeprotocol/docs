---
description: Generated reference for every type in the Molecule GraphQL schema.
---

# API types

Every object, input, enum and union in the Molecule GraphQL schema, generated from the schema itself. Operation arguments live with their operation in the guide pages; this page is what those tables refer to.

## AgreementSortBy

| Value | Description |
| --- | --- |
| `id` |  |
| `contentHash` |  |
| `mimeType` |  |
| `type` |  |
| `url` |  |
| `ipnftId` |  |

## AssignmentAgreementType

Which assignment agreement was generated. Derived from the IPNFT id, never supplied by the caller.

| Value | Description |
| --- | --- |
| `POI_ASSIGNMENT` | Proof-of-Invention assignment, used when the IPNFT id exceeds the uint128 range. |
| `RESEARCH_ASSIGNMENT` | Research assignment, used when the IPNFT id is within the uint128 range. |

## ChainSortBy

| Value | Description |
| --- | --- |
| `id` |  |
| `createdAt` |  |
| `updatedAt` |  |
| `name` |  |
| `chainId` |  |
| `logoUrl` |  |
| `market` | Not a usable sort key: it names no `Chain` field, so it is ignored and the order stays unspecified. |

## DataRoomAccessLevel

Who may read a data room file. The level is a label on the file; the restriction itself comes from encryption (see `ADMIN`).

| Value | Description |
| --- | --- |
| `PUBLIC` | Anyone. |
| `HOLDERS` | Intended for files readable by the lab's token holders, but neither assigned nor enforced today. |
| `ADMIN` | Confidential, readable only by lab members. Enforced by envelope encryption, so the URL alone yields ciphertext. |

## DidLinkingStatus

States of the background DID-linking state machine.

| Value | Description |
| --- | --- |
| `PENDING` | Linking is queued or being prepared. |
| `SUBMITTED` | User operation has been submitted onchain and awaits confirmation. |
| `LINKED` | Both DIDs are linked onchain; terminal. |
| `FAILED` | Most recent attempt failed; the worker may retry. |

## IPNFTSortBy

| Value | Description |
| --- | --- |
| `id` | Lexicographic, not numeric: `10` sorts before `9`. |
| `createdAt` |  |
| `updatedAt` |  |
| `mintedAt` |  |
| `chainId` |  |
| `owner` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. Use `userId` instead. |
| `userId` |  |
| `originalOwner` |  |
| `ipt` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. |
| `tokenUri` |  |
| `symbol` |  |
| `name` |  |
| `image` |  |
| `description` |  |
| `externalUrl` |  |
| `initialSymbol` |  |
| `organization` |  |
| `topic` |  |
| `fundingAmountCurrency` |  |
| `fundingAmountValue` | Numeric, on the stored integer; `fundingAmountDecimals` varies, so the order is not monetary. |
| `fundingAmountDecimals` |  |
| `fundingAmountCurrencyType` |  |
| `researchLead` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. Use `researchLeadId` instead. |
| `researchLeadId` |  |
| `agreements` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. |
| `schemaVersion` |  |

## IptAgreementType

Kind of IPT (IP Token) agreement.

| Value | Description |
| --- | --- |
| `IPT_MEMBERSHIP` | Membership terms an IPT holder accepts. |

## IPTSortBy

| Value | Description |
| --- | --- |
| `id` |  |
| `createdAt` |  |
| `updatedAt` |  |
| `ipnft` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. Use `ipnftId` instead. |
| `ipnftId` |  |
| `l2TokenAddress` |  |
| `holderCount` |  |
| `markets` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. |
| `name` |  |
| `symbol` |  |
| `decimals` |  |
| `agreementCid` |  |
| `agreementMimeType` |  |
| `originalOwner` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. Use `originalOwnerId` instead. |
| `originalOwnerId` |  |
| `image` |  |
| `links` | Not a usable sort key (a list); the request fails with `INTERNAL_ERROR`. |
| `capped` |  |
| `circulatingSupply` | Lexicographic, not numeric: the supply is stored as a decimal string. |
| `totalIssued` | Lexicographic, not numeric: the total is stored as a decimal string. |

## LabActivityFilter

Kind of activity-feed entry. Omit the filter to receive every kind.

| Value | Description |
| --- | --- |
| `ANNOUNCEMENT` | Announcements only. |
| `FILE` | File events only (added, updated, removed). |

## LabMemberRole

Onchain role of a lab member. Roles are granted and revoked onchain through the lab's AccessResolver contract (`grantRole(oclId, account, role, expiry, isAgent)`, role 1 = VIEWER, 2 = CONTRIBUTOR; OWNER is whoever holds the LabNft). This API reflects those grants and cannot change them.

| Value | Description |
| --- | --- |
| `OWNER` | Holds the lab's LabNft; full control of the lab and its data room. |
| `CONTRIBUTOR` | May write to the data room (upload files, post announcements). |
| `VIEWER` | Read-only membership: may read the data room, including confidential files, but not write to it. |

## LabMemberSource

Where a membership record came from. `ONCHAIN_EVENT` and `MULTISIG_RESOLUTION` derive from canonical owner state, `ACCESS_CONTRACT` is the legacy V2 IPNFT-auth contract, and `ACCESS_RESOLVER_EVENT` records stream from AccessResolver V3 role events and may have an expiry.

| Value | Description |
| --- | --- |
| `ONCHAIN_EVENT` | Derived from the LabNft ownership events onchain. |
| `MULTISIG_RESOLUTION` | Derived by resolving a multisig owner to its signers. |
| `ACCESS_CONTRACT` | Granted through the legacy V2 IPNFT access contract. |
| `ACCESS_RESOLVER_EVENT` | Granted by an AccessResolver V3 RoleGranted event; the only source whose grants can carry an `expiry`. |

## LabOrderField

Sort key for `labsConnection`. The server always appends the lab's unique `oclId` as a final tiebreaker in the same direction as the primary key, so the total order, and therefore every cursor, is stable.

| Value | Description |
| --- | --- |
| `MINTED_AT` | LabNft mint block timestamp (`mintedAt`). Labs whose mint time is not yet recorded sort last. |
| `LATEST_CONTRIBUTION_AT` | Time of the lab's most recent data room activity (`latestContributionAt`). Labs with no recorded activity sort last. |
| `NAME` | Lab display name (`name`), in the server's string collation order. Labs without a name sort last. |
| `TRL_VALUE` | Numeric rank of `trlValue`. Labs with no assessment or a `pre-trl-*` value sort last in either direction. |

## LegalAgreementType

Legal agreements a lab owner can sign through this API. Distinct from the IPNFT-side `Agreement` type and `agreement(s)` queries, which describe the documents attached to an IP-NFT's metadata.

| Value | Description |
| --- | --- |
| `ASSIGNMENT_AGREEMENT` | IP assignment agreement between the lab owner and the lab. |

## MarketSortBy

| Value | Description |
| --- | --- |
| `id` |  |
| `createdAt` |  |
| `updatedAt` |  |
| `liquidityUsd` |  |
| `pairAddress` |  |
| `usdPrice` |  |
| `usdPrice24hrPercentageChange` |  |
| `chain` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. Use `chainId` instead. |
| `chainId` |  |
| `marketCapUsd` |  |
| `tradingVolume24hr` |  |
| `token` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. Use `iptId` instead. |
| `iptId` |  |

## OclAgreementType

Kind of OCL (Onchain Lab) agreement.

| Value | Description |
| --- | --- |
| `OCL_MEMBERSHIP` | Membership terms a Lab token holder accepts. |

## OrderDirection

Sort direction for an ordering key.

| Value | Description |
| --- | --- |
| `ASC` |  |
| `DESC` |  |

## ResearchLeadSortBy

| Value | Description |
| --- | --- |
| `id` |  |
| `createdAt` |  |
| `updatedAt` |  |
| `name` |  |
| `email` |  |
| `ipnfts` | Not a usable sort key; the request fails with `INTERNAL_ERROR`. |

## SortOrder

Sort direction for the `sortOrder` argument of the list queries.

| Value | Description |
| --- | --- |
| `asc` | Ascending (the default when `sortBy` is given without `sortOrder`). |
| `desc` |  |

## TokenKind

Classification of token contracts, established by onchain events or asserted for manually linked contracts.

| Value | Description |
| --- | --- |
| `IPT` | Legacy IP Token. |
| `LAB_TOKEN` | Native lab token minted for the lab. |
| `WRAPPED_LAB_TOKEN` | Existing ERC-20 used as a lab token, with its wrapper in `Token.wrapperAddress`. |
| `BRIDGED` | Bridged copy of another token, connected to it by a `BRIDGE_OF` relation. |
| `LOCKED` | Locked-token wrapper, connected to its underlying token by a `LOCKS` relation. |
| `AGENT` | Agent persona token. |
| `EXTERNAL` | Placeholder for a manually linked or related contract, upgraded when its classification becomes known. |

## TokenLinkSource

How a token's lab link, or a relation between tokens, was established.

| Value | Description |
| --- | --- |
| `ONCHAIN_EVENT` | Derived from an onchain event. |
| `MANUAL` | Set by a lab owner through `linkToken` or `unlinkToken`. |
| `MIGRATION` | Carried over from earlier records. |

## TokenOrderField

Values for sorting tokens, with unique token `id` appended as a tiebreaker in the primary key's direction.

| Value | Description |
| --- | --- |
| `CREATED_AT` |  |
| `TOKENIZED_AT` | Tokenization block time; unknown timestamps sort last in either direction. |
| `SYMBOL` | Token symbol, compared case-sensitively. |
| `HOLDER_COUNT` | Number of holders; unknown counts sort last in either direction. |

## TokenRelationType

Directed relationship from a dependent token to its canonical or underlying token.

| Value | Description |
| --- | --- |
| `BRIDGE_OF` | `token` is a bridged copy of `related`, the canonical token. |
| `LOCKS` | `token` locks `related`, the underlying token. |
| `WRAPS` | `token` wraps `related`, the underlying token. |

## UserSortBy

| Value | Description |
| --- | --- |
| `id` |  |
| `createdAt` |  |
| `updatedAt` |  |
| `address` |  |
| `ipnft` | Not a usable sort key: it names no `User` field, so it is ignored and the order stays unspecified. |
| `ipt` | Not a usable sort key: it names no `User` field, so it is ignored and the order stays unspecified. |

## AgreementFilterBy

Exact-match filters for `agreements`. Every given field must match exactly (case-sensitive); fields are combined with AND. Timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String` |  |
| `contentHash` | `String` |  |
| `mimeType` | `String` |  |
| `type` | `String` |  |
| `url` | `String` |  |
| `ipnftId` | `String` |  |

## ChainFilterBy

Exact-match filters for `chains`. Every given field must match exactly (case-sensitive); fields are combined with AND. Timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `Int` |  |
| `createdAt` | `String` |  |
| `updatedAt` | `String` |  |
| `name` | `String` |  |
| `chainId` | `Int` |  |
| `logoUrl` | `String` |  |
| `market` | `String` | Not usable: it does not name a `Chain` field, so it is ignored. |

## CreateLabInput

Input for creating a new lab. The onchain LabNft (oclId) must already exist; this mutation registers the lab and creates its data room.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | Canonical 32-byte oclId (lowercase 0x-hex) of the onchain lab. |

## EncryptionMetadataInput

Encryption metadata stored with an encrypted (`ADMIN`) file, passed to `finishCreateOrUpdateFile` after the ciphertext upload. `kms` also requires `encryptedDek`, `iv` and `contentHash`, `bls` adds `keyId`, and omitting `encryptionSystem` requires every `Legacy:` field. A missing field fails with `INTERNAL_ERROR`.

| Name | Type | Description |
| --- | --- | --- |
| `encryptionSystem` | `String` | Encryption system identifier: "kms", "bls", or null/absent for a legacy ciphertext written before the onchain-verified envelope cutover. |
| `accessControlConditions` | `String!` | JSON-encoded array of access control conditions that `decryptDataKey` evaluates onchain, left to right, against the caller's wallet before releasing the key. Elements alternate between conditions and boolean operators `{ "operator": "and" \| "or" }`. A condition is either `{ "conditionType": "evmContract", "chain", "contractAddress", "functionName", "functionParams", "functionAbi", "returnValueTest": { "key", "comparator", "value" } }` or `{ "conditionType": "evmBasic", "chain", "contractAddress", "method", "parameters", "returnValueTest" }`. The literal `":userAddress"` in `functionParams` / `parameters` is replaced by the caller's wallet at evaluation time. Accepted `chain` values: "ethereum", "eth", "base", "sepolia", "sepolia-testnet", "sepolia-base", "baseSepolia". The Molecule app makes a confidential file readable by all lab members with two conditions joined by "or", both on the lab's AccessResolver contract: `hasRole(oclId, ":userAddress", "1")` = true (viewer or above) and `isAuthorizedSignerForTba(":userAddress", labAccountAddress)` = true (the LabNft owner). Evaluation fails closed. |
| `encryptedBy` | `String!` | Address that performed the encryption. |
| `encryptedAt` | `String!` | ISO 8601 timestamp when the file was encrypted. |
| `encryptedDek` | `String` | KMS/BLS: Base64-encoded wrapped key, as returned by `generateDataEncryptionKey`. |
| `iv` | `String` | KMS/BLS: Base64-encoded initialization vector. |
| `contentHash` | `String` | KMS/BLS: Hash of the encrypted content. |
| `keyId` | `String` | BLS: Key identifier. |
| `dataToEncryptHash` | `String` | Legacy: Hash of the data to be encrypted, recorded by the legacy client. |
| `chain` | `String` | Legacy: Blockchain network used for access control by the legacy client. |
| `litSdkVersion` | `String` | Legacy: SDK version that produced the legacy ciphertext. |
| `litNetwork` | `String` | Legacy: Network identifier from the legacy client (e.g., datil-test, habanero). |
| `templateName` | `String` | Legacy: Template name for the access control pattern. |
| `contractVersion` | `String` | Legacy: Version of the encryption contract. |

## IntFilter

Positive integer filter requiring exactly one of `eq` or `in`. Invalid or empty operators fail with `VALIDATION_FAILED`.

| Name | Type | Description |
| --- | --- | --- |
| `eq` | `Int` | Exact match on a positive chain id. |
| `in` | `[Int!]` | Set of matching positive chain ids, with 1-100 values. |

## IPNFTFilterBy

Exact-match filters for `ipnfts`. Every given field must match exactly (case-sensitive); fields are combined with AND. Nested filters (`owner`, `researchLead`) match on the related record's fields, and timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String` |  |
| `createdAt` | `String` |  |
| `updatedAt` | `String` |  |
| `chainId` | `Int` |  |
| `owner` | `UserFilterBy` | Match on the current owner's fields, e.g. `{ address }`. |
| `userId` | `String` |  |
| `originalOwner` | `String` | Match the stored casing exactly. |
| `ipt` | `String` | Not usable: `ipt` is a relation, not a text field; a value here makes the request fail. |
| `tokenUri` | `String` |  |
| `symbol` | `String` |  |
| `name` | `String` |  |
| `image` | `String` |  |
| `description` | `String` |  |
| `externalUrl` | `String` |  |
| `initialSymbol` | `String` |  |
| `organization` | `String` |  |
| `topic` | `String` |  |
| `fundingAmountCurrency` | `String` |  |
| `fundingAmountValue` | `String` | Compared as a decimal string, not numerically. |
| `fundingAmountDecimals` | `Int` |  |
| `fundingAmountCurrencyType` | `String` |  |
| `researchLead` | `ResearchLeadFilterBy` | Match on the research lead's fields, e.g. `{ email }`. |
| `researchLeadId` | `String` |  |
| `agreements` | `AgreementFilterBy` | Not usable: `agreements` is a list relation, which this exact-match filter cannot express; a value here makes the request fail. |
| `schemaVersion` | `String` |  |

## IPTFilterBy

Exact-match filters for `ipts`. Every given field must match exactly (case-sensitive); fields are combined with AND. Nested filters (`ipnft`, `originalOwner`) match on the related record's fields, and timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String` |  |
| `createdAt` | `String` |  |
| `updatedAt` | `String` |  |
| `ipnft` | `IPNFTFilterBy` | Match on the parent IP-NFT's fields, e.g. `{ topic }`. |
| `ipnftId` | `String` |  |
| `l2TokenAddress` | `String` | Match the stored casing exactly. |
| `holderCount` | `String` | Not usable: the holder count is numeric and this filter is text; a value here makes the request fail. |
| `markets` | `String` | Not usable: `markets` is a relation, not a text field; a value here makes the request fail. |
| `name` | `String` |  |
| `symbol` | `String` |  |
| `decimals` | `Int` |  |
| `agreementCid` | `String` |  |
| `agreementMimeType` | `String` |  |
| `originalOwner` | `UserFilterBy` | Match on the original owner's fields, e.g. `{ address }`. |
| `originalOwnerId` | `String` |  |
| `image` | `String` |  |
| `links` | `String` | Not usable: `links` is a list, which this exact-match filter cannot express; a value here makes the request fail. |
| `capped` | `String` | Not usable: `capped` is a boolean and this filter is text; a value here makes the request fail. |
| `circulatingSupply` | `String` | Compared as a decimal string, not numerically. |
| `totalIssued` | `String` | Compared as a decimal string, not numerically. |

## LabFilter

Filter for `labsConnection`. Populated fields combine with AND. Validation is strict: malformed values are VALIDATION_FAILED, never silently ignored.

| Name | Type | Description |
| --- | --- | --- |
| `name` | `StringFilter` | Match on the lab display name. |
| `shortname` | `StringFilter` | Match on the lab shortname. |
| `oclIds` | `[String!]` | Batch lookup: labs whose oclId is in this set (1-100 ids). |
| `ipnftIds` | `[String!]` | Batch lookup: labs whose linked IPNFT id is in this set (1-100 ids). |
| `isVerified` | `Boolean` | Molecule's verification decision. Labs with no recorded decision count as unverified, so `isVerified: false` matches them too. |
| `hasIpnft` | `Boolean` | True selects labs with a linked IPNFT, false selects labs without one, and omitting it returns all. |
| `hasDataRoom` | `Boolean` | True selects labs whose data room exists, false selects minted labs without one, and omitting it returns all. |
| `trlValue` | `TrlValueFilter` | Minimum Technology Readiness Level; see `TrlValueFilter`. |
| `member` | `LabMembershipFilter` | Wallet address; selects labs where that wallet holds an active role. |

## LabMembershipFilter

Membership predicate: labs where the wallet holds an active role. Applied in SQL before pagination, so `totalCount` and cursors reflect the filtered set.

| Name | Type | Description |
| --- | --- | --- |
| `walletAddress` | `String!` | EVM wallet address of the member; any checksum casing is accepted. |
| `role` | `LabMemberRole` | Restrict to one role; omit to accept any active role. |

## LabOrderInput

One ordering key for `labsConnection`.

| Name | Type | Description |
| --- | --- | --- |
| `field` | `LabOrderField!` | Lab attribute to order by. |
| `direction` | `OrderDirection!` | Direction for this key; the server applies it to the `oclId` tiebreaker as well. |

## LinkTokenInput

Token to link to a lab. New tokens require a name and symbol, supplied here or read from the contract; unreadable metadata fails with `VALIDATION_FAILED`, reason `TOKEN_METADATA_UNREADABLE`. Tracked tokens retain their stored metadata.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | Lab id as 32-byte 0x-hex; the caller must own this lab. |
| `chainId` | `Int!` | Indexed chain id; others fail with `VALIDATION_FAILED`. |
| `address` | `String!` | Contract address, accepting any checksum casing; stored lowercase. |
| `kind` | `TokenKind!` | Classification to assert: `EXTERNAL` or `BRIDGED` for new tokens. Existing kinds must match unless upgrading `EXTERNAL` to `BRIDGED`, or the request fails. |
| `name` | `String` | Display name for a new token, read from its contract when omitted; ignored for tracked tokens. |
| `symbol` | `String` | Ticker symbol for a new token, read from its contract when omitted; ignored for tracked tokens. |
| `decimals` | `Int` | Decimal places for a new token, from 0-255; ignored for tracked tokens. When omitted, read from the contract with a fallback of 18. |
| `relations` | `[TokenRelationInput!]` | Up to 10 relations to add, preserving existing edges. Duplicates and self-relations fail with `VALIDATION_FAILED`. |

## MarketFilterBy

Exact-match filters for `markets`. Every given field must match exactly (case-sensitive); fields are combined with AND. Nested filters (`chain`, `token`) match on the related record's fields, and timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String` |  |
| `createdAt` | `String` |  |
| `updatedAt` | `String` |  |
| `liquidityUsd` | `String` | Not usable: liquidity is numeric and this filter is text; a value here makes the request fail. |
| `pairAddress` | `String` | Match the stored casing exactly. |
| `usdPrice` | `String` | Not usable: the price is numeric and this filter is text; a value here makes the request fail. |
| `usdPrice24hrPercentageChange` | `String` | Not usable: the price change is numeric and this filter is text; a value here makes the request fail. |
| `chain` | `ChainFilterBy` | Match on the chain's fields, e.g. `{ chainId }`. |
| `chainId` | `Int` |  |
| `marketCapUsd` | `String` | Not usable: market cap is numeric and this filter is text; a value here makes the request fail. |
| `tradingVolume24hr` | `String` | Not usable: trading volume is numeric and this filter is text; a value here makes the request fail. |
| `token` | `IPTFilterBy` | Match on the traded IP Token's fields, e.g. `{ symbol }`. |
| `iptId` | `String` |  |
| `inverted` | `Boolean` |  |

## MoleculeLabActivityFilters

Filters for the `Lab.activity` field. That field is not served by this API (see its description), so these filters have no effect.

| Name | Type | Description |
| --- | --- | --- |
| `byTags` | `[String!]` | Restrict to entries carrying any of these tags. |
| `byCategories` | `[String!]` | Restrict to entries in any of these categories. |
| `byAccessLevels` | `[String!]` | Restrict to entries with any of these access levels (PUBLIC, HOLDERS, ADMIN). |

## OclIdFilter

Lab-id filter requiring exactly one of `eq` or `in`. Invalid or empty operators fail with `VALIDATION_FAILED`.

| Name | Type | Description |
| --- | --- | --- |
| `eq` | `String` | Exact match on a 32-byte 0x-hex lab id; case-insensitive. |
| `in` | `[String!]` | Set of 1-100 matching lab ids, case-insensitive, for fetching several labs' tokens together. |

## ResearchLeadFilterBy

Exact-match filters for `researchLeads`. Every given field must match exactly (case-sensitive); fields are combined with AND. Timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String` |  |
| `createdAt` | `String` |  |
| `updatedAt` | `String` |  |
| `name` | `String` |  |
| `email` | `String` |  |
| `ipnfts` | `String` | Not usable: `ipnfts` is a relation, not a text field; a value here makes the request fail. |

## SearchLabsFilters

Filters for `searchLabs`. Each list restricts hits to entries matching any of its values; lists are combined with AND. Values are forwarded to the search backend as given.

| Name | Type | Description |
| --- | --- | --- |
| `byOclIds` | `[String!]` | Restrict hits to these labs (32-byte 0x-hex OCL ids). |
| `byTags` | `[String!]` | Restrict hits to files and announcements carrying any of these tags. |
| `byCategories` | `[String!]` | Restrict hits to files and announcements in any of these categories. |
| `byAccessLevels` | `[String!]` | Restrict hits to entries with any of these access levels (PUBLIC, HOLDERS, ADMIN). |
| `byKinds` | `[String!]` | Restrict hits to these kinds of entry: "FILE" and/or "ANNOUNCEMENT". Any other value is rejected by the search backend. |

## SignLegalAgreementInput

Input of `signLegalAgreement`. Everything that was passed to `legalAgreementTemplate` to produce the signed document must be echoed here verbatim; the backend regenerates the document and its hash from it.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | OCL identifier (0x..., 32-byte hex) of the lab. |
| `type` | `LegalAgreementType!` | Which legal agreement is being signed. |
| `walletAddress` | `String!` | Signer's wallet address. Must equal the EIP-712 signer, the lab's current LabNft owner, and (on the user auth path) the authenticated wallet. |
| `signature` | `String!` | EIP-712 signature (0x...) over the LegalAgreementAcceptance typed data. |
| `issuedAt` | `AWSTimestamp!` | Echoed VERBATIM from legalAgreementTemplate. Regeneration input + signed field. |
| `signerName` | `String` | Signer identity (natural person) for the SIGNATURES block. Must be echoed verbatim from the `legalAgreementTemplate` call that produced the signature, since it is covered by `contentHash`. |
| `entity` | `String` | Signing entity (if applicable). Echoed verbatim; see signerName. |
| `title` | `String` | Signer title. Echoed verbatim; see signerName. |

## StringFilter

String operator object. All matching is case-insensitive. Provide at least one operator; an empty object is VALIDATION_FAILED.

| Name | Type | Description |
| --- | --- | --- |
| `eq` | `String` | Exact match, case-insensitive. |
| `contains` | `String` | Substring match, not list membership. |

## TokenFilter

Filters for tokens, combined with AND. Malformed values fail with `VALIDATION_FAILED`.

| Name | Type | Description |
| --- | --- | --- |
| `kind` | `TokenKindFilter` | Token classification filter. |
| `chainId` | `IntFilter` | Deployment chain filter. |
| `address` | `StringFilter` | Contract address filter; case-insensitive. |
| `oclId` | `OclIdFilter` | Linked lab id filter. Fails with `VALIDATION_FAILED` on `Lab.tokens`, where the lab is implied. |
| `symbol` | `StringFilter` | Token symbol filter; case-insensitive. |

## TokenKindFilter

Token-kind filter requiring exactly one of `eq` or `in`. Invalid or empty operators fail with `VALIDATION_FAILED`.

| Name | Type | Description |
| --- | --- | --- |
| `eq` | `TokenKind` | Exact match on a token kind. |
| `in` | `[TokenKind!]` | Set of matching token kinds, with 1-100 values. |

## TokenOrderInput

One ordering key for `tokens`.

| Name | Type | Description |
| --- | --- | --- |
| `field` | `TokenOrderField!` | Field to order by. |
| `direction` | `OrderDirection!` | Direction for this ordering key. |

## TokenRelationInput

Canonical or underlying token related to the linked token. Unknown contracts become `EXTERNAL` placeholders without acquiring a lab link.

| Name | Type | Description |
| --- | --- | --- |
| `type` | `TokenRelationType!` | Kind of relationship from the linked token to this one. |
| `chainId` | `Int!` | Indexed chain id; others fail with `VALIDATION_FAILED`. |
| `address` | `String!` | Contract address, accepting any checksum casing. Self-relations fail with `VALIDATION_FAILED`. |
| `name` | `String` | Display name for an unknown token, read from its contract when omitted; ignored for tracked tokens. |
| `symbol` | `String` | Ticker symbol for an unknown token, read from its contract when omitted; ignored for tracked tokens. |
| `decimals` | `Int` | Decimal places for an unknown token, from 0-255; ignored for tracked tokens. When omitted, read from the contract with a fallback of 18. |

## TrlValueFilter

Minimum-TRL filter over the numeric rank of `trlValue` (`trl-N` maps to N, `trl-gt-N` maps to N+1). Labs without an assessment, or with a `pre-trl-*` value, never match. Provide `gte`; an empty object is VALIDATION_FAILED.

| Name | Type | Description |
| --- | --- | --- |
| `gte` | `Int` | Rank between 0 and 10. |

## UnlinkTokenInput

Detach a token from a lab.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | Lab id as 32-byte 0x-hex; the caller must own this lab. |
| `chainId` | `Int!` | Indexed chain id; others fail with `VALIDATION_FAILED`. |
| `address` | `String!` | Contract address, accepting any checksum casing. |

## UpdateLabNftMetadataInput

Patch input for LabNft display metadata. All fields optional: an omitted field is left unchanged and an explicit `null` clears it; `name` is the exception and cannot be cleared.

| Name | Type | Description |
| --- | --- | --- |
| `name` | `String` | New display name, 2-100 characters, from which the lab's shortname is rederived. Clearing it fails with `VALIDATION_FAILED`, and a shortname clash with another lab fails with `CONFLICT`. |
| `description` | `String` | New description of the lab. |
| `image` | `String` | See `generateLabImageUploadUrl` for hosting an image. |
| `externalUrl` | `String` | New external link for the lab. |
| `websiteUrl` | `AWSURL` | Website URL, using absolute HTTPS without credentials, at most 2048 characters. `null` clears it; invalid URLs fail with `VALIDATION_FAILED`. |
| `xUrl` | `AWSURL` | X (Twitter) profile URL; validation and clearing rules match `UpdateLabNftMetadataInput.websiteUrl`. |
| `telegramUrl` | `AWSURL` | Telegram group or channel URL; validation and clearing rules match `UpdateLabNftMetadataInput.websiteUrl`. |
| `governanceUrl` | `AWSURL` | Governance forum or voting URL; validation and clearing rules match `UpdateLabNftMetadataInput.websiteUrl`. |
| `projectFormat` | `String` | Active slug from `LabTaxonomyResult.projectFormats`; `null` clears it. Unknown or retired slugs fail with `VALIDATION_FAILED`, with the rejected slug in error details. |
| `therapeuticFields` | `[String!]` | Active slugs from `LabTaxonomyResult.therapeuticFields`, deduplicated in order; `null` or `[]` clears them. More than 10 unique slugs or unknown or retired slugs fail with `VALIDATION_FAILED`. |

## UserFilterBy

Exact-match filters for `users`. Every given field must match exactly (case-sensitive); fields are combined with AND. Timestamp fields are ISO-8601 strings.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String` |  |
| `createdAt` | `String` |  |
| `updatedAt` | `String` |  |
| `address` | `String` | Match the stored casing exactly. |
| `ipnft` | `String` | Not usable: it does not name a `User` field, so it is ignored. |
| `ipt` | `String` | Not usable: it does not name a `User` field, so it is ignored. |

## ActivitiesResult

Payload of `activities`. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `activities` | `[LabActivityNode!]!` | Activity entries of the requested page across all labs, newest first. |

## Agreement

Legal agreement document attached to an IP-NFT (e.g. the assignment agreement referenced from its metadata).

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Random UUID, regenerated whenever the IP-NFT is re-indexed. Use `contentHash` as the stable identifier. |
| `contentHash` | `String!` | Hash of the agreement document's content, as recorded in the IP-NFT metadata. |
| `mimeType` | `String!` | MIME type recorded in the metadata; defaults to `application/json` when absent. |
| `type` | `String!` | Kind of agreement, as named in the IP-NFT metadata (e.g. the assignment agreement). |
| `url` | `String!` |  |
| `ipnftId` | `String!` |  |
| `encryption` | `AWSJSON` | Encryption details recorded for the document when it is encrypted, as an opaque JSON object; null for plaintext documents. |

## Announcement

Announcement posted to a lab's data room, with its attached files.

| Name | Type | Description |
| --- | --- | --- |
| `lab` | `LabRef!` |  |
| `systemTime` | `AWSDateTime!` | When the announcement was written by the data room backend. |
| `eventTime` | `AWSDateTime!` | When the announcement was posted. |
| `id` | `String!` | Unique within the data room only, so key any cache on the lab as well. |
| `headline` | `String!` |  |
| `body` | `String!` |  |
| `attachments` | `[DataRoomFile!]!` | Files attached to the announcement (at least one; announcements cannot be created without an attachment). |
| `changeBy` | `String!` | Identity of the member who posted the announcement, as `did:ethr:` plus the EIP-55 checksummed wallet address. Older records may hold a bare address or an unverified value. |

## ApiError

Standard error of the Molecule GraphQL API, returned inside a mutation's `*Result` envelope and thrown by queries as a GraphQL error with `code` in `errorType` and the rest in `errorInfo`. Null `error` means success. Some failures arrive instead as plain GraphQL errors with no `code`, which clients must handle too.

| Name | Type | Description |
| --- | --- | --- |
| `code` | `String!` | Stable machine-readable code from the error-code catalogue. The only field clients should branch on; treat an unknown code as a non-retryable failure. |
| `message` | `String!` | Human-readable explanation for developers; never empty. Not part of the contract, so do not parse or match on it. |
| `requestId` | `String!` | Correlation id for the request that failed. Include it in bug reports. |
| `retryable` | `Boolean!` | Whether retrying the same request unchanged can plausibly succeed. Retry with exponential backoff. |
| `details` | `AWSJSON` | Structured context for the failure. Keys: `field` (offending input), `reason` (second-level code), `hint` (next step), `docs` (URL); unknown keys may appear and must be ignored. |

## Chain

Blockchain on which IP-NFTs and IP Tokens are deployed.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `Int!` | Molecule's record id for the chain, and the value `Market.chainId` references. |
| `createdAt` | `AWSDateTime!` | When Molecule first indexed the chain. |
| `updatedAt` | `AWSDateTime!` | When Molecule last refreshed the chain record. |
| `name` | `String!` |  |
| `chainId` | `Int!` | EVM chain id (e.g. 1 for Ethereum mainnet, 8453 for Base). |
| `logoUrl` | `String!` |  |
| `markets` | `[Market!]!` | Markets on this chain; the arguments are accepted but ignored and the full relation is returned. Whether this field resolves data is unverified, so prefer `markets(filterBy: { chainId })`. |

**`markets` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortBy` | `MarketSortBy` | Accepted but ignored on this nested field. |
| `filterBy` | `MarketFilterBy` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

## ConnectionPageInfo

Cursor-based (Relay-style) pagination info for connections. The page-numbered `PageInfo` serves `labs`, `searchLabs` and the activity feeds.

| Name | Type | Description |
| --- | --- | --- |
| `hasNextPage` | `Boolean!` | Whether more items exist after this page. When paginating backward it reflects whether a `before` cursor was supplied. |
| `hasPreviousPage` | `Boolean!` | Whether more items exist before this page. When paginating forward it reflects whether an `after` cursor was supplied. |
| `startCursor` | `String` | Cursor of the first edge; null on an empty page. |
| `endCursor` | `String` | Cursor of the last edge; null on an empty page. |

## CreateAnnouncementResult

Result of creating an announcement.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## CreateLabResult

Result of creating a lab.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Human-readable message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |
| `lab` | `LabRef` | Created lab details if successful (minimal fields only). |

## DataRoom

Lab's data room: the container for its files and announcements.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` |  |
| `alias` | `String!` |  |
| `status` | `String` | Free-text status reported by the backend, e.g. `In Progress` or `Completed`. The value set is not fixed and may change. |
| `telegramChatId` | `String` | Always null; no Telegram integration is served by this API. |
| `description` | `String` |  |
| `createdAt` | `AWSDateTime!` |  |
| `lastUpdatedAt` | `AWSDateTime` |  |
| `owner` | `DataRoomOwner!` |  |
| `keywords` | `[String!]` | Empty array rather than null when the data room has no keywords. |
| `files` | `[DataRoomFile!]` | Files in the data room. Populated by `labWithDataRoomAndFiles`; null when the files could not be listed. |

## DataRoomEntry

Data room file as it appeared in an activity-feed event: the version the event refers to, not necessarily the current one.

| Name | Type | Description |
| --- | --- | --- |
| `ref` | `String!` | Stable reference (dataset id) of the file across versions. Use it with `finishCreateOrUpdateFile(ref)` to upload a new version and `updateFileMetadata(ref)` to edit metadata. |
| `path` | `String!` |  |
| `tags` | `[String!]` | Tags on this version. Null when none. |
| `description` | `String` | Description of this version. Null when none. |
| `version` | `Int!` | Version number of the file this event refers to (1 for the first upload). |
| `accessLevel` | `String!` | Access level of this version: `PUBLIC`, `ADMIN` (confidential, encrypted; see `DataRoomAccessLevel`) or the unenforced `HOLDERS`. |
| `eventTime` | `AWSDateTime!` | When this version took effect. |
| `systemTime` | `AWSDateTime!` | When this version was written by the data room backend. |
| `changeBy` | `String!` | Identity of the member who made the change, as `did:ethr:` plus the EIP-55 checksummed wallet address. Older records may hold a bare address or an unverified value. |
| `categories` | `[String!]` | Categories of this version. Null when none. |
| `contentType` | `String!` |  |
| `contentHash` | `String!` |  |
| `contentText` | `String` | Searchable text extracted from or supplied for this version. Null when none. |
| `lab` | `LabRef!` |  |
| `encryptionMetadata` | `EncryptionMetadata` | Encryption metadata when the file is encrypted (KMS, BLS or a legacy ciphertext); null for plaintext files. |

## DataRoomFile

File in a lab's data room, with its current version's metadata and, when the caller may read it, a time-limited download URL.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` |  |
| `did` | `String!` |  |
| `path` | `String!` |  |
| `version` | `Int` | Version number of the file's current revision. Null when no version is reported. |
| `contentType` | `String!` | Defaults to `application/octet-stream` when the backend reports none. |
| `accessLevel` | `DataRoomAccessLevel!` | Access level of the file. `ADMIN` files are encrypted: read `encryptionMetadata` and unwrap the key with `decryptDataKey`. |
| `createdAt` | `AWSDateTime!` | When the file's data room entry took effect. On `dataRoomFile` it is the current version's time, the same as `updatedAt`. |
| `updatedAt` | `AWSDateTime` | When this version of the file took effect. |
| `name` | `String` | Falls back through the dataset name, the file description, and finally the last path segment, so it may repeat `description`. |
| `createdBy` | `String` | Identity of the member who last changed the file, not its original creator; the same value as `changeBy`. Older records may hold a bare address or an unverified value. |
| `contentHash` | `String` |  |
| `downloadUrl` | `String` | Pre-signed URL that expires at `downloadUrlExpiry`. Send every entry of `downloadHeaders` with the request. |
| `downloadHeaders` | `[Header!]` | Headers that must be sent with the `downloadUrl` request; the download fails without them. |
| `downloadUrlExpiry` | `AWSDateTime` | When `downloadUrl` stops working. Request the file again for a fresh URL. |
| `encryptionMetadata` | `EncryptionMetadata` | Encryption metadata if the file is encrypted (KMS, BLS, or legacy ciphertext). |
| `description` | `String` |  |
| `tags` | `[String!]` | Empty array rather than null when the file has no tags. |
| `categories` | `[String!]` | Empty array rather than null when the file has no categories. |
| `contentText` | `String` |  |
| `changeBy` | `String` | Identity of the member who last changed the file, as `did:ethr:` plus the EIP-55 checksummed wallet address. Older records may hold a bare address or an unverified value. |

## DataRoomFileSearchEntry

Lightweight data room entry for search results.

| Name | Type | Description |
| --- | --- | --- |
| `lab` | `LabRef!` | Lab the file belongs to (search-result hydration tier; see `LabRef`). |
| `path` | `String!` | Path of the file within the data room. |
| `ref` | `String!` | Stable reference (dataset id) of the file across versions. |
| `systemTime` | `AWSDateTime!` | When the matching version was written by the data room backend. |
| `eventTime` | `AWSDateTime!` | When the matching version took effect. |
| `file` | `DataRoomFile!` | File's current metadata. |

## DataRoomOwner

Owner of a data room. Data room IDs follow the format `<contract_address>_<token_id>` (e.g. `0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_9`), which keeps them globally unique across contracts.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` |  |
| `accountName` | `String!` |  |
| `displayName` | `String!` | Display name reported by the data room backend. |

## DecryptDataKeyResult

Result of decrypting a data encryption key.

| Name | Type | Description |
| --- | --- | --- |
| `plaintextDEK` | `String` | Base64-encoded data encryption key for decrypting the file client-side. Null on failure. |
| `iv` | `String` | Base64-encoded initialization vector stored with the file. Null on failure. |
| `message` | `String` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## DeleteDataRoomFileResult

Result type for file deletion operations.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String` |  |
| `filePath` | `String` |  |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## DidLinkStatus

Snapshot of DID-linking state for an OCL. `status` is null until the first linking attempt reaches PENDING, and `linkedDidCount` counts the DIDs recorded as active onchain, so a count with a non-terminal `status` means the terminal transition has not landed yet.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte 0x-hex OCL id of the lab. |
| `status` | `DidLinkingStatus` | Current state of the linking state machine; null before the first attempt. |
| `userOpHash` | `String` | Hash of the user operation submitted for the most recent attempt; null until one has been submitted. |
| `txHash` | `String` | Hash of the transaction that included the user operation; null until it was mined. |
| `accountDid` | `String` | DID of the lab's smart account (the ERC-6551 account); null until known. |
| `dataRoomDid` | `String` | DID of the lab's data room; null until known. |
| `linkedDidCount` | `Int!` | Number of DIDs recorded as linked onchain for this lab (2 when both the account and data room DIDs are linked). |
| `attempts` | `Int!` | Number of linking attempts made so far (0 before the first). |
| `updatedAt` | `AWSDateTime` | When the linking state last changed; null before the first attempt. |

## DidLinkStatusResult

Payload of the public getDidLinkStatus query. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String` | Human-readable status message. |
| `didLinkStatus` | `DidLinkStatus` | DID-linking snapshot for the lab. Always present for a well-formed `oclId`, with an unknown lab or one having no linking record reporting null status, hashes and DIDs and zero counts. |

## EncryptionMetadata

Encryption metadata for encrypted files. Supports multiple encryption systems: - Legacy (pre-cutover ciphertexts): identified by absent/null encryptionSystem - KMS envelope encryption: encryptionSystem = "kms" - BLS threshold encryption: encryptionSystem = "bls" (future).

| Name | Type | Description |
| --- | --- | --- |
| `encryptionSystem` | `String` | Encryption system identifier, one of "kms", "bls", or null. Null or absent indicates a legacy ciphertext written before the onchain-verified envelope cutover. |
| `accessControlConditions` | `AWSJSON!` | Access control conditions the file was encrypted with, shaped as documented on `EncryptionMetadataInput.accessControlConditions`. `decryptDataKey` evaluates them onchain against the caller's wallet. |
| `encryptedBy` | `String!` | Wallet address that performed the encryption. |
| `encryptedAt` | `AWSDateTime!` | ISO 8601 timestamp when encryption was performed. |
| `encryptedDek` | `String` | KMS/BLS: Base64-encoded encrypted data encryption key. |
| `iv` | `String` | KMS/BLS: Base64-encoded initialization vector used for AES-GCM encryption. |
| `contentHash` | `String` | KMS/BLS: Hash of the encrypted content for integrity verification. |
| `keyId` | `String` | BLS: Key identifier for the BLS key used. |
| `dataToEncryptHash` | `String` | Legacy: Hash of the original plaintext data from the legacy encryption client. |
| `chain` | `String` | Legacy: Blockchain network the legacy conditions were authored against (e.g. 'ethereum', 'base'). |
| `litSdkVersion` | `String` | Legacy: SDK version that produced the legacy ciphertext. |
| `litNetwork` | `String` | Legacy: Network identifier from the legacy client. |
| `templateName` | `String` | Legacy: Template name used for access control. |
| `contractVersion` | `String` | Legacy: Contract version for access control. |

## EvmTokenizationError

Failure detail of a tokenization-service operation, present exactly when the result's `isSuccess` is false. Codes: INVALID_INPUT, INVALID_METADATA, MISSING_METADATA_FIELD, METADATA_UPLOAD_FAILED, IMAGE_UPLOAD_FAILED, UNSUPPORTED_IMAGE_TYPE, INVALID_TERMS_SIGNATURE, SIGNOFF_FAILED, TERMS_MESSAGE_FAILED, INTERNAL_ERROR.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String` | Human-readable explanation for developers. Not part of the contract, so do not parse or match on it. |
| `code` | `String` | Machine-readable failure code, from the set listed on `EvmTokenizationError`. |
| `retryable` | `Boolean!` | Whether retrying the same request unchanged can plausibly succeed. True only for transient upload, storage or signing failures, never for validation failures. |
| `details` | `AWSJSON` | Structured context. Currently always null for this service. |

## FileCategoriesAndTagsResult

Payload of `fileCategoriesAndTags`. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `data` | `[FileCategory!]` | Every valid category with its tags. |

## FileCategory

File category and tags valid within it, as configured by Molecule.

| Name | Type | Description |
| --- | --- | --- |
| `name` | `String!` |  |
| `tags` | `[String!]!` |  |

## FinishFileUploadResult

Result of finishing a file upload.

| Name | Type | Description |
| --- | --- | --- |
| `datasetId` | `ID` | Dataset id of the file; its stable `ref` for later versions and metadata updates. Null on failure. |
| `contentHash` | `String` | Hash of the stored file content. Null on failure. |
| `version` | `Int` | Version number of the file after this upload (1 for a new file). Null on failure. |
| `newHead` | `String` | Head hash of the data room after the commit, usable as `expectedHead` for optimistic concurrency in later operations. Null on failure. |
| `message` | `String` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## GenerateAssignmentAgreementResult

Result of generateAssignmentAgreement. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `agreementCid` | `String` | IPFS CID of the generated agreement document. Null on failure. |
| `agreementUrl` | `String` | HTTPS gateway URL of the agreement document. Null on failure. |
| `agreementContentHash` | `String` | SHA-256 of the agreement document as 64 lowercase hex characters without a `0x` prefix, referenced as an agreement `content_hash` in the IPNFT metadata. Null on failure. |
| `agreementUri` | `String` | IPFS URI of the agreement (`ipfs://<agreementCid>`). Null on failure. |
| `agreementType` | `AssignmentAgreementType` | Which agreement was generated, derived from the IPNFT id. Null on failure. |
| `generatedAt` | `AWSDateTime` | When the agreement document was generated. Null on failure. |
| `isSuccess` | `Boolean!` | True when the agreement was generated and stored. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |

## GenerateDataEncryptionKeyResult

Result of generating a standalone data encryption key.

| Name | Type | Description |
| --- | --- | --- |
| `plaintextDEK` | `String` | Base64-encoded data encryption key for encrypting content client-side. Null on failure. |
| `encryptedDek` | `String` | Base64-encoded wrapped key to store alongside the ciphertext. Null on failure. |
| `encryptionSystem` | `String` | Encryption system that produced the key; always `kms`. Null on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## GenerateIptMembershipAgreementResult

Result of generateIptMembershipAgreement. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `agreementCid` | `String` | IPFS CID of the generated agreement document. Null on failure. |
| `agreementUrl` | `String` | HTTPS gateway URL of the agreement document. Null on failure. |
| `agreementContentHash` | `String` | SHA-256 of the agreement document as 64 lowercase hex characters without a `0x` prefix. Null on failure. |
| `agreementUri` | `String` | IPFS URI of the agreement (`ipfs://<agreementCid>`); pass its CID to `getIptTermsMessage`. Null on failure. |
| `agreementType` | `IptAgreementType` | Always IPT_MEMBERSHIP on success. Null on failure. |
| `generatedAt` | `AWSDateTime` | When the agreement document was generated. Null on failure. |
| `isSuccess` | `Boolean!` | True when the agreement was generated and stored. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |

## GenerateLabImageUploadUrlResult

Result of `generateLabImageUploadUrl`. The presigned PUT URL is single-use and expires per `expiresAt`, and the lab's `image` is updated asynchronously once the uploaded object has been processed.

| Name | Type | Description |
| --- | --- | --- |
| `uploadUrl` | `String` | Pre-signed HTTPS URL for a single PUT of the image, sent with the same Content-Type the URL was generated for. Null on failure. |
| `key` | `String` | Storage key of the image (`<oclId>/<uuid>.<extension>`). Null on failure. |
| `expiresAt` | `AWSDateTime` | When `uploadUrl` stops accepting uploads (15 minutes after issue). Null on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## GenerateOclMembershipAgreementResult

Result of generateOclMembershipAgreement. The document is stored in the public OCL agreements bucket rather than on IPFS, so it is addressed by a storage key and URL instead of a CID. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `agreementKey` | `String` | Storage key of the agreement document, of the form `<oclId>/agreements/<uuid>.json`, to pass to `getOclTermsMessage`. Null on failure. |
| `agreementUrl` | `String` | Public HTTPS URL of the agreement document: the agreements base URL followed by `agreementKey`. Null on failure. |
| `agreementContentHash` | `String` | SHA-256 of the agreement document as `0x` plus 64 lowercase hex characters, the onchain `contentHash` format. Deterministic for a given oclId, symbol and title, and null on failure. |
| `agreementType` | `OclAgreementType` | Always OCL_MEMBERSHIP on success. Null on failure. |
| `generatedAt` | `AWSDateTime` | When this response was produced, which is informational only and not part of the deterministic stored document. Null on failure. |
| `isSuccess` | `Boolean!` | True when the agreement was generated and stored. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |

## GeneratePresignedUploadUrlResult

Result of generateImageUploadUrl. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `uploadUrl` | `String` | Pre-signed HTTPS URL for a single PUT of the image; the request's Content-Type must equal the `contentType` the URL was generated for. Null on failure. |
| `key` | `String` | Storage key of the image (`ipnft-<ipnftId>/<filename>`), to pass as `imageKey` to `uploadMetadataWithImageKey` once the upload completes. Null on failure. |
| `expiresAt` | `AWSDateTime` | When `uploadUrl` stops accepting uploads. Null on failure. |
| `isSuccess` | `Boolean!` | True when the URL was generated. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |

## GetTermsMessageResult

Terms message to be signed by a wallet, shared by the IPNFT, IPT and OCL terms queries. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String` | Exact text the wallet must sign (`personal_sign` / EIP-191). Empty string on failure. |
| `digest` | `String` | keccak256 of `message` as `0x`-hex. Empty string on failure. |
| `isSuccess` | `Boolean!` | True when the message was produced. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the query failed. |

## Header

One HTTP header to send with a file download or upload request.

| Name | Type | Description |
| --- | --- | --- |
| `key` | `String!` |  |
| `value` | `String!` |  |

## InitiateFileUploadResult

Result of initiating a file upload.

| Name | Type | Description |
| --- | --- | --- |
| `datasetId` | `ID` | Dataset id of the file. Currently always null at this step: the id is assigned when `finishCreateOrUpdateFile` completes. |
| `uploadToken` | `String` | Opaque token identifying this upload; pass it to `finishCreateOrUpdateFile`. Null on failure. |
| `uploadUrl` | `String` | Pre-signed URL to send the file bytes to. Null on failure. |
| `uploadUrlExpiry` | `AWSDateTime` | When `uploadUrl` stops accepting uploads. Currently always null; the URL is short-lived, so upload promptly. |
| `method` | `String` | HTTP method to use against `uploadUrl` (normally "PUT"). Null on failure. |
| `headers` | `[Header!]` | Headers that must be sent with the upload request. Null on failure. |
| `expectedHeadHash` | `String` | Currently always null; not used by the upload protocol. |
| `useMultipart` | `Boolean` | Whether the upload must be performed as a multipart upload. Null on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## IPNFT

Onchain token representing the legal rights to a research project's IP and data. The project fields are a read-only snapshot of the metadata the token was minted with, not editable through this API.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Onchain token id as a decimal string (e.g. `37`). |
| `createdAt` | `AWSDateTime!` | When Molecule first indexed the IP-NFT. |
| `updatedAt` | `AWSDateTime!` | When Molecule last refreshed the record. |
| `mintedAt` | `AWSDateTime` | When the token was minted. Null when not yet known. |
| `chainId` | `Int!` | EVM chain id the token lives on. |
| `owner` | `User!` | Current owner. |
| `userId` | `String!` | Id of the current owner (`owner.id`). |
| `originalOwner` | `String!` | Wallet address of the original minter. |
| `ipt` | `IPT` | **Deprecated.** Use `Lab.tokens`. Removed after 2027-01-01. IP Token minted against this IP-NFT, if it has been tokenized. |
| `tokenUri` | `String!` | URI of the token metadata document, typically an `ipfs://` URI. |
| `symbol` | `String!` | Ticker taken from the mint event; `initialSymbol` is what the metadata proposed. |
| `name` | `String!` |  |
| `image` | `String!` |  |
| `description` | `String!` |  |
| `externalUrl` | `String!` |  |
| `initialSymbol` | `String!` | Symbol proposed at minting. |
| `organization` | `String!` |  |
| `topic` | `String!` |  |
| `trlValue` | `String` | Technology Readiness Level of the project, assessed by Molecule with AI assistance rather than derived onchain. Null when no assessment exists, and the rest of the IP-NFT is still returned. |
| `trlRationale` | `String` | Human-readable explanation of `trlValue`. Null when no assessment exists. |
| `fundingAmountCurrency` | `String!` | Funding currency code (e.g. "USD"). |
| `fundingAmountValue` | `String!` | Funding amount as an integer in the currency's smallest unit, as a decimal string (divide by 10^`fundingAmountDecimals`). |
| `fundingAmountDecimals` | `Int!` | Number of decimals of `fundingAmountValue`. |
| `fundingAmountCurrencyType` | `String!` | Kind of currency code in `fundingAmountCurrency` (e.g. "ISO4217"). |
| `researchLead` | `ResearchLead!` | Project's research lead. |
| `researchLeadId` | `String!` | Email address of the research lead, which is also `ResearchLead.id`. |
| `agreements` | `[Agreement!]!` | Legal agreement documents attached to the IP-NFT. The arguments are accepted but ignored: the full list is returned. |
| `schemaVersion` | `String!` | Version of the metadata schema the token was minted with (e.g. "1.0.0"). |
| `oclId` | `String` | Canonical 32-byte oclId (lowercase 0x-hex) of the lab associated with this IPNFT, if one exists. Null when the IPNFT has no linked lab. |

**`agreements` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortBy` | `AgreementSortBy` | Accepted but ignored on this nested field. |
| `filterBy` | `AgreementFilterBy` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

## IPT

IP Token (IPT): the ERC-20 token issued against an IP-NFT, whose holders are governed by the token's membership agreement.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | ERC-20 contract address, not a numeric id. For a token issued against an IP-NFT this is its L1 contract; a token with no IP-NFT is identified by its own contract address. |
| `createdAt` | `AWSDateTime!` | When Molecule first indexed the token. |
| `updatedAt` | `AWSDateTime!` | When Molecule last refreshed the record. |
| `ipnft` | `IPNFT` | IP-NFT the token was issued against. Null for tokens with no IP-NFT. |
| `ipnftId` | `String` | Id of the parent IP-NFT (`IPNFT.id`). Null for tokens with no IP-NFT. |
| `l2TokenAddress` | `String` | Deterministic ERC-20 address on the default L2, derived at tokenization from the L1 contract, name, symbol and decimals. Equal to `id` when there is no IP-NFT, and not a sign of bridging. |
| `holderCount` | `Int` | Null until the holder indexer has run for this token. |
| `mintedAt` | `AWSDateTime` | Null until an indexer backfills it; it is not set at tokenization. |
| `markets` | `[Market!]!` | Markets trading this token. The arguments are accepted but ignored: the full list is returned. |
| `name` | `String!` |  |
| `symbol` | `String!` |  |
| `decimals` | `Int!` | Always 18 for a token issued against an IP-NFT; a token with no IP-NFT reports its own value. |
| `agreementCid` | `String` | IPFS CID of the token's membership agreement. Null when none. |
| `agreementMimeType` | `String!` | MIME type of the membership agreement (e.g. "application/json"). |
| `originalOwner` | `User!` |  |
| `originalOwnerId` | `String!` | Wallet address of the original owner, which is also `User.id`. |
| `image` | `String!` | Inherited from the parent IP-NFT image. For a token with no IP-NFT it is that token's own logo, or an empty string when it has none. |
| `links` | `[String!]!` | Related links (URLs). The arguments are accepted but ignored: the full list is returned. |
| `capped` | `Boolean!` | Whether token issuance is capped. |
| `circulatingSupply` | `String` | Circulating supply in the token's smallest unit, as a decimal string. Null when not known. |
| `totalIssued` | `String!` | Total issued supply in the token's smallest unit, as a decimal string. |

**`markets` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortBy` | `MarketSortBy` | Accepted but ignored on this nested field. |
| `filterBy` | `MarketFilterBy` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

**`links` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

## Lab

Single onchain lab with its data room, as returned by `labWithDataRoomAndFiles`. Lightweight references to labs elsewhere use `LabRef`.

| Name | Type | Description |
| --- | --- | --- |
| `systemTime` | `AWSDateTime!` | When the lab record was last written by the data room backend. Reflects backend processing (including migrations), not the lab's creation; use `mintedAt` for that. |
| `eventTime` | `AWSDateTime!` | When the lab record's current state took effect in the data room backend. |
| `mintedAt` | `AWSDateTime` | LabNft mint block timestamp from the OclIdentityCreated event. |
| `oclId` | `String!` | Canonical 32-byte oclId (lowercase 0x-hex); primary identifier. |
| `shortname` | `String` | Human-readable slug used as the URL segment (e.g. "vita-fast"), derived from `name` and re-derived on rename, at which point the previous slug stops resolving. Persist `oclId`, not the slug. |
| `labAccountAddress` | `String!` | ERC-6551 token-bound smart-account address for this lab; the deterministic account address tied to the LabNft `tokenId`. |
| `labNftTokenId` | `String!` | LabNft tokenId as a decimal string. Replaces the legacy `ipnftTokenId`. |
| `account` | `LabAccount!` | Owning account of this lab's data room. |
| `dataRoom` | `DataRoom!` | Lab's data room: metadata plus, when `files` is selected, its files. |
| `ipnft` | `IPNFT` | Legacy IPNFT this lab was migrated from, fully hydrated (owner, researchLead, agreements, ipt) for the single-lab query. Null when the lab has no linked IPNFT. |
| `announcements` | `[Announcement!]!` | Not served by this API: nothing populates it, so selecting it nulls the whole `Lab` (the field is non-null). Use `labActivity(oclId, filter: ANNOUNCEMENT)` for a lab's announcements instead. |
| `activity` | `LabActivityNode!` | Not served by this API: nothing populates it, so selecting it nulls the whole `Lab` (the field is non-null). Use the top-level `labActivity(oclId)` query instead. |
| `name` | `String` | LabNft display name. Defaults to "Lab #<tokenId>" until the owner sets a custom value via `updateLabNftMetadata`. |
| `description` | `String` | Owner-editable description (see `updateLabNftMetadata`). Null when unset. |
| `image` | `String` | LabNft image URL; owner-editable. Null until the owner sets a custom image. |
| `externalUrl` | `String` | External link surfaced by the LabNft tokenURI metadata. |
| `hasDataRoom` | `Boolean!` | Always true on this type, which resolves only labs whose data room exists. Use the `lab` query to tell a lab still being set up from one that is ready. |
| `members` | `[LabMember!]` | Active members of the lab, the same data as the top-level `listLabMembers` query and public with no authentication required. Null only on lookup failure. |
| `tokens` | `TokenConnection` | Tokens linked to this lab, using `tokens` pagination. Invalid arguments, including `filter.oclId`, fail with `VALIDATION_FAILED`. |
| `legalAgreementStatus` | `LegalAgreementStatusResult` | Signed status of a legal agreement for this lab, inlined so a client can fetch it per lab instead of calling `legalAgreementStatus` N times. Public, and null only when the lab cannot be resolved. |
| `trlValue` | `String` | Technology Readiness Level of the lab, from Molecule's AI-assisted assessment rather than onchain. Null when no assessment exists. |
| `trlRationale` | `String` | Human-readable explanation for the assigned `trlValue`. Null when no assessment exists. |
| `trlLastUpdated` | `AWSDateTime` | When `trlValue` last changed. Null when no assessment exists. |
| `weightedScore` | `Float` | Weighted project score from Molecule's AI-assisted assessment, summing all weighted criterion scores, where 5.0 is best and 1.0 worst. Not derived onchain, and null when no assessment exists. |
| `scoreInterpretation` | `String` | Human-readable summary of the overall assessment behind `weightedScore`. Null when no assessment exists. |
| `criterionScores` | `[AWSJSON!]` | Per-criterion breakdown behind `weightedScore`, each entry shaped like `{ "criterion": String, "score": Number }` with further keys possible over time. Null when no assessment exists. |
| `scoredAt` | `AWSDateTime` | When the scoring behind `weightedScore` was last computed. Null when no assessment exists. |
| `todos` | `[AWSJSON!]` | Action items for the lab from Molecule's assessment (AI-assisted), each a JSON object shaped like `{ "todo": String, "completed": Boolean }`; further keys may be added over time. Null when none exist. |
| `isVerified` | `Boolean` | Whether Molecule has verified this lab. Null when no verification decision has been recorded. |
| `websiteUrl` | `AWSURL` | Lab website URL; null when unset or metadata is unavailable. |
| `xUrl` | `AWSURL` | Lab X (Twitter) profile URL; null when unset or metadata is unavailable. |
| `telegramUrl` | `AWSURL` | Lab Telegram group or channel URL; null when unset or metadata is unavailable. |
| `governanceUrl` | `AWSURL` | Lab governance forum or voting URL; null when unset or metadata is unavailable. |
| `projectFormat` | `String` | Project format slug; active choices are in `LabTaxonomyResult.projectFormats`, but stored retired slugs remain readable. Null when unset or metadata is unavailable. |
| `therapeuticFields` | `[String!]` | Therapeutic-field slugs in their saved order; active choices are in `LabTaxonomyResult.therapeuticFields`. Empty when unset, null when metadata is unavailable. |
| `isRwaEnabled` | `Boolean` | Whether a `LOCKED` token is linked to this lab or a locked token's `LOCKS` relation targets one of its tokens; derived on each read. Null when the lab's metadata could not be read, never false. |

**`activity` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `page` | `Int` | Accepted but ignored; see the field description. |
| `perPage` | `Int` | Accepted but ignored; see the field description. |
| `filters` | `MoleculeLabActivityFilters` | Accepted but ignored; see the field description. |

**`tokens` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `first` | `Int` | Forward page size, from 1-100; mutually exclusive with `last`. |
| `after` | `String` | Opaque cursor after which to continue; requires `first`. |
| `last` | `Int` | Backward page size, from 1-100; mutually exclusive with `first`. |
| `before` | `String` | Opaque cursor before which to continue; requires `last`. |
| `filter` | `TokenFilter` | Typed filters, combined with AND; `oclId` is implied by the parent lab. |
| `orderBy` | `[TokenOrderInput!]` | Up to 3 ordering keys. Default `CREATED_AT DESC`. |

**`legalAgreementStatus` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `type` | `LegalAgreementType!` | Which legal agreement to report on. |

## LabAccount

Owning account of a lab's data room in the data room backend, one per lab. Not an identifier for the lab itself: use `oclId` or `shortname`.

| Name | Type | Description |
| --- | --- | --- |
| `accountName` | `String!` | Empty string when the backend reports no owning account. |

## LabActivityResult

Payload of `labActivity`. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `nodes` | `[LabActivityNode!]!` | Activity entries of the requested page, newest first. |
| `pageInfo` | `PageInfo!` | Pagination information for the requested page. |

## LabEventAnnouncement

Announcement was posted to a lab.

| Name | Type | Description |
| --- | --- | --- |
| `lab` | `LabRef!` | Lab the announcement was posted to. |
| `announcement` | `Announcement!` |  |

## LabEventFileAdded

New file was added to a data room.

| Name | Type | Description |
| --- | --- | --- |
| `lab` | `LabRef!` | Lab whose data room changed. |
| `entry` | `DataRoomEntry!` |  |

## LabEventFileRemoved

Data room file was removed.

| Name | Type | Description |
| --- | --- | --- |
| `lab` | `LabRef!` | Lab whose data room changed. |
| `entry` | `DataRoomEntry!` | Removed file, as of its last version. |

## LabEventFileUpdated

New version of an existing data room file was uploaded.

| Name | Type | Description |
| --- | --- | --- |
| `lab` | `LabRef!` | Lab whose data room changed. |
| `entry` | `DataRoomEntry!` |  |

## LabMember

Single lab member entry, derived from the indexed onchain role state.

| Name | Type | Description |
| --- | --- | --- |
| `walletAddress` | `String!` | Lowercased wallet address of the member. |
| `role` | `LabMemberRole!` | Effective role on the lab. |
| `source` | `LabMemberSource!` | Source row that authoritatively defines this membership. |
| `expiry` | `String` | Unix-seconds expiry as a decimal string, encoded as a string because BigInt unix-seconds is unsafe as a JSON Int. Null means the grant is permanent. |
| `isAgent` | `Boolean!` | True if the member is an agent identity (separate from human auth, surfaced for UI but not used for authorization). |
| `grantedAt` | `String!` | ISO-8601 timestamp the row was first persisted. |

## LabRef

Lightweight lab reference, returned wherever a lab is referenced rather than fully expanded. How much is hydrated depends on the query, so each field documents where it is null. Beyond the identifier fields a null can mean the source was briefly unavailable, or a value served from a mirror that lags by a few hours.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | Canonical 32-byte oclId (lowercase 0x-hex); primary identifier. |
| `mintedAt` | `AWSDateTime` | LabNft mint block timestamp from the OclIdentityCreated event. |
| `shortname` | `String` | Human-readable slug (see `Lab.shortname`), seeded as "lab-<tokenId>" at index time and re-derived on rename. Null on `searchLabs` and for unbackfilled legacy labs; replaces the legacy token `symbol`. |
| `ipnftId` | `String` | Linked legacy IPNFT tokenId. Null on `searchLabs` and when no IPNFT is linked; `ipnft` carries the full object on the queries that hydrate it. |
| `hasDataRoom` | `Boolean` | Whether the lab's data room exists; it must be created before files or announcements can be added. Null on `searchLabs`, which does not mean false. |
| `ipnft` | `IPNFT` | Legacy IPNFT this lab was migrated from, hydrated when selected. Null when no IPNFT is linked, and on the activity feeds and `searchLabs`, which expose `ipnftId` for a follow-up lookup. |
| `labAccountAddress` | `String!` | ERC-6551 token-bound smart-account address for this lab; deterministic from the LabNft tokenId via the OnChainLabFactory. Replaces the misleadingly-named `ipnftAddress`. |
| `labNftTokenId` | `String!` | LabNft tokenId as a decimal string. Replaces the legacy `ipnftTokenId`. |
| `latestContributionAt` | `AWSDateTime` | Timestamp of the lab's latest data room activity, lagging a few hours on `labsConnection` and `lab`. Null on `labs` unless selected, on other queries, when there is none, or on lookup failure. |
| `name` | `String` | LabNft display name (defaults to "Lab #<tokenId>"). Null on `searchLabs`. |
| `description` | `String` | Owner-editable description (see `updateLabNftMetadata`). Null when unset and on `searchLabs`. |
| `image` | `String` | LabNft image URL, null until the owner sets a custom image. Null on `searchLabs`. |
| `externalUrl` | `String` | External link surfaced by the LabNft tokenURI metadata. Null on `searchLabs`. |
| `members` | `[LabMember!]` | Active members of the lab, mirroring the public `listLabMembers` query. Null only on lookup failure. |
| `legalAgreementStatus` | `LegalAgreementStatusResult` | Signed status of a legal agreement for this lab, inlined so a client can fetch it per lab instead of calling `legalAgreementStatus` N times. Public, and null only when the lab cannot be resolved. |
| `trlValue` | `String` | Technology Readiness Level from Molecule's AI-assisted assessment rather than onchain. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `trlRationale` | `String` | Human-readable explanation for the assigned `trlValue`. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `trlLastUpdated` | `AWSDateTime` | When `trlValue` last changed. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `isVerified` | `Boolean` | Whether Molecule has verified this lab. Null when no verification decision has been recorded, and on the activity feeds and `searchLabs`. |
| `weightedScore` | `Float` | Weighted score from Molecule's assessment, summing all weighted criterion scores, where 5.0 is best and 1.0 worst. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `scoreInterpretation` | `String` | Human-readable summary of the overall assessment behind `weightedScore`. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `criterionScores` | `[AWSJSON!]` | Per-criterion breakdown behind `weightedScore`, each entry shaped like `{ "criterion": String, "score": Number }` with further keys possible over time. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `scoredAt` | `AWSDateTime` | When the scoring behind `weightedScore` was last computed. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `todos` | `[AWSJSON!]` | Action items for the lab from Molecule's assessment (AI-assisted), each a JSON object shaped like `{ "todo": String, "completed": Boolean }`; further keys may be added over time. Null when none exist, and on the activity feeds and `searchLabs`. |
| `websiteUrl` | `AWSURL` | Lab website URL; null when unset or not hydrated, including on `searchLabs`. |
| `xUrl` | `AWSURL` | Lab X (Twitter) profile URL; null when unset or not hydrated, including on `searchLabs`. |
| `telegramUrl` | `AWSURL` | Lab Telegram group or channel URL; null when unset or not hydrated, including on `searchLabs`. |
| `governanceUrl` | `AWSURL` | Lab governance forum or voting URL; null when unset or not hydrated, including on `searchLabs`. |
| `projectFormat` | `String` | Project format slug, including stored retired slugs; active choices are in `LabTaxonomyResult.projectFormats`. Null when unset or not hydrated, including on `searchLabs`. |
| `therapeuticFields` | `[String!]` | Therapeutic-field slugs in their saved order; active choices are in `LabTaxonomyResult.therapeuticFields`. Empty when unset, null when not hydrated, including on `searchLabs`. |
| `isRwaEnabled` | `Boolean` | Whether the lab is RWA-enabled, as defined by `Lab.isRwaEnabled`. Null when metadata is not hydrated, including on `searchLabs`. |

**`legalAgreementStatus` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `type` | `LegalAgreementType!` | Which legal agreement to report on. |

## LabRefConnection

Relay-style connection over labs.

| Name | Type | Description |
| --- | --- | --- |
| `edges` | `[LabRefEdge!]!` | Page's labs, each with its cursor, in the requested order. |
| `nodes` | `[LabRef!]!` | Convenience list of nodes (same order as edges), for callers that don't need per-item cursors. |
| `pageInfo` | `ConnectionPageInfo!` | Cursors and has-more flags for continuing from this page. |
| `totalCount` | `Int` | Total labs matching the filter, ignoring pagination. Computed only when selected and the most expensive part of the query, so omit it when it is not needed. |

## LabRefEdge

Edge in the labs connection: one lab plus its opaque position cursor.

| Name | Type | Description |
| --- | --- | --- |
| `node` | `LabRef!` | Lab at this position. |
| `cursor` | `String!` | Opaque cursor for this edge, to pass as `after` or `before` to continue from here. Never construct or parse one: the format is not part of the API contract and may change without notice. |

## LabsResult

Result type for paginated labs queries.

| Name | Type | Description |
| --- | --- | --- |
| `nodes` | `[LabRef!]!` | Labs on the current page. An empty list means the page is genuinely empty and never stands in for a failure, which fails the request with `UPSTREAM_UNAVAILABLE` or `TIMEOUT` instead. |
| `totalCount` | `Int!` | Labs matching the query, ignoring pagination; `0` means nothing matched, never a failure. On the `walletAddress` form, labs whose identifier cannot be read are excluded from this count and `nodes`. |
| `pageInfo` | `PageInfo!` |  |

## LabTaxonomyResult

Active lab taxonomy terms in display order, for classification through `updateLabNftMetadata`.

| Name | Type | Description |
| --- | --- | --- |
| `projectFormats` | `[LabTaxonomyTerm!]!` | Choices for `Lab.projectFormat`; each lab may select one. |
| `therapeuticFields` | `[LabTaxonomyTerm!]!` | Choices for `Lab.therapeuticFields`; each lab may select several. |

## LabTaxonomyTerm

Lab classification term with a stable slug and display title.

| Name | Type | Description |
| --- | --- | --- |
| `slug` | `String!` | Stable identifier of lowercase letters, digits and single hyphens, such as `brain-health`. |
| `title` | `String!` |  |

## LegalAgreementStatusResult

Status of a legal agreement for a lab. Queried through `legalAgreementStatus` a failure is thrown and `error` is always null; selected as a field on a lab, an upstream failure degrades in band instead of nulling the parent. Non-null `error` means the status is undetermined and the booleans are placeholders.

| Name | Type | Description |
| --- | --- | --- |
| `signed` | `Boolean!` | Whether any template version of this agreement type has been signed for this lab. Always present, but a placeholder rather than a verdict when `error` is non-null. |
| `isCurrentVersionSigned` | `Boolean!` | Whether the current template version has been signed, which is what clients route the signing flow on. A placeholder when `error` is non-null, so check `error` first. |
| `currentTemplateVersion` | `String` | Version of the agreement template a signer would be asked to sign today (e.g. "1.0.0"). Compare against `signedVersions[].templateVersion`. |
| `signedVersions` | `[SignedLegalAgreementVersion!]` | Every template version signed for this lab, oldest first, and empty when nothing has been signed. Only authoritative when `error` is null. |
| `error` | `ApiError` | Field-surface degraded signal (see type docstring). Null on success. |

## LegalAgreementTemplateResult

Payload of `legalAgreementTemplate`. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `agreement` | `AWSJSON` | Populated agreement JSON, for display only. Clients must not re-serialize or re-hash it, and should sign over `contentHash` as given. |
| `contentHash` | `String` | keccak256 (0x-prefixed) of the canonical JSON of `agreement`. This is the `contentHash` field of the EIP-712 LegalAgreementAcceptance payload (see AA-1). |
| `templateVersion` | `String` | Version of the agreement template the document was populated from (the current version for this agreement type, e.g. "1.0.0"). |
| `agreementType` | `LegalAgreementType` | Agreement type the document was populated for; echoes the `type` argument. |
| `issuedAt` | `AWSTimestamp` | Unix epoch seconds at generation. Clients MUST echo this verbatim into both the EIP-712 payload and the signLegalAgreement mutation; the backend regenerates the document from it. |

## LinkTokenResult

Result of `linkToken`.

| Name | Type | Description |
| --- | --- | --- |
| `token` | `Token` | Token after linking, hydrated as requested; null when `error` is non-null. |
| `error` | `ApiError` | Null on success; otherwise the failure, with no writes applied. |

## ListLabMembersResult

Result type for the listLabMembers query. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Human-readable status message. |
| `members` | `[LabMember!]!` | Active members on the lab. Empty array when the lab has no recorded members. |

## Market

DEX trading pair of an IP Token, with its latest price and liquidity figures.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Address of the pair contract, the same value as `pairAddress`. |
| `createdAt` | `AWSDateTime!` | When Molecule first indexed the pair. |
| `updatedAt` | `AWSDateTime!` | When the record was last updated (i.e. when the figures were refreshed). |
| `liquidityUsd` | `Float!` | Total liquidity of the trading pair, in USD. |
| `pairAddress` | `String!` |  |
| `usdPrice` | `Float!` | Current token price, in USD. |
| `usdPrice24hrPercentageChange` | `Float` | Price change over the last 24 hours, in percent. Null when not available. |
| `chain` | `Chain!` |  |
| `chainId` | `Int!` | Record id of the chain the market trades on (`Chain.id`). |
| `marketCapUsd` | `Float!` | USD market cap reported by the pool data provider, else its fully diluted valuation, else `usdPrice` times the circulating supply. An unknown supply then gives zero, not a zero valuation. |
| `tradingVolume24hr` | `Float!` | Trading volume over the last 24 hours, in USD. |
| `token` | `IPT!` | **Deprecated.** Use `Token.markets`. Removed after 2027-01-01. |
| `iptId` | `String!` | **Deprecated.** Use the parent token of `Token.markets`. Removed after 2027-01-01. |
| `inverted` | `Boolean!` | Whether the pair's token order is inverted relative to the IP Token (the IP Token is the pair's second token). |
| `name` | `String!` | Market name. May be empty. |

## MoveEntryResult

Result type for move entry operations.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Human-readable status message; mirrors `error.message` on failure. |
| `newHead` | `String` | New head hash after successful move (for optimistic concurrency control). |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## OnChainEvent

One transaction's onchain activity classified into a single timeline entry. An OCL creation renders as one `New Onchain Lab created` entry rather than a burst of raw events, and its constituent events stay available in `events`.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` | Cursor-stable group id, formatted `<blockNumber>:<logIndex>` from the newest matching event of the transaction. Pass the last entry's value as `cursor` to fetch the next page. |
| `chainId` | `Int!` | EVM chain id the transaction was mined on (e.g. 8453 for Base). |
| `txHash` | `String!` | Hash of the transaction, 0x-prefixed. Every entry in `events` shares it. |
| `blockNumber` | `String!` | Block height of the newest matching event of the transaction, as a decimal string. |
| `blockTimestamp` | `AWSDateTime!` | Block timestamp of the transaction, from its latest event carrying block metadata. Falls back to `1970-01-01T00:00:00.000Z` only while every event is awaiting reconciliation. |
| `type` | `String!` | One of: OCL_CREATED, OCL_TOKENIZED, OCL_TRANSFERRED, OCL_DID_LINKED, ROLE_GRANTED, ROLE_REVOKED, ROLE_CHANGED, IPT_TOKENIZED, IPNFT_MINTED, IPNFT_TRANSFERRED, IPNFT_METADATA_UPDATED, OTHER. |
| `title` | `String!` | Human-readable title, e.g. `New Onchain Lab created`. For `OTHER` it is the raw event name. |
| `args` | `AWSJSON!` | Structured facts of the classified action, such as oclId, from/to, role and account. Addresses appear lowercased in full; titles use shortened forms. |
| `events` | `[RawOnChainEvent!]!` | Raw events of the transaction in ascending log order. Includes events that did not match the oclId or wallet filter, giving full transaction context. |

## PageInfo

Page-based pagination information. Pages are 0-indexed.

| Name | Type | Description |
| --- | --- | --- |
| `hasNextPage` | `Boolean!` |  |
| `hasPreviousPage` | `Boolean!` | Whether a page before the current one exists (true for any page but the first). |
| `currentPage` | `Int!` | Page number, 0-indexed. |
| `totalPages` | `Int!` | Total pages; 0 when there are no results. |

## RawOnChainEvent

One decoded onchain event emitted by a Molecule contract.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` | Cursor-stable event id, formatted `<blockNumber>:<logIndex>`. Pass the last entry's value as `cursor` to fetch the next page. |
| `chainId` | `Int!` | EVM chain id the event was emitted on (e.g. 8453 for Base). |
| `contractAddress` | `String!` | Address of the emitting contract, lowercase 0x-hex. |
| `contractName` | `String!` | Logical subsystem the event came from. One of `accessresolver`, `ocl`, `ipnft`, `ipt`, `bio-agent`. |
| `eventName` | `String!` | Name of the decoded Solidity event exactly as emitted (e.g. `OclIdentityCreated`, `RoleGranted`, `Transfer`). Raw per-event name, not the classified `OnChainEvent.type`. |
| `blockNumber` | `String!` | Block height of the event as a decimal string, since block numbers can exceed the range of `Int`. |
| `blockTimestamp` | `AWSDateTime!` | Timestamp of the block that included the event. An event ingested before its block metadata was available carries `1970-01-01T00:00:00.000Z` until it is reconciled. |
| `txHash` | `String!` | Hash of the transaction that emitted the event, 0x-prefixed. With `logIndex` it identifies the event uniquely. |
| `logIndex` | `Int!` | Position of the event's log within its block. |
| `args` | `AWSJSON!` | Decoded event arguments. BigInts are decimal strings; addresses are lowercased. |

## ResearchLead

Research lead of an IP-NFT's project.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Email address of the research lead, taken from the IP-NFT metadata. The same value as `email`. |
| `createdAt` | `AWSDateTime!` | When Molecule first indexed the research lead. |
| `updatedAt` | `AWSDateTime!` | When Molecule last refreshed the record. |
| `name` | `String!` |  |
| `email` | `String!` |  |
| `ipnfts` | `[IPNFT!]!` | IP-NFTs led by this person. The arguments are accepted but ignored: the full list is returned. |

**`ipnfts` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortBy` | `IPNFTSortBy` | Accepted but ignored on this nested field. |
| `filterBy` | `IPNFTFilterBy` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

## SearchLabsAnnouncement

Lightweight announcement type for search results (no embedded lab).

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Announcement id, unique within the data room. |
| `headline` | `String!` | Title of the announcement. |
| `body` | `String!` | Body text of the announcement. |
| `attachments` | `[DataRoomFile!]!` | Files attached to the announcement. |
| `changeBy` | `String!` | Identity of the member who posted the announcement, as `did:ethr:` plus the EIP-55 checksummed wallet address. Older records may hold a bare address or an unverified value. |
| `systemTime` | `AWSDateTime!` | When the announcement was written by the data room backend. |
| `eventTime` | `AWSDateTime!` | When the announcement was posted. |

## SearchLabsAnnouncementHit

Announcement search result.

| Name | Type | Description |
| --- | --- | --- |
| `announcement` | `SearchLabsAnnouncement!` | Matching announcement. |
| `lab` | `LabRef!` | Lab the announcement was posted to (search-result hydration tier; see `LabRef`). |

## SearchLabsFileHit

File search result.

| Name | Type | Description |
| --- | --- | --- |
| `entry` | `DataRoomFileSearchEntry!` | Matching file and the lab it belongs to. |

## SearchLabsResult

Search results with pagination.

| Name | Type | Description |
| --- | --- | --- |
| `nodes` | `[SearchLabsHit!]!` | Hits on the requested page, most relevant first. An empty list means the search matched nothing or this page held only hits this API cannot yet render, and never stands in for a failure. |
| `totalCount` | `Int!` | Hits the search matched across all pages, counting matches rather than what this page could render, so it can exceed the length of `nodes`. Falls back to a lower bound when matches cannot be counted. |
| `pageInfo` | `PageInfo!` | Pagination information for the requested page. |

## ServiceSignInMessageResult

Result type for the getServiceSignInMessage query.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String!` | Message the service must sign with its wallet. Embeds a server-issued single-use nonce, so it must be fetched fresh before each signing. |
| `expiresAt` | `String` | ISO-8601 expiry of the embedded nonce; after this the signature is rejected and a fresh message must be requested. |

## ServiceTokenResult

Result of generating or extending a service token. Success when `error == null`; the token fields are populated only on success.

| Name | Type | Description |
| --- | --- | --- |
| `token` | `String` | Generated JWT token for service authentication. |
| `tokenId` | `String` | Unique identifier for the token (for tracking/revocation). |
| `serviceName` | `String` |  |
| `expiresAt` | `AWSDateTime` |  |
| `createdAt` | `AWSDateTime` |  |
| `message` | `String` | Message with additional information; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## ServiceTokenRevocationResult

Result of revoking a service token.

| Name | Type | Description |
| --- | --- | --- |
| `tokenId` | `String` | ID of the revoked token. |
| `message` | `String!` | Message describing the revocation result; mirrors `error.message` on failure. |
| `revokedAt` | `AWSDateTime` | Date/time when the token was revoked. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## SignedLegalAgreementVersion

One signed template version of a legal agreement for a lab.

| Name | Type | Description |
| --- | --- | --- |
| `templateVersion` | `String!` | Version of the agreement template that was signed (e.g. "1.0.0"). |
| `path` | `String!` | Canonical data room path of the signed artifact. |
| `signer` | `String` | Wallet address that signed. Null when the acceptance record is not available. |
| `signedAt` | `String` | ISO-8601 time the backend recorded the acceptance, an audit clock only, the legally effective date being `issuedAt`. Null when the acceptance record is not available. |
| `contentHash` | `String` | keccak256 (0x-prefixed) of the signed agreement document; the `contentHash` covered by the EIP-712 signature. Null when the acceptance record is not available. |
| `issuedAt` | `AWSTimestamp` | Signed effective date in epoch seconds, the value the EIP-712 signature covers, and the one to render as the effective date. Distinct from `signedAt`, the audit-only commit clock. |
| `signature` | `String` | EIP-712 signature over the acceptance typed data, so a third party can re-verify it offline. The backend's verification at sign time is authoritative. |

## SignLegalAgreementResult

Result of signLegalAgreement. Success when `error == null`; no partial writes. The payload fields are nullable and populated on success; `message` mirrors `error.message` on failure.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String` | OCL id of the lab the agreement was signed for. |
| `path` | `String` | Canonical data room path of the stored signed artifact (backend-derived). |
| `contentHash` | `String` | contentHash of the agreement node; matches the signed EIP-712 field. |
| `templateVersion` | `String` | Version of the agreement template that was signed (e.g. "1.0.0"). |
| `datasetId` | `String` | Data room dataset id of the stored signed artifact. |
| `version` | `Int` | Version number of the stored artifact in the data room (1 for a first signing of this template version). |
| `message` | `String` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## SignoffMetadataResult

Result of signoffMetadata. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `authorization` | `String` | Mint authorization for the IPNFT contract's mint call: a `0x`-hex EIP-191 signature by the Molecule key over `keccak256(abi.encodePacked(minter, to, ipnftId, tokenURI))`. Null on failure. |
| `isSuccess` | `Boolean!` | True when the terms signature was verified and the authorization issued. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |

## Token

Tracked token contract with its lab link, typed relations and markets. Manual lab links may be overridden by later onchain events.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` | Opaque token id, stable across re-indexing. |
| `chainId` | `Int!` | Chain id the contract is deployed on. |
| `address` | `String!` | Contract address, lowercase 0x-hex. |
| `kind` | `TokenKind!` |  |
| `name` | `String!` |  |
| `symbol` | `String!` |  |
| `decimals` | `Int!` | Number of decimal places for balances; `LOCKED` tokens use their underlying token's decimals. |
| `oclId` | `String` | OclId of the linked lab; null when the token is not linked to a lab. |
| `lab` | `LabRef` | Linked lab identity and display metadata; null when unlinked. Data-room, activity and assessment fields are null here; use `lab` to retrieve them. |
| `linkSource` | `TokenLinkSource!` | How the lab link was established. |
| `linkedBy` | `String` | Lowercase wallet address that manually linked this token; null for links established by onchain events or migration. |
| `linkedAt` | `AWSDateTime` | Timestamp when the lab link was last set or cleared, by any source; null if never linked. |
| `image` | `String` | Image URL; null when none is set. |
| `links` | `[String!]!` | Related URLs; empty when none. |
| `isCapped` | `Boolean!` | Whether the token supply is capped. |
| `initialSupply` | `String` | Initial supply as a decimal string in base units; null when unknown. |
| `totalIssued` | `String` | Total issued supply as a decimal string in base units; null when unknown. |
| `circulatingSupply` | `String` | Circulating supply as a decimal string in base units; null when unknown. |
| `holderCount` | `Int` | Number of distinct holders; null when not yet counted. |
| `totalLocked` | `String` | Locked amount as a decimal string in base units; null except on `LOCKED` tokens. |
| `unlockDelay` | `String` | Unlock delay as a decimal string in seconds; null except on `LOCKED` tokens. |
| `wrapperAddress` | `String` | Lowercase wrapper contract address; null except on `WRAPPED_LAB_TOKEN` tokens. |
| `agreementCid` | `String` | Content identifier of the membership agreement document; null when none. |
| `agreementMimeType` | `String` | MIME type of the membership agreement document; null when none. |
| `agreementStorageKey` | `String` | Storage key of the membership agreement document; null when none. |
| `agreementContentHash` | `String` | 0x-prefixed SHA-256 of the membership agreement document; null when none. |
| `mintedAt` | `AWSDateTime` | Block time the token contract was created; null when unknown. |
| `tokenizedAt` | `AWSDateTime` | Block time the token was tokenized for its lab; null when unknown. |
| `tokenizedBy` | `String` | Address that performed the tokenization, lowercase; null when unknown. |
| `relations` | `[TokenRelation!]!` | Relations involving this token, oldest first. Above 50 items fails with `COMPLEXITY_LIMIT_EXCEEDED`, reason `RESULT_CARDINALITY_LIMIT`. |
| `markets` | `[TokenMarket!]!` | Markets trading this token, highest liquidity first. Above 50 items fails with `COMPLEXITY_LIMIT_EXCEEDED`, reason `RESULT_CARDINALITY_LIMIT`. |
| `createdAt` | `AWSDateTime!` | When the record was created. |
| `updatedAt` | `AWSDateTime!` | When the record was last updated. |

## TokenConnection

Cursor-paginated token connection.

| Name | Type | Description |
| --- | --- | --- |
| `edges` | `[TokenEdge!]!` |  |
| `nodes` | `[Token!]!` | Tokens in the same order as `edges`. |
| `pageInfo` | `ConnectionPageInfo!` |  |
| `totalCount` | `Int` | Number of tokens matching the filter across all pages; computed only when selected. |

## TokenEdge

Token and its opaque pagination cursor.

| Name | Type | Description |
| --- | --- | --- |
| `node` | `Token!` |  |
| `cursor` | `String!` | Opaque position for `after` or `before`; its format may change. |

## TokenMarket

Trading pair for a token, with the latest price and liquidity figures.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Market id (the pool address). |
| `chainId` | `Int!` | Chain id of the pool. |
| `pairAddress` | `String!` | Address of the pair contract. |
| `name` | `String!` | Market name; empty when not known. |
| `isInverted` | `Boolean!` | Whether the token is the pair's second token. |
| `liquidityUsd` | `Float!` | Total liquidity in USD. |
| `usdPrice` | `Float!` | Current token price in USD. |
| `usdPrice24hrPercentageChange` | `Float` | Price change over the last 24 hours, in percent. Null when not available. |
| `marketCapUsd` | `Float!` | Market capitalization in USD. |
| `tradingVolume24hr` | `Float!` | Trading volume over the last 24 hours, in USD. |
| `createdAt` | `AWSDateTime!` | When the record was created. |
| `updatedAt` | `AWSDateTime!` | When the figures were last refreshed. |

## TokenRef

Token identity used at relation endpoints; full records are available through `tokenById`.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` |  |
| `chainId` | `Int!` | Chain id the contract is deployed on. |
| `address` | `String!` | Contract address, lowercase 0x-hex. |
| `kind` | `TokenKind!` |  |
| `name` | `String!` |  |
| `symbol` | `String!` |  |
| `decimals` | `Int!` | Number of decimals of the contract. |
| `oclId` | `String` | OclId of the linked lab; null when the token is not linked to a lab. |

## TokenRelation

Directed relation from a dependent token to its canonical or underlying token.

| Name | Type | Description |
| --- | --- | --- |
| `relationType` | `TokenRelationType!` |  |
| `token` | `TokenRef!` | Dependent token: bridged copy, locked wrapper, or wrapper. |
| `related` | `TokenRef!` | Canonical or underlying token. |
| `source` | `TokenLinkSource!` | How the relation was established. |
| `createdAt` | `AWSDateTime!` | When the relation was recorded. |

## UnlinkTokenResult

Result of `unlinkToken`.

| Name | Type | Description |
| --- | --- | --- |
| `token` | `Token` | Token after unlinking, hydrated as requested; null when `error` is non-null. |
| `error` | `ApiError` | Null on success; otherwise the failure, with no writes applied. |

## UpdateFileMetadataResult

Result type for file metadata update operations.

| Name | Type | Description |
| --- | --- | --- |
| `ref` | `String` | Reference (DID) of the updated file. |
| `message` | `String!` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## UpdateLabNftMetadataResult

Result of `updateLabNftMetadata`.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String` | OCL id of the updated lab. Null on failure. |
| `message` | `String` | Human-readable status message; mirrors `error.message` on failure. |
| `error` | `ApiError` | Null on success. Non-null means the mutation failed. |

## UploadMetadataWithKeyResult

Result of uploadMetadataWithImageKey. Success when `isSuccess` is true.

| Name | Type | Description |
| --- | --- | --- |
| `metadataCid` | `String` | IPFS CID of the stored IPNFT metadata document, used as `metadataCid` for `getTermsMessage` and as the mint's `tokenURI`. Null on failure. |
| `metadataUrl` | `String` | HTTPS gateway URL of the stored metadata document. Null on failure. |
| `isSuccess` | `Boolean!` | True when the metadata was validated and stored. |
| `error` | `EvmTokenizationError` | Null on success. Non-null means the mutation failed. |

## User

Wallet known to the IP-NFT registry as an owner or original minter.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `String!` | Checksummed wallet address, the same value as `address`. |
| `createdAt` | `AWSDateTime!` | When Molecule first indexed the wallet. |
| `updatedAt` | `AWSDateTime!` | When Molecule last refreshed the record. |
| `address` | `String!` | Checksummed EVM address; match it exactly when filtering. |
| `ipnfts` | `[IPNFT!]!` | IP-NFTs owned by this wallet; the arguments are accepted but ignored. Whether this field resolves data is unverified, so prefer `ipnfts(filterBy: { userId })`. |
| `ipts` | `[IPT!]!` | **Deprecated.** Use `tokens` with an IPT kind filter. Removed after 2027-01-01. IP Tokens originally issued to this wallet; the arguments are accepted but ignored. Whether this field resolves data is unverified, so prefer `ipts(filterBy: { originalOwnerId })`. |

**`ipnfts` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortBy` | `IPNFTSortBy` | Accepted but ignored on this nested field. |
| `filterBy` | `IPNFTFilterBy` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

**`ipts` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `limit` | `Int` | Accepted but ignored on this nested field. |
| `skip` | `Int` | Accepted but ignored on this nested field. |
| `sortBy` | `IPTSortBy` | Accepted but ignored on this nested field. |
| `filterBy` | `IPTFilterBy` | Accepted but ignored on this nested field. |
| `sortOrder` | `SortOrder` | Accepted but ignored on this nested field. |

## LabActivityNode

One activity-feed entry. Select `__typename` to tell file events from announcements.

One of `LabEventFileUpdated`, `LabEventFileRemoved`, `LabEventFileAdded`, `LabEventAnnouncement`.

## SearchLabsHit

Union type for search results - can be either a file or announcement.

One of `SearchLabsFileHit`, `SearchLabsAnnouncementHit`.

