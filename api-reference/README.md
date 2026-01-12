# 📡 API Reference

## Overview

The Molecule Protocol provides programmatic APIs for building applications, integrations, and automated workflows on top of decentralized science infrastructure. These APIs enable developers to query project data, mint IP-NFTs, tokenize intellectual property, and manage research datarooms.

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

Create IP-NFTs from research projects and tokenize them into fungible IP Tokens (IPTs).

**Purpose:**
* Mint new IP-NFTs with legal agreements
* Upload artwork and metadata to IPFS
* Tokenize IP-NFTs into tradeable ERC-20 tokens
* Manage the complete on-chain minting workflow

**Authentication:** API Key required

[View Tokenization API Documentation →](tokenization-api.md)

***

### 📁 Labs API (Programmatic File Upload)

Upload files to project datarooms for secure, decentralized research data storage.

**Purpose:**
* Automate file uploads to Labs datarooms
* Integrate with data pipelines and CI/CD
* Batch upload research data
* Manage file versions and metadata
* Query projects and files (public access)

**Authentication:**
* **All queries** (read operations): API Key only - all queries are public
* **All mutations** (write operations): API Key + Service Token required

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

| API | Required Headers | Example |
|-----|-----------------|---------|
| **Data API** | `x-api-key` | `x-api-key: YOUR_API_KEY` |
| **Tokenization API** | `x-api-key` | `x-api-key: YOUR_API_KEY` |
| **Labs API (queries)** | `x-api-key` | `x-api-key: YOUR_API_KEY` |
| **Labs API (mutations)** | `x-api-key`<br/>`X-Service-Token` | `x-api-key: YOUR_API_KEY`<br/>`X-Service-Token: YOUR_SERVICE_TOKEN` |

**Labs API Authentication Details:**
* **All queries are public**: Only API Key required for any read operation
* **All mutations are protected**: API Key + Service Token required for any write operation
* **Service Token**: Identifies which specific lab/dataroom you have write access to
* File-level access control is handled via Lit Protocol encryption, not query authentication

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

| If you want to... | Use this API |
|-------------------|--------------|
| Browse IP-NFTs and IPTs | [Data API](data-api.md) |
| Check market prices and trading data | [Data API](data-api.md) |
| Mint a new IP-NFT | [Tokenization API](tokenization-api.md) |
| Create IP Tokens from an IP-NFT | [Tokenization API](tokenization-api.md) |
| Upload files to a Lab dataroom | [Labs API](../molecule-labs/programmatic-file-upload.md) |

### 3. Make Your First Request

**Example (Data API):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query { ipnfts(limit: 5) { id metadata { name } } }"
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

* [Molecule Documentation](https://docs.molecule.to/documentation/)
* [Smart Contract Addresses](../ip-nfts/technical-components-of-ip-nfts/smart-contract-addresses.md)
* [What are IP-NFTs?](../ip-nfts/intro-to-ip-nft.md)
* [What are IP Tokens?](../ip-tokens/what-are-ipts.md)

***

_Last updated: January 2025_
