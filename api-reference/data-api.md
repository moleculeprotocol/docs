# 📊 Data API

## Overview

The Data API provides read-only access to query and browse intellectual property assets across the Molecule Protocol. Use these queries to build marketplaces, token screeners, portfolio trackers, and discovery interfaces for decentralized science projects.

**Features:**
* Query IP-NFTs (Intellectual Property NFTs) and their metadata
* Browse IP Tokens (IPTs) with market data
* Access trading metrics and liquidity information
* Filter, sort, and paginate results
* Build data-driven applications

***

## Authentication

All Data API requests require an API key.

### Obtaining an API Key

To request an API key:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team
3. Provide your intended use case
4. You'll receive your API key

### Using Your API Key

Include the API key in all requests using the `x-api-key` header:

```bash
x-api-key: YOUR_API_KEY
```

***

## API Endpoint

```
Production: https://production.graphql.api.molecule.xyz/graphql
Staging:    https://staging.graphql.api.molecule.xyz/graphql
```

***

## Queries

### List IP-NFTs

Query and browse all IP-NFTs on the platform with filtering, sorting, and pagination.

**GraphQL Query:**

```graphql
query ListIPNFTs(
  $limit: Int
  $skip: Int
  $sortBy: IPNFTSortBy
  $sortOrder: SortOrder
  $filterBy: IPNFTFilterBy
) {
  ipnfts(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    createdAt
    mintedAt
    chainId
    originalOwner
    owner {
      id
      address
    }
    metadata {
      name
      description
      image
      initialSymbol
      organization
      topic
      fundingAmount
      externalUrl
      researchLead {
        name
        email
      }
    }
    ipt {
      id
      metadata {
        symbol
        totalIssued
      }
    }
  }
}
```

**Parameters:**

| Parameter  | Type          | Description                                   |
| ---------- | ------------- | --------------------------------------------- |
| limit      | Int           | Maximum number of results (recommended: 20-50) |
| skip       | Int           | Number of results to skip (for pagination)    |
| sortBy     | IPNFTSortBy   | Field to sort by (e.g., `createdAt`, `mintedAt`) |
| sortOrder  | SortOrder     | Sort direction: `asc` or `desc`              |
| filterBy   | IPNFTFilterBy | Filter criteria (owner, chainId, etc.)       |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query ListIPNFTs($limit: Int, $skip: Int, $sortBy: IPNFTSortBy, $sortOrder: SortOrder) { ipnfts(limit: $limit, skip: $skip, sortBy: $sortBy, sortOrder: $sortOrder) { id createdAt owner { address } metadata { name description image topic organization } ipt { id metadata { symbol } } } }",
    "variables": {
      "limit": 20,
      "skip": 0,
      "sortBy": "createdAt",
      "sortOrder": "desc"
    }
  }'
```

**Response Example:**

```json
{
  "data": {
    "ipnfts": [
      {
        "id": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37_1",
        "createdAt": "2024-01-15T10:30:00.000Z",
        "owner": {
          "address": "0x1234567890123456789012345678901234567890"
        },
        "metadata": {
          "name": "Novel Cancer Immunotherapy Research",
          "description": "Groundbreaking CAR-T cell therapy development",
          "image": "ipfs://QmXnnyufdzAWL...",
          "topic": "Oncology",
          "organization": "Research Institute"
        },
        "ipt": {
          "id": "1_0xabcd...",
          "metadata": {
            "symbol": "CART-IPT"
          }
        }
      }
    ]
  }
}
```

***

### Get Single IP-NFT

Retrieve detailed information about a specific IP-NFT by its ID.

**GraphQL Query:**

```graphql
query GetIPNFT($id: ID!) {
  ipnft(id: $id) {
    id
    createdAt
    mintedAt
    chainId
    originalOwner
    owner {
      id
      address
    }
    metadata {
      name
      description
      image
      symbol
      initialSymbol
      organization
      topic
      fundingAmount
      externalUrl
      tokenUri
      researchLead {
        id
        name
        email
      }
      agreements
    }
    ipt {
      id
      l2TokenAddress
      holderCount
      metadata {
        symbol
        name
        decimals
        totalIssued
        circulatingSupply
      }
    }
  }
}
```

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query GetIPNFT($id: ID!) { ipnft(id: $id) { id metadata { name description topic } ipt { metadata { symbol totalIssued } } } }",
    "variables": {
      "id": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37_1"
    }
  }'
```

***

### List IP Tokens (IPTs)

Query and browse all IP Tokens with their associated IP-NFTs and market data.

**GraphQL Query:**

```graphql
query ListIPTs(
  $limit: Int
  $skip: Int
  $sortBy: IPTSortBy
  $sortOrder: SortOrder
  $filterBy: IPTFilterBy
) {
  ipts(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    createdAt
    l2TokenAddress
    holderCount
    ipnft {
      id
      owner {
        address
      }
      metadata {
        name
        description
        image
        topic
        organization
      }
    }
    metadata {
      symbol
      name
      decimals
      totalIssued
      circulatingSupply
      agreementCid
      image
    }
    markets {
      id
      name
      chainId
      pairAddress
      liquidityUsd
      tradingVolume24hr
      usdPrice
      usdPrice24hrPercentageChange
      marketCapUsd
    }
  }
}
```

