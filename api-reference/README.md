# 📡 API Overview

## Overview

The Molecule Protocol provides programmatic APIs for building applications, integrations, and automated workflows on top of decentralized science infrastructure. These APIs enable developers to query project data, tokenize research, and manage research datarooms.

## API Areas

### 📁 Labs API

Upload files to lab datarooms for secure, decentralized research data storage, and query labs, members, activity, and legal-agreement status.

**Purpose:**

* Create labs (datarooms) for onchain labs (OCLs)
* Automate file uploads to lab datarooms
* Integrate with data pipelines and CI/CD
* Batch upload research data
* Manage file versions, metadata, and LabNFT display metadata
* Query labs, files, members, activity, and onchain events (mostly public access)
* Manage service tokens and legal-agreement signing

**Authentication:**

* **Most queries** (read operations): consumer credential only — public. One exception, `legalAgreementTemplate`, needs a Service Token or an authenticated session.
* **Write mutations** (write operations): consumer credential plus **either** a Service Token (`X-Service-Token`) **or** a Privy user session (`Authorization` + `x-wallet-address`) — the two paths are interchangeable. Exceptions: `extendServiceToken` and `revokeServiceToken` are Service-Token-only, and `generateServiceToken` bootstraps a token from a Privy session or wallet signature.

[View Labs API Documentation →](labs-api/README.md)

***

### 🔐 Tokenization API

Tokenize Labs into fungible IP Tokens (IPTs) on Base.

**Purpose:**

* Tokenize Labs into tradeable ERC-20 tokens
* Generate Lab (OCL) membership agreements
* Manage the complete onchain tokenization workflow

**Authentication:** Consumer credential required

[View Tokenization API Documentation →](tokenization-api.md)

***

### 💳 x402 Gateway

Pay-per-call HTTP 402 gateway that fronts a set of Labs API write mutations with per-request USDC settlement on Base.

**Purpose:**

* Give autonomous agents and third-party tools write access without a long-lived service token
* Pay per mutation call in USDC, settled on Base
* Mint short-lived, scoped service tokens on the fly after payment

**Authentication:** Per-request stablecoin payment (no long-lived service token required)

[View x402 Gateway Documentation →](x402-gateway.md)

***

### 📊 IPNFT API (Deprecated)

Query and browse IP-NFTs, IP Tokens (IPTs), and market data across the Molecule ecosystem.

**Purpose:**

* Browse all IP-NFTs and IPTs on the platform
* Query metadata, ownership, and project details
* Access trading data and market metrics
* Build marketplace UIs and token screeners

**Authentication:** Consumer credential required

[View IPNFT API Documentation (Deprecated) →](ipnft-api.md)

***

## Authentication

All Molecule APIs require a consumer credential; the Labs API additionally uses a Service Token for write operations. Obtaining credentials, the per-API header requirements, and the full Labs API authentication model (public queries vs. protected mutations) are documented on the dedicated [Authentication](authentication.md) page.

***

## API Endpoints

All APIs use the same GraphQL endpoint:

```
Production: https://production.graphql.api.molecule.xyz/graphql
Staging:    https://staging.graphql.api.molecule.xyz/graphql
```

***

## Quick Start Guide

### 1. Get API Access

Contact the Molecule team via [Discord](https://t.co/L0VEiy4Bjk) to obtain your consumer credential.

### 2. Choose Your API

| If you want to...                        | Use this API                            |
| ---------------------------------------- | --------------------------------------- |
| Upload files to a Lab dataroom           | [Labs API](labs-api/README.md)          |
| Tokenize a Lab into IP Tokens (IPTs)     | [Tokenization API](tokenization-api.md) |
| Pay per call without a long-lived token  | [x402 Gateway](x402-gateway.md)         |
| Browse IP-NFTs and IPTs (legacy)         | [IPNFT API (Deprecated)](ipnft-api.md)  |
| Check market prices and trading data (legacy) | [IPNFT API (Deprecated)](ipnft-api.md) |

### 3. Make Your First Request

**Example (Labs API — public `labs` query, consumer credential only):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query { labs(perPage: 5) { nodes { oclId name shortname } totalCount } }"
  }'
```

***

## Getting Support

If you encounter any issues or have questions about the APIs:

* **Discord**: Join our [Discord community](https://t.co/L0VEiy4Bjk) for support
* **Documentation**: Check the specific API documentation pages linked above
* **Contact**: Reach out to the Molecule development team

***

## Additional Resources

* [Smart Contract Addresses](../references/contracts/)

***

_Last updated: July 2026_
