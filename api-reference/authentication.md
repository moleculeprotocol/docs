# 🔐 Authentication

All Molecule APIs require authentication. This page covers how to obtain credentials, which headers each API expects, and the specific authentication model for the Labs API.

## Obtaining API Access

All Molecule APIs require authentication with a consumer credential. To request access:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team with your use case
3. You'll receive:
   * **Consumer credential** (`mol_<consumerId>_<secret>`) - Required for all APIs
   * **Service Token** - Additional token for Labs API (if needed)

Send the consumer credential as a bearer token: `Authorization: Bearer mol_<consumerId>_<secret>`. Treat the entire string as a secret — it is not split into a public/private part.

## Authentication Headers

| API                                       | Required Headers                                                                                                  | Example                                                                                                                       |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Labs API (queries)**                    | `Authorization`                                                                                                   | `Authorization: Bearer mol_<consumerId>_<secret>`                                                                            |
| **Labs API (mutations, service token)**   | <p><code>Authorization</code><br><code>X-Service-Token</code></p>                                                 | <p><code>Authorization: Bearer mol_&lt;consumerId&gt;_&lt;secret&gt;</code><br><code>X-Service-Token: YOUR_SERVICE_TOKEN</code></p> |
| **Labs API (mutations, Privy user)**      | <p><code>Authorization</code><br><code>x-wallet-address</code></p>                                                | <p><code>Authorization: Bearer PRIVY_TOKEN</code><br><code>x-wallet-address: 0x…</code></p>                                  |
| **Tokenization API**                      | `Authorization`                                                                                                   | `Authorization: Bearer mol_<consumerId>_<secret>`                                                                            |
| **IPNFT API (Deprecated)**                | `Authorization`                                                                                                   | `Authorization: Bearer mol_<consumerId>_<secret>`                                                                            |

***

## Labs API Authentication

The Labs API has different authentication requirements depending on the operation type:

> **Rule of thumb**: Most **queries** are public (consumer credential only). Write **mutations** are authenticated, and most accept **either** a Service Token **or** a Privy user session — pick whichever fits your caller. The exceptions are called out below: one query is gated, the two Service Token lifecycle mutations are service-token-only, and `generateServiceToken` bootstraps a token with a Privy session or wallet signature.

Summary of the model:

* **Most queries are public**: consumer credential only for read operations. Exception: `legalAgreementTemplate` requires a Service Token or an authenticated session.
* **Write mutations are authenticated, with two interchangeable paths**: consumer credential plus **either** `X-Service-Token` (machine callers — services, bots, agents) **or** `Authorization` + `x-wallet-address` (Privy user session — browser and app callers). Authorization is then evaluated against the caller's identity either way.
* **Exceptions**: `extendServiceToken` and `revokeServiceToken` accept **only** a Service Token. `generateServiceToken` accepts **only** a Privy session or wallet signature, since it mints the token in the first place.
* **Service Token**: Identifies which specific lab/dataroom you have write access to.
* File-level access control is handled via Molecule's Onchain-Verified Envelope Encryption, not query authentication — see [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md).

### Public Queries (Read-Only)

**These queries** are public and only require a consumer credential:

- `labs` - List all labs with pagination
- `labWithDataRoomAndFiles` - Get lab details and files
- `labActivity` - Get activity feed for a lab, (available filters: ANNOUNCEMENT | FILE)
- `activities` - Get global activity feed, (available filters: ANNOUNCEMENT | FILE)
- `dataRoomFile` - Get file by path
- `searchLabs` - Search across labs, files, and announcements
- `fileCategoriesAndTags` - List valid file categories and their tags
- `getServiceSignInMessage` - Get the message a service signs to obtain a token
- `getDidLinkStatus` - Get background DID-linking status for a lab
- `legalAgreementStatus` - Check whether a lab's legal agreement is signed
- `onChainActivity` - Onchain event feed for a lab or wallet
- `listLabMembers` - List a lab's members

```bash
Authorization: Bearer YOUR_CONSUMER_CREDENTIAL
```

**Authenticated query** — consumer credential **plus** a Service Token, or an authenticated user session:

- `legalAgreementTemplate` - Get the populated agreement to sign (the signer's authenticated session, or a service token)

### Protected Mutations (Write Operations)

All write mutations require a **consumer credential** plus proof of caller identity. For most mutations there are **two interchangeable ways** to prove identity — the resolver accepts a Service Token if one is present, and otherwise falls back to authenticating the Privy user:

**Option 1 — Service Token** (services, bots, agents, CI/CD):

```bash
Authorization: Bearer YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

**Option 2 — Privy user session** (browser and app callers acting as a signed-in user):

```bash
Authorization: Bearer YOUR_PRIVY_TOKEN
x-wallet-address: YOUR_WALLET_ADDRESS
```

Either way, the caller still has to be authorized for the target lab — a Service Token carries its own lab scope, and a Privy session is checked against the wallet's onchain role (LabNFT owner, authorized multisig signer, or an active role on `AccessResolver`). Supplying neither returns a `NO_AUTH` error naming both paths.

**Mutations accepting either path:**

- `createLab` - Create a lab (data room) for an onchain lab (OCL) · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `initiateCreateOrUpdateFile` - Initiate file upload · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `finishCreateOrUpdateFile` - Complete file upload · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `updateFileMetadata` - Update file metadata
- `deleteDataRoomFile` - Delete a file
- `createAnnouncement` - Create an announcement · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `updateLabNftMetadata` - Update LabNFT display metadata (OCL admin only)
- `generateLabImageUploadUrl` - Get a presigned URL to upload a LabNFT image (OCL admin only)
- `signLegalAgreement` - Record acceptance of a legal agreement
- `generateDataEncryptionKey` - Generate a standalone data encryption key · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)
- `decryptDataKey` - Decrypt a file's data key for an authorized caller · 💳 also available pay-per-call via [x402 Gateway](x402-gateway.md)

**Service-Token-only mutations** — these manage token lifecycle and reject Privy sessions:

- `extendServiceToken` - Extend service token expiration
- `revokeServiceToken` - Revoke a service token

```bash
Authorization: Bearer YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

> **`generateServiceToken` is the bootstrap exception**, in the opposite direction: it mints a Service Token, so it accepts *only* a consumer credential plus either a Privy session or a wallet signature — not a pre-existing Service Token. See [Obtaining Tokens](labs-api/service-tokens.md#obtaining-tokens).

> **Pay-per-call alternative.** Mutations tagged 💳 above can also be called through the [x402 Gateway](x402-gateway.md), which settles a USDC payment on Base per request and mints a short-lived service token on the fly — no long-lived credentials required. Useful for autonomous AI agents and third-party tools that pay for users.

### Obtaining a Consumer Credential and Service Token

To obtain access credentials:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team and provide:
   - Your wallet address (will be linked to the service token)
   - Intended use case / service name
   - Which lab/dataroom you need access to
   - Desired token expiration period
3. The team will generate and provide you with:
   - **Consumer credential** (`mol_<consumerId>_<secret>`) - Used for all Molecule APIs
   - **Service Token** (JWT string) - Grants access to specific lab
   - **Token ID** - For management operations

### Using Your Credentials

**For all queries** (read-only operations):

```bash
Authorization: Bearer YOUR_CONSUMER_CREDENTIAL
```

**For mutations** (write operations) — as a service:

```bash
Authorization: Bearer YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

**For mutations** — as a signed-in user (accepted by all mutations except `extendServiceToken` and `revokeServiceToken`):

```bash
Authorization: Bearer YOUR_PRIVY_TOKEN
x-wallet-address: YOUR_WALLET_ADDRESS
```

**Why more than one header for mutations?**

- **Consumer credential**: Authenticates you as a valid Molecule API consumer
- **Service Token**: Identifies which specific lab/dataroom your service has write access to
- **Privy token + wallet address**: Identifies the human caller instead, whose write access is derived from their wallet's onchain role

Which path to choose: use a Service Token for unattended callers (backends, bots, agents, CI/CD) where there is no user session to draw on. Use the Privy path when a signed-in user is driving the request, so the action is attributed to their wallet and governed by their onchain role rather than a shared service credential.

**Security Warnings:**

- Service tokens are shown only once during generation - store them securely immediately
- Never commit tokens or consumer credentials to version control
- Never log credentials in application logs
- Store in environment variables or secure secret management systems
- Rotate tokens regularly (quarterly recommended)

> Service Token lifecycle operations (extending, revoking) are documented in [Service Tokens](labs-api/service-tokens.md).