**Parameters:**

| Parameter  | Type        | Description                                      |
| ---------- | ----------- | ------------------------------------------------ |
| limit      | Int         | Maximum number of results                        |
| skip       | Int         | Number of results to skip (for pagination)       |
| sortBy     | IPTSortBy   | Field to sort by (e.g., `createdAt`, `holderCount`) |
| sortOrder  | SortOrder   | Sort direction: `asc` or `desc`                 |
| filterBy   | IPTFilterBy | Filter criteria (ipnftId, l2TokenAddress, etc.) |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query ListIPTs($limit: Int, $sortBy: IPTSortBy, $sortOrder: SortOrder) { ipts(limit: $limit, sortBy: $sortBy, sortOrder: $sortOrder) { id metadata { symbol name totalIssued } markets { usdPrice liquidityUsd tradingVolume24hr } ipnft { metadata { name topic } } } }",
    "variables": {
      "limit": 20,
      "sortBy": "createdAt",
      "sortOrder": "desc"
    }
  }'
```

**Response Example:**

```json
{
  "data": {
    "ipts": [
      {
        "id": "1_0xabcdef...",
        "metadata": {
          "symbol": "CART-IPT",
          "name": "Cancer Research IP Token",
          "totalIssued": "1000000000000000000000000"
        },
        "markets": [
          {
            "usdPrice": 0.45,
            "liquidityUsd": 125000.50,
            "tradingVolume24hr": 8500.25
          }
        ],
        "ipnft": {
          "metadata": {
            "name": "Novel Cancer Immunotherapy Research",
            "topic": "Oncology"
          }
        }
      }
    ]
  }
}
```

***

### Get Single IP Token

Retrieve detailed information about a specific IPT by its ID.

**GraphQL Query:**

```graphql
query GetIPT($id: ID!) {
  ipt(id: $id) {
    id
    createdAt
    l2TokenAddress
    holderCount
    ipnft {
      id
      metadata {
        name
        description
        topic
      }
    }
    metadata {
      symbol
      name
      decimals
      totalIssued
      circulatingSupply
      agreementCid
    }
    markets {
      chainId
      name
      pairAddress
      liquidityUsd
      usdPrice
      marketCapUsd
    }
  }
}
```

***

### Query Markets

Access trading and market data for IP Tokens.

**GraphQL Query:**

```graphql
query ListMarkets(
  $limit: Int
  $skip: Int
  $sortBy: MarketSortBy
  $sortOrder: SortOrder
  $filterBy: MarketFilterBy
) {
  markets(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    name
    pairAddress
    chainId
    liquidityUsd
    tradingVolume24hr
    usdPrice
    usdPrice24hrPercentageChange
    marketCapUsd
    token {
      id
      metadata {
        symbol
        name
      }
      ipnft {
        metadata {
          name
          topic
        }
      }
    }
    chain {
      name
      chainId
    }
  }
}
```

**Example - Get Markets by Trading Volume:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "query ListMarkets($limit: Int, $sortBy: MarketSortBy, $sortOrder: SortOrder) { markets(limit: $limit, sortBy: $sortBy, sortOrder: $sortOrder) { name usdPrice liquidityUsd tradingVolume24hr token { metadata { symbol } } } }",
    "variables": {
      "limit": 10,
      "sortBy": "tradingVolume24hr",
      "sortOrder": "desc"
    }
  }'
```

***

## Common Patterns

### Pagination

Use `skip` and `limit` for pagination:

```javascript
// Page 1
{ "limit": 20, "skip": 0 }

// Page 2
{ "limit": 20, "skip": 20 }

// Page 3
{ "limit": 20, "skip": 40 }
```

### Sorting

Sort results by any field:

```javascript
{
  "sortBy": "createdAt",  // or "mintedAt", "updatedAt", etc.
  "sortOrder": "desc"      // or "asc"
}
```

### Filtering

Filter results by specific criteria:

**Filter by owner:**
```javascript
{
  "filterBy": {
    "userId": "0x1234567890123456789012345678901234567890"
  }
}
```

**Filter by chain:**
```javascript
{
  "filterBy": {
    "chainId": 1  // Ethereum mainnet
  }
}
```

**Filter IPTs by IPNFT:**
```javascript
{
  "filterBy": {
    "ipnftId": "0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_37_1"
  }
}
```

***

## Response Types

### IPNFT Type

```typescript
{
  id: String                    // Unique identifier
  createdAt: DateTime           // Creation timestamp
  mintedAt: DateTime            // Minting timestamp
  chainId: Int                  // Blockchain network ID
  originalOwner: String         // Original minter address
  owner: {                      // Current owner
    id: String
    address: String
  }
  metadata: {                   // Project metadata
    name: String
    description: String
    image: String               // IPFS URL
    symbol: String
    initialSymbol: String
    organization: String
    topic: String
    fundingAmount: JSON
    externalUrl: String
    researchLead: {
      name: String
      email: String
    }
  }
  ipt: {                        // Associated IP Token (if tokenized)
    id: String
    metadata: {
      symbol: String
      totalIssued: String
    }
  }
}
```

