---
description: >-
  Every Molecule-specific term used in the API docs, defined in one or two
  sentences, with a link to the page that goes deeper.
icon: book-open
---

# Glossary

If a term in the tutorials is unfamiliar, it is defined here. Each entry is short on purpose — follow the link when you need the full picture.

***

## The Lab

**Lab** — the core object you build against. A Lab is an NFT that owns its own smart-contract wallet, so it can hold assets, store research files, and grant other people access to them. One research project, one Lab. Full model: [Molecule Labs](../technical-deep-dive/onchain-lab.md).

**LabNFT** — the ERC-721 token that represents a Lab. Whoever holds it controls the Lab. Minting a LabNFT is the onchain step that brings a new Lab into existence; you do it once, before any API call can attach data to it.

**Lab account (Token Bound Account, TBA)** — the smart-contract wallet permanently bound to the LabNFT (ERC-6551). It has no private key of its own: it takes its authority from whoever currently holds the NFT. Its address is the Lab's permanent identity, and it does not change when the NFT is sold or transferred.

**OCL / `oclId`** — "onchain lab". `oclId` is the Lab's canonical identifier: a 32-byte hex string (`0x…`) emitted in the `OclIdentityCreated` event when the LabNFT is minted. Every Labs API call that targets a Lab takes this value. How it is derived: [Lab Management](../api-reference/labs-api/lab-management.md#how-oclid-is-derived).

**`shortname`** — a URL slug derived server-side from the Lab's name, used in the Lab's public page address. It is `null` until the server has derived it, so a freshly minted Lab is reachable by `oclId` before it is reachable by slug.

***

## Data

**Data room** — a Lab's file store. Every file you upload lands in the Lab's data room at a `path` you choose, and the data room keeps every version of it rather than overwriting.

**Kamu** — the data layer behind the data room. It records each file's version history, content hash, author and provenance. You never call Kamu directly; the Labs API does it for you. Details: [Data Storage](../technical-deep-dive/data/data-storage.md).

**`accessLevel`** — whether a file is `PUBLIC` (stored as-is, readable by anyone) or confidential (encrypted before upload, readable only by wallets that pass its access conditions).

**DEK (data encryption key)** — a fresh AES-256-GCM key generated per confidential file. You encrypt the file with it locally, and the API stores the key in wrapped form. It is released to you only when an onchain check confirms your wallet still qualifies. Details: [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md).

**Access control conditions** — the rules attached to a confidential file that decide who may decrypt it, evaluated against live onchain state at the moment of the request rather than at upload time.

***

## Access and roles

**Owner** — the wallet holding the LabNFT. Passes every permission check and is the only role that can grant Contributor.

**Contributor** — an explicit onchain grant (`ROLE_CONTRIBUTOR = 2`). Can read and write the data room and grant Viewers, but cannot add other Contributors or transfer the NFT. This is the role an agent needs in order to upload.

**Viewer** — an explicit onchain grant (`ROLE_VIEWER = 1`). Read-only, including decrypting confidential files.

Role checks are hierarchical: a Contributor passes Viewer checks, and the Owner passes everything. Full matrix: [Roles & Permissions](../technical-deep-dive/roles-and-permissions.md).

**AccessResolver** — the contract that holds those role grants and answers `hasRole`. Granting a role is an onchain transaction sent by the Lab owner. Reference: [AccessResolver](contracts/accessresolver.md).

***

## Calling the API

**Consumer credential** — the `mol_<consumerId>_<secret>` string that identifies your integration. It goes in the `Authorization` header on every request, with **no `Bearer` prefix**, and it is issued per environment. This is the one credential you have to request from the Molecule team.

**Service token** — proof that you control a particular wallet. You issue it yourself by signing a message with that wallet, then send it as the `X-Service-Token` header. What it lets you write is decided by that wallet's onchain role on the target Lab, not by the token itself. Reference: [Service Tokens](../api-reference/labs-api/service-tokens.md).

**Privy user token** — the alternative to a service token, used when a human is signed in through the Molecule app rather than a script. It is the one credential that does use `Authorization: Bearer …`. Reference: [Authentication](../api-reference/authentication.md).

**EOA (externally owned account)** — an ordinary wallet controlled by a private key, as opposed to a smart-contract wallet. This is what you sign with and what pays gas when you mint a LabNFT yourself.

**x402** — a gateway that lets you pay per request in USDC on Base instead of holding a long-lived service token. The price comes back in the `402` response; read it from there rather than hardcoding it. Reference: [x402 Gateway](../api-reference/x402-gateway.md).

**Indexer / indexer lag** — onchain events reach the API through an indexing service, which takes a moment to catch up. A transaction that has confirmed onchain is therefore not immediately visible to the API, which is why a write straight after a mint or a role grant can fail and should be retried rather than treated as a real error.

**Staging vs production** — staging runs on Base Sepolia with testnet funds and nothing costs real money; production runs on Base mainnet. Credentials are per environment and are not interchangeable.
