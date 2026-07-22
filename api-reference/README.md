# 📡 API Overview

## Overview

The Molecule Protocol provides programmatic APIs for building applications, integrations, and automated workflows on top of decentralized science infrastructure. These APIs enable developers to query project data, tokenize research, and manage research datarooms.

## API Areas

### 📊 Data API

Query and browse IP-NFTs, IP Tokens (IPTs), and market data across the Molecule ecosystem.

**Purpose:**

* Browse all IP-NFTs and IPTs on the platform
* Query metadata, ownership, and project details
* Access trading data and market metrics
* Build marketplace UIs and token screeners

**Authentication:** API Key required

[View Data API Documentation →](data-api.md)

***

### 🔐 Tokenization API

Tokenize Labs into fungible IP Tokens (IPTs) on Base.

**Purpose:**

* Tokenize Labs into tradeable ERC-20 tokens
* Generate Lab (OCL) membership agreements
* Manage the complete onchain tokenization workflow

**Authentication:** API Key required

[View Tokenization API Documentation →](tokenization-api.md)

***

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

* **Most queries** (read operations): API Key only — public. One exception, `legalAgreementTemplate`, needs a Service Token or an authenticated session.
* **Write mutations** (write operations): API Key + Service Token required — except `generateServiceToken`, which bootstraps a token from a Privy session or wallet signature.

[View Labs API Documentation →](labs-api.md)

***

## Authentication

### Obtaining API Access

All Molecule APIs require authentication with an API key. To request access:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team with your use case
3. You'll receive:
   * **API Key** - Required for all APIs
   * **Service Token** - Additional token for Labs API (if needed)

### Authentication Headers

| API                      | Required Headers                                              | Example                                                                                         |
| ------------------------ | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Data API**             | `x-api-key`                                                   | `x-api-key: YOUR_API_KEY`                                                                       |
| **Tokenization API**     | `x-api-key`                                                   | `x-api-key: YOUR_API_KEY`                                                                       |
| **Labs API (queries)**   | `x-api-key`                                                   | `x-api-key: YOUR_API_KEY`                                                                       |
| **Labs API (mutations)** | <p><code>x-api-key</code><br><code>X-Service-Token</code></p> | <p><code>x-api-key: YOUR_API_KEY</code><br><code>X-Service-Token: YOUR_SERVICE_TOKEN</code></p> |

**Labs API Authentication Details:**

* **Most queries are public**: API Key only for read operations. Exception: `legalAgreementTemplate` requires a Service Token or an authenticated session.
* **Write mutations are protected**: API Key + Service Token required. Exception: `generateServiceToken` mints a token and needs only an API Key plus a Privy session or wallet signature.
* **Service Token**: Identifies which specific lab/dataroom you have write access to
* File-level access control is handled via Molecule's Onchain-Verified Envelope Encryption (Lit Protocol retained for legacy files), not query authentication — see [Data Privacy & Access](../core-infrastructure/data/data-privacy-and-access.md)

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

Contact the Molecule team via [Discord](https://t.co/L0VEiy4Bjk) to obtain your API key.

### 2. Choose Your API

| If you want to...                    | Use this API                            |
| ------------------------------------ | --------------------------------------- |
| Browse IP-NFTs and IPTs              | [Data API](data-api.md)                 |
| Check market prices and trading data | [Data API](data-api.md)                 |
| Tokenize a Lab into IP Tokens (IPTs) | [Tokenization API](tokenization-api.md) |
| Upload files to a Lab dataroom       | [Labs API](labs-api.md)                 |

### 3. Make Your First Request

**Example (Data API):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query { ipnfts(limit: 5) { id name trlValue } }"
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