### IPT Type

```typescript
{
  id: String                    // Unique identifier
  createdAt: DateTime           // Creation timestamp
  l2TokenAddress: String        // ERC-20 contract address
  holderCount: Int              // Number of token holders
  ipnft: {                      // Parent IP-NFT
    id: String
    metadata: {
      name: String
      description: String
      topic: String
    }
  }
  metadata: {                   // Token metadata
    symbol: String
    name: String
    decimals: Int
    totalIssued: String         // Total supply (wei format)
    circulatingSupply: String
    agreementCid: String        // IPFS CID of membership agreement
  }
  markets: [{                   // Trading markets
    chainId: Int
    pairAddress: String
    liquidityUsd: Float
    usdPrice: Float
    tradingVolume24hr: Float
    marketCapUsd: Float
  }]
}
```

### Market Type

```typescript
{
  id: String                    // Unique identifier
  name: String                  // Market name
  pairAddress: String           // DEX pair contract address
  chainId: Int                  // Blockchain network ID
  liquidityUsd: Float           // Total liquidity in USD
  tradingVolume24hr: Float      // 24h trading volume in USD
  usdPrice: Float               // Current token price in USD
  usdPrice24hrPercentageChange: Float  // 24h price change %
  marketCapUsd: Float           // Market capitalization in USD
  token: {                      // Associated IPT
    id: String
    metadata: {
      symbol: String
      name: String
    }
  }
  chain: {                      // Blockchain info
    name: String
    chainId: Int
  }
}
```

***

## Example Use Cases

### Building a Marketplace UI

```javascript
// Fetch recent IP-NFTs with full metadata
const response = await fetch('https://production.graphql.api.molecule.xyz/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': process.env.API_KEY,
  },
  body: JSON.stringify({
    query: `
      query RecentIPNFTs {
        ipnfts(limit: 20, sortBy: "createdAt", sortOrder: "desc") {
          id
          metadata {
            name
            description
            image
            topic
            organization
          }
          ipt {
            id
            metadata { symbol }
          }
        }
      }
    `,
  }),
});

const data = await response.json();
// Display IP-NFTs in marketplace grid
```

### Token Screener / Price Tracker

```javascript
// Get top IPTs by trading volume
const response = await fetch('https://production.graphql.api.molecule.xyz/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': process.env.API_KEY,
  },
  body: JSON.stringify({
    query: `
      query TopIPTsByVolume {
        ipts(limit: 10, sortBy: "createdAt", sortOrder: "desc") {
          metadata {
            symbol
            name
          }
          markets {
            usdPrice
            usdPrice24hrPercentageChange
            tradingVolume24hr
            liquidityUsd
            marketCapUsd
          }
        }
      }
    `,
  }),
});

const data = await response.json();
// Display price table with 24h change indicators
```

### Portfolio Tracker

```javascript
// Get all IP-NFTs owned by a specific wallet
const walletAddress = "0x1234567890123456789012345678901234567890";

const response = await fetch('https://production.graphql.api.molecule.xyz/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': process.env.API_KEY,
  },
  body: JSON.stringify({
    query: `
      query UserPortfolio($filterBy: IPNFTFilterBy) {
        ipnfts(filterBy: $filterBy) {
          id
          metadata {
            name
            topic
          }
          ipt {
            metadata {
              symbol
              totalIssued
            }
            markets {
              usdPrice
              marketCapUsd
            }
          }
        }
      }
    `,
    variables: {
      filterBy: {
        userId: walletAddress
      }
    }
  }),
});

const data = await response.json();
// Calculate total portfolio value
```

***

## Error Handling

### Common Errors

| Status Code | Error                | Description                                  |
| ----------- | -------------------- | -------------------------------------------- |
| 401         | Unauthorized         | Missing or invalid API key                   |
| 400         | Bad Request          | Invalid query syntax or parameters           |
| 404         | Not Found            | Requested resource doesn't exist             |
| 500         | Internal Server Error| Server error - retry the request             |

### Troubleshooting

**401 Unauthorized Error:**
* Verify `x-api-key` header is included
* Check that your API key is valid and not expired
* Ensure no typos in the API key

**Empty Results:**
* Check filter criteria - may be too restrictive
* Verify the chainId if filtering by chain
* Try removing filters to see all results

**GraphQL Errors:**
* Check query syntax is valid
* Ensure field names match the schema
* Verify variable types match parameter types

***

## Getting Support

For questions or issues with the Data API:

* Join our [Discord community](https://t.co/L0VEiy4Bjk)
* Check the [API Overview](README.md) for authentication help
* Review the [GraphQL Schema](https://docs.molecule.to/) for all available fields

***

_Last updated: December 2024_
