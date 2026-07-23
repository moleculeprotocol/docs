# 🔐 Authentication

All Molecule APIs require authentication. This page covers how to obtain credentials, which headers each API expects, and the specific authentication model for the Labs API.

## Obtaining API Access

All Molecule APIs require authentication with an API key. To request access:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team with your use case
3. You'll receive:
   * **API Key** - Required for all APIs
   * **Service Token** - Additional token for Labs API (if needed)

## Authentication Headers

| API                      | Required Headers                                              | Example                                                                                         |
| ------------------------ | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Labs API (queries)**   | `x-api-key`                                                   | `x-api-key: YOUR_API_KEY`                                                                       |
| **Labs API (mutations)** | <p><code>x-api-key</code><br><code>X-Service-Token</code></p> | <p><code>x-api-key: YOUR_API_KEY</code><br><code>X-Service-Token: YOUR_SERVICE_TOKEN</code></p> |
| **Tokenization API**     | `x-api-key`                                                   | `x-api-key: YOUR_API_KEY`                                                                       |
| **IPNFT API (Deprecated)**             | `x-api-key`                                                   | `x-api-key: YOUR_API_KEY`                                                                       |

***

## Labs API Authentication

The Labs API has different authentication requirements depending on the operation type:

> **Rule of thumb**: Most **queries** are public (API Key only) and all write **mutations** require a Service Token (API Key + Service Token). The exceptions are called out below — a couple of queries are role-gated, and `generateServiceToken` bootstraps a token with a wallet signature.

Summary of the model:

* **Most queries are public**: API Key only for read operations. Exception: `legalAgreementTemplate` requires a Service Token or an authenticated session.
* **Write mutations are protected**: API Key + Service Token required. Exception: `generateServiceToken` mints a token and needs only an API Key plus a Privy session or wallet signature.
* **Service Token**: Identifies which specific lab/dataroom you have write access to.
* File-level access control is handled via Molecule's Onchain-Verified Envelope Encryption, not query authentication — see [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md).

### Public Queries (Read-Only)

**These queries** are public and only require an API Key:

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
x-api-key: YOUR_API_KEY
```

**Authenticated query** — API Key **plus** a Service Token, or an authenticated user session:

- `legalAgreementTemplate` - Get the populated agreement to sign (the signer's authenticated session, or a service token)

### Protected Mutations (Write Operations)

**All write mutations** require authentication with **two headers**:

1. **API Key** - For general API authentication
2. **Service Token** - For lab-specific write access control

**Protected mutations include:**

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
- `extendServiceToken` - Extend service token expiration
- `revokeServiceToken` - Revoke a service token

> **`generateServiceToken` is the bootstrap exception**: it mints a Service Token, so it needs only an API Key plus either a Privy session or a wallet signature — not a pre-existing Service Token. See [Obtaining Tokens](labs-api/service-tokens.md#obtaining-tokens).

> **Pay-per-call alternative.** Mutations tagged 💳 above can also be called through the [x402 Gateway](x402-gateway.md), which settles a USDC payment on Base per request and mints a short-lived service token on the fly — no long-lived credentials required. Useful for autonomous AI agents and third-party tools that pay for users.

```bash
x-api-key: YOUR_API_KEY
X-Service-Token: YOUR_SERVICE_TOKEN
```

### Obtaining API Key and Service Token

To obtain access credentials:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team and provide:
   - Your wallet address (will be linked to the service token)
   - Intended use case / service name
   - Which lab/dataroom you need access to
   - Desired token expiration period
3. The team will generate and provide you with:
   - **API Key** - Used for all Molecule APIs
   - **Service Token** (JWT string) - Grants access to specific lab
   - **Token ID** - For management operations

### Using Your Credentials

**For all queries** (read-only operations):

```bash
x-api-key: YOUR_API_KEY
```

**For all mutations** (write operations):

```bash
x-api-key: YOUR_API_KEY
X-Service-Token: YOUR_SERVICE_TOKEN
```

**Why two tokens for mutations?**

- **API Key**: Authenticates you as a valid Molecule API user
- **Service Token**: Identifies which specific lab/dataroom you have write access to

**Security Warnings:**

- Service tokens are shown only once during generation - store them securely immediately
- Never commit tokens or API keys to version control
- Never log credentials in application logs
- Store in environment variables or secure secret management systems
- Rotate tokens regularly (quarterly recommended)

> Service Token lifecycle operations (extending, revoking) are documented in [Service Tokens](labs-api/service-tokens.md).
