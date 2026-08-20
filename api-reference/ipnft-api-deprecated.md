# 📊 IPNFT API (Deprecated)

## Overview

The IPNFT API provides read-only access to query and browse intellectual property assets across the Molecule Protocol. Use these queries to build marketplaces, token screeners, portfolio trackers, and discovery interfaces for decentralized science projects.

**Features:**

- Query IP-NFTs (Intellectual Property NFTs) and their project details
- Browse IP Tokens (IPTs) with market data
- Access trading metrics and liquidity information
- Query users, research leads, chains, and agreements
- Filter, sort, and paginate results
- Build data-driven applications

---

## Authentication

All IPNFT API requests require a consumer credential (see [Authentication](authentication.md)).

### Obtaining a Consumer Credential

To request one:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team
3. Provide your intended use case
4. You'll receive a consumer credential (`mol_<consumerId>_<secret>`)

### Using Your Consumer Credential

Send it as the `Authorization` header value directly — **no `Bearer` prefix**:

```bash
Authorization: mol_<consumerId>_<secret>
```

---

## API Endpoint

```
Production: https://production.graphql.api.molecule.xyz/graphql
Staging:    https://staging.graphql.api.molecule.xyz/graphql
```

---

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
    updatedAt
    mintedAt
    chainId
    originalOwner
    tokenUri
    symbol
    name
    description
    image
    externalUrl
    initialSymbol
    organization
    topic
    trlValue
    trlRationale
    fundingAmountCurrency
    fundingAmountValue
    fundingAmountDecimals
    fundingAmountCurrencyType
    schemaVersion
    owner {
      id
      address
    }
    researchLead {
      name
      email
    }
    agreements {
      id
      contentHash
      mimeType
      type
      url
    }
    ipt {
      id
      symbol
      totalIssued
    }
  }
}
```

**Parameters:**

| Parameter | Type          | Description                                                                             |
| --------- | ------------- | --------------------------------------------------------------------------------------- |
| limit     | Int           | Maximum number of results (recommended: 20-50)                                          |
| skip      | Int           | Number of results to skip (for pagination)                                              |
| sortBy    | IPNFTSortBy   | Field to sort by (e.g., `createdAt`, `mintedAt`, `name`, `topic`, `fundingAmountValue`) |
| sortOrder | SortOrder     | Sort direction: `asc` or `desc`                                                         |
| filterBy  | IPNFTFilterBy | Filter criteria (owner, chainId, topic, etc.)                                           |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query ListIPNFTs($limit: Int, $skip: Int, $sortBy: IPNFTSortBy, $sortOrder: SortOrder) { ipnfts(limit: $limit, skip: $skip, sortBy: $sortBy, sortOrder: $sortOrder) { id createdAt owner { address } name description image topic organization ipt { id symbol } } }",
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
        "id": "37",
        "createdAt": "2024-01-15T10:30:00.000Z",
        "owner": {
          "address": "0x1234567890123456789012345678901234567890"
        },
        "name": "Novel Cancer Immunotherapy Research",
        "description": "Groundbreaking CAR-T cell therapy development",
        "image": "ipfs://QmXnnyufdzAWL...",
        "topic": "Oncology",
        "organization": "Research Institute",
        "ipt": {
          "id": "0xabcd...",
          "symbol": "CART-IPT"
        }
      }
    ]
  }
}
```

---

### Get Single IP-NFT

Retrieve detailed information about a specific IP-NFT by its ID.

**GraphQL Query:**

```graphql
query GetIPNFT($id: ID!) {
  ipnft(id: $id) {
    id
    createdAt
    updatedAt
    mintedAt
    chainId
    originalOwner
    tokenUri
    symbol
    name
    description
    image
    externalUrl
    initialSymbol
    organization
    topic
    trlValue
    trlRationale
    fundingAmountCurrency
    fundingAmountValue
    fundingAmountDecimals
    fundingAmountCurrencyType
    schemaVersion
    userId
    owner {
      id
      address
    }
    researchLead {
      id
      name
      email
    }
    agreements {
      id
      contentHash
      mimeType
      type
      url
    }
    ipt {
      id
      l2TokenAddress
      holderCount
      symbol
      name
      decimals
      totalIssued
      circulatingSupply
    }
  }
}
```

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query GetIPNFT($id: ID!) { ipnft(id: $id) { id name description topic symbol owner { address } ipt { symbol totalIssued } agreements { id type url } } }",
    "variables": {
      "id": "37"
    }
  }'
```

---

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
    updatedAt
    mintedAt
    l2TokenAddress
    holderCount
    symbol
    name
    decimals
    totalIssued
    circulatingSupply
    agreementCid
    agreementMimeType
    image
    links
    capped
    ipnftId
    originalOwnerId
    ipnft {
      id
      name
      description
      image
      topic
      organization
      owner {
        address
      }
    }
    originalOwner {
      id
      address
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

| Parameter | Type        | Description                                                           |
| --------- | ----------- | --------------------------------------------------------------------- |
| limit     | Int         | Maximum number of results                                             |
| skip      | Int         | Number of results to skip (for pagination)                            |
| sortBy    | IPTSortBy   | Field to sort by (e.g., `createdAt`, `holderCount`, `name`, `symbol`) |
| sortOrder | SortOrder   | Sort direction: `asc` or `desc`                                       |
| filterBy  | IPTFilterBy | Filter criteria (ipnftId, symbol, originalOwnerId, etc.)              |

**Example Request (curl):**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query ListIPTs($limit: Int, $sortBy: IPTSortBy, $sortOrder: SortOrder) { ipts(limit: $limit, sortBy: $sortBy, sortOrder: $sortOrder) { id symbol name totalIssued markets { usdPrice liquidityUsd tradingVolume24hr } ipnft { name topic } } }",
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
        "id": "0xabcdef...",
        "symbol": "CART-IPT",
        "name": "Cancer Research IP Token",
        "totalIssued": "1000000000000000000000000",
        "markets": [
          {
            "usdPrice": 0.45,
            "liquidityUsd": 125000.5,
            "tradingVolume24hr": 8500.25
          }
        ],
        "ipnft": {
          "name": "Novel Cancer Immunotherapy Research",
          "topic": "Oncology"
        }
      }
    ]
  }
}
```

---

### Get Single IP Token

Retrieve detailed information about a specific IPT by its ID.

**GraphQL Query:**

```graphql
query GetIPT($id: ID!) {
  ipt(id: $id) {
    id
    createdAt
    updatedAt
    mintedAt
    l2TokenAddress
    holderCount
    symbol
    name
    decimals
    totalIssued
    circulatingSupply
    agreementCid
    agreementMimeType
    image
    links
    capped
    ipnft {
      id
      name
      description
      topic
    }
    originalOwner {
      id
      address
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

---

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
    createdAt
    updatedAt
    name
    pairAddress
    chainId
    liquidityUsd
    tradingVolume24hr
    usdPrice
    usdPrice24hrPercentageChange
    marketCapUsd
    inverted
    iptId
    token {
      id
      symbol
      name
      ipnft {
        name
        topic
      }
    }
    chain {
      name
      chainId
      logoUrl
    }
  }
}
```

**Example - Get Markets by Trading Volume:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -d '{
    "query": "query ListMarkets($limit: Int, $sortBy: MarketSortBy, $sortOrder: SortOrder) { markets(limit: $limit, sortBy: $sortBy, sortOrder: $sortOrder) { name usdPrice liquidityUsd tradingVolume24hr token { symbol } chain { name logoUrl } } }",
    "variables": {
      "limit": 10,
      "sortBy": "tradingVolume24hr",
      "sortOrder": "desc"
    }
  }'
```

A single market can also be fetched by its id:

```graphql
query GetMarket($id: ID!) {
  market(id: $id) {
    name
    usdPrice
    liquidityUsd
    tradingVolume24hr
    token {
      symbol
    }
  }
}
```

---

### Query Users

Query users and their associated IP-NFTs and IPTs.

**GraphQL Query:**

```graphql
query ListUsers(
  $limit: Int
  $skip: Int
  $sortBy: UserSortBy
  $sortOrder: SortOrder
  $filterBy: UserFilterBy
) {
  users(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    createdAt
    updatedAt
    address
    ipnfts {
      id
      name
      topic
    }
    ipts {
      id
      symbol
      name
    }
  }
}
```

**Get Single User:**

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    address
    createdAt
    updatedAt
    ipnfts {
      id
      name
      topic
      organization
    }
    ipts {
      id
      symbol
      name
      totalIssued
    }
  }
}
```

---

### Query Research Leads

Query research leads associated with IP-NFTs.

**GraphQL Query:**

```graphql
query ListResearchLeads(
  $limit: Int
  $skip: Int
  $sortBy: ResearchLeadSortBy
  $sortOrder: SortOrder
  $filterBy: ResearchLeadFilterBy
) {
  researchLeads(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    createdAt
    updatedAt
    name
    email
    ipnfts {
      id
      name
      topic
    }
  }
}
```

**Get Single Research Lead:**

```graphql
query GetResearchLead($id: ID!) {
  researchLead(id: $id) {
    id
    name
    email
    createdAt
    updatedAt
    ipnfts {
      id
      name
      topic
      organization
    }
  }
}
```

---

### Query Chains

Query blockchain networks where markets are deployed.

**GraphQL Query:**

```graphql
query ListChains(
  $limit: Int
  $skip: Int
  $sortBy: ChainSortBy
  $sortOrder: SortOrder
  $filterBy: ChainFilterBy
) {
  chains(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    createdAt
    updatedAt
    name
    chainId
    logoUrl
    markets {
      id
      name
      usdPrice
      liquidityUsd
    }
  }
}
```

**Get Single Chain:**

```graphql
query GetChain($id: ID!) {
  chain(id: $id) {
    id
    name
    chainId
    logoUrl
    createdAt
    updatedAt
    markets {
      id
      name
      usdPrice
      liquidityUsd
      tradingVolume24hr
    }
  }
}
```

---

### Query Agreements

Query legal agreements associated with IP-NFTs.

**GraphQL Query:**

```graphql
query ListAgreements(
  $limit: Int
  $skip: Int
  $sortBy: AgreementSortBy
  $sortOrder: SortOrder
  $filterBy: AgreementFilterBy
) {
  agreements(
    limit: $limit
    skip: $skip
    sortBy: $sortBy
    sortOrder: $sortOrder
    filterBy: $filterBy
  ) {
    id
    contentHash
    mimeType
    type
    url
    ipnftId
  }
}
```

**Get Single Agreement:**

```graphql
query GetAgreement($id: ID!) {
  agreement(id: $id) {
    id
    contentHash
    mimeType
    type
    url
    ipnftId
  }
}
```

---

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
  "sortBy": "createdAt",  // or "mintedAt", "updatedAt", "name", "topic", etc.
  "sortOrder": "desc"      // or "asc"
}
```

### Filtering

Filter results by specific criteria. The API supports both direct field filtering and nested relation filtering.

#### Basic Filtering

**Filter IP-NFTs by topic:**

```javascript
{
  "filterBy": {
    "topic": "Oncology"
  }
}
```

**Filter by owner (using user ID):**

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

**Filter IPTs by symbol:**

```javascript
{
  "filterBy": {
    "symbol": "VITA"
  }
}
```

**Filter IPTs by IPNFT:**

```javascript
{
  "filterBy": {
    "ipnftId": "37"
  }
}
```

#### Nested Relation Filtering

The API supports filtering by nested relation properties for more flexible queries.

**Filter IP-NFTs by owner address:**

```javascript
{
  "filterBy": {
    "owner": {
      "address": "0x1234567890123456789012345678901234567890"
    }
  }
}
```

**Filter IP-NFTs by owner ID:**

```javascript
{
  "filterBy": {
    "owner": {
      "id": "0x1234567890123456789012345678901234567890"
    }
  }
}
```

**Filter IP-NFTs by research lead:**

```javascript
{
  "filterBy": {
    "researchLead": {
      "email": "researcher@university.edu"
    }
  }
}
```

**Filter IP-NFTs by agreement properties:**

```javascript
{
  "filterBy": {
    "agreements": {
      "mimeType": "application/pdf"
    }
  }
}
```

**Filter IPTs by original owner:**

```javascript
{
  "filterBy": {
    "originalOwner": {
      "address": "0x1234567890123456789012345678901234567890"
    }
  }
}
```

**Filter IPTs by parent IPNFT properties:**

```javascript
{
  "filterBy": {
    "ipnft": {
      "topic": "Oncology"
    }
  }
}
```

**Filter markets by chain properties:**

```javascript
{
  "filterBy": {
    "chain": {
      "chainId": 1  // Ethereum mainnet
    }
  }
}
```

**Filter markets by token properties:**

```javascript
{
  "filterBy": {
    "token": {
      "symbol": "VITA-IPT"
    }
  }
}
```

**Filter IPTs by IPNFT owner (deeply nested):**

```javascript
{
  "filterBy": {
    "ipnft": {
      "owner": {
        "address": "0x1234567890123456789012345678901234567890"
      }
    }
  }
}
```

#### Combining Filters

You can combine multiple filters in a single query. All filters are combined with AND logic - results must match all criteria.

**Combine scalar and relation filters:**

```javascript
{
  "filterBy": {
    "chainId": 1,
    "owner": {
      "address": "0x1234567890123456789012345678901234567890"
    }
  }
}
```

**Combine multiple field filters:**

```javascript
{
  "filterBy": {
    "topic": "Oncology",
    "organization": "University Lab",
    "owner": {
      "address": "0x1234567890123456789012345678901234567890"
    }
  }
}
```

**Combine nested relation filters (IPT query):**

```javascript
{
  "filterBy": {
    "symbol": "VITA",
    "ipnft": {
      "owner": {
        "address": "0x1234567890123456789012345678901234567890"
      },
      "topic": "Longevity"
    }
  }
}
```

---

## Response Types

### IPNFT Type

```typescript
{
  id: String                    // Unique identifier — the onchain tokenId as a string (e.g. "37")
  oclId: String                 // Linked lab oclId, null when the IP-NFT has no linked lab
  createdAt: DateTime           // Creation timestamp
  updatedAt: DateTime           // Last update timestamp
  mintedAt: DateTime            // Minting timestamp
  chainId: Int                  // Blockchain network ID
  originalOwner: String         // Original minter address
  tokenUri: String              // Token metadata URI
  symbol: String                // Token symbol
  name: String                  // Project name
  description: String           // Project description
  image: String                 // IPFS image URL
  externalUrl: String           // External project URL
  initialSymbol: String         // Initial token symbol
  organization: String          // Organization name
  topic: String                 // Research topic
  trlValue: String              // Technology readiness levels value
  trlRationale: String          // Technology readiness levels rationale
  fundingAmountCurrency: String // Funding currency code
  fundingAmountValue: String    // Funding amount value
  fundingAmountDecimals: Int    // Funding currency decimals
  fundingAmountCurrencyType: String // Currency type (e.g., "ERC20", "native")
  schemaVersion: String         // Metadata schema version
  userId: String                // Owner user ID
  researchLeadId: String        // Research lead ID
  owner: {                      // Current owner
    id: String
    address: String
    createdAt: DateTime
    updatedAt: DateTime
  }
  researchLead: {               // Research lead
    id: String
    name: String
    email: String
  }
  agreements: [{                // Legal agreements
    id: String
    contentHash: String
    mimeType: String
    type: String
    url: String
    ipnftId: String
  }]
  ipt: {                        // Associated IP Token (if tokenized)
    id: String
    symbol: String
    totalIssued: String
  }
}
```

### IPT Type

```typescript
{
  id: String                    // Unique identifier
  createdAt: DateTime           // Creation timestamp
  updatedAt: DateTime           // Last update timestamp
  mintedAt: DateTime            // Minting timestamp
  l2TokenAddress: String        // ERC-20 contract address
  holderCount: Int              // Number of token holders
  symbol: String                // Token symbol
  name: String                  // Token name
  decimals: Int                 // Token decimals
  totalIssued: String           // Total supply (wei format)
  circulatingSupply: String     // Circulating supply
  agreementCid: String          // IPFS CID of membership agreement
  agreementMimeType: String     // Agreement file MIME type
  image: String                 // Token image URL
  links: [String]               // Related links
  capped: Boolean               // Whether token issuance is capped
  ipnftId: String               // Parent IP-NFT ID
  originalOwnerId: String       // Original owner user ID
  ipnft: {                      // Parent IP-NFT
    id: String
    name: String
    description: String
    topic: String
  }
  originalOwner: {              // Original token owner
    id: String
    address: String
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
  id: String; // Unique identifier
  createdAt: DateTime; // Creation timestamp
  updatedAt: DateTime; // Last update timestamp
  name: String; // Market name
  pairAddress: String; // DEX pair contract address
  chainId: Int; // Blockchain network ID
  liquidityUsd: Float; // Total liquidity in USD
  tradingVolume24hr: Float; // 24h trading volume in USD
  usdPrice: Float; // Current token price in USD
  usdPrice24hrPercentageChange: Float; // 24h price change %
  marketCapUsd: Float; // Market capitalization in USD
  inverted: Boolean; // Whether the pair is inverted
  iptId: String; // Associated IPT ID
  token: {
    // Associated IPT
    id: String;
    symbol: String;
    name: String;
  }
  chain: {
    // Blockchain info
    id: Int;
    name: String;
    chainId: Int;
    logoUrl: String;
  }
}
```

### User Type

```typescript
{
  id: String; // Unique identifier
  createdAt: DateTime; // Creation timestamp
  updatedAt: DateTime; // Last update timestamp
  address: String; // Wallet address
  ipnfts: [IPNFT]; // Owned IP-NFTs
  ipts: [IPT]; // Owned IPTs
}
```

### ResearchLead Type

```typescript
{
  id: String; // Unique identifier
  createdAt: DateTime; // Creation timestamp
  updatedAt: DateTime; // Last update timestamp
  name: String; // Research lead name
  email: String; // Research lead email
  ipnfts: [IPNFT]; // Associated IP-NFTs
}
```

### Chain Type

```typescript
{
  id: Int; // Unique identifier
  createdAt: DateTime; // Creation timestamp
  updatedAt: DateTime; // Last update timestamp
  name: String; // Chain name
  chainId: Int; // Blockchain network ID (e.g., 1 for Ethereum)
  logoUrl: String; // Chain logo URL
  markets: [Market]; // Markets on this chain
}
```

### Agreement Type

```typescript
{
  id: String; // Unique identifier
  contentHash: String; // Content hash
  mimeType: String; // File MIME type
  type: String; // Agreement type
  url: String; // Agreement URL
  ipnftId: String; // Parent IP-NFT ID
}
```

---

## Example Use Cases

### Building a Marketplace UI

```javascript
// Fetch recent IP-NFTs with full details
const response = await fetch(
  "https://production.graphql.api.molecule.xyz/graphql",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": process.env.CONSUMER_CREDENTIAL,
    },
    body: JSON.stringify({
      query: `
      query RecentIPNFTs {
        ipnfts(limit: 20, sortBy: createdAt, sortOrder: desc) {
          id
          name
          description
          image
          topic
          organization
          ipt {
            id
            symbol
          }
        }
      }
    `,
    }),
  },
);

const data = await response.json();
// Display IP-NFTs in marketplace grid
```

### Token Screener / Price Tracker

```javascript
// Get top IPTs by trading volume
const response = await fetch(
  "https://production.graphql.api.molecule.xyz/graphql",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": process.env.CONSUMER_CREDENTIAL,
    },
    body: JSON.stringify({
      query: `
      query TopIPTsByVolume {
        ipts(limit: 10, sortBy: createdAt, sortOrder: desc) {
          symbol
          name
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
  },
);

const data = await response.json();
// Display price table with 24h change indicators
```

### Portfolio Tracker

```javascript
// Get all IP-NFTs owned by a specific wallet
const walletAddress = "0x1234567890123456789012345678901234567890";

const response = await fetch(
  "https://production.graphql.api.molecule.xyz/graphql",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": process.env.CONSUMER_CREDENTIAL,
    },
    body: JSON.stringify({
      query: `
      query UserPortfolio($filterBy: IPNFTFilterBy) {
        ipnfts(filterBy: $filterBy) {
          id
          name
          topic
          ipt {
            symbol
            totalIssued
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
          owner: {
            address: walletAddress,
          },
        },
      },
    }),
  },
);

const data = await response.json();
// Calculate total portfolio value
```

---

## Advanced Filtering

### Relation Filtering vs Direct Filtering

The IPNFT API supports two approaches to filtering:

1. **Direct Field Filtering**: Filter by the ID of a related entity
2. **Relation Filtering**: Filter by properties of related entities

Both approaches work and can be used based on your needs.

**Example - Finding IP-NFTs by Owner:**

```javascript
// Approach 1: Direct field filtering (when you know the user ID)
{
  "filterBy": {
    "userId": "0x1234..."
  }
}

// Approach 2: Relation filtering (when you want to filter by owner properties)
{
  "filterBy": {
    "owner": {
      "address": "0x1234..."
    }
  }
}
```

### Multi-Level Nested Filtering

You can filter through multiple levels of relations:

```javascript
// Find all IP Tokens whose parent IP-NFT is owned by a specific wallet
{
  "filterBy": {
    "ipnft": {
      "owner": {
        "address": "0x1234567890123456789012345678901234567890"
      }
    }
  }
}

// Find markets for tokens with a specific symbol
{
  "filterBy": {
    "token": {
      "symbol": "VITA-IPT"
    }
  }
}
```

### Available Relation Filters

| Query Type | Relation Field  | Supported Filters                              | Example                                       |
| ---------- | --------------- | ---------------------------------------------- | --------------------------------------------- |
| `ipnfts`   | `owner`         | `id`, `address`                                | `owner: { address: "0x..." }`                 |
| `ipnfts`   | `researchLead`  | `id`, `name`, `email`                          | `researchLead: { email: "..." }`              |
| `ipnfts`   | `agreements`    | `id`, `contentHash`, `mimeType`, `type`, `url` | `agreements: { mimeType: "application/pdf" }` |
| `ipts`     | `ipnft`         | All IPNFT filter fields                        | `ipnft: { topic: "Oncology" }`                |
| `ipts`     | `originalOwner` | `id`, `address`                                | `originalOwner: { address: "0x..." }`         |
| `markets`  | `chain`         | `id`, `chainId`, `name`                        | `chain: { chainId: 1 }`                       |
| `markets`  | `token`         | All IPT filter fields                          | `token: { symbol: "VITA" }`                   |

### Filter Matching

All filters use **exact equality matching** by default. For example:

```javascript
{
  "filterBy": {
    "topic": "Longevity"  // Exact match only
  }
}
```

---

## Error Handling

### Common Errors

| Status Code | Error                 | Description                        |
| ----------- | --------------------- | ---------------------------------- |
| 401         | Unauthorized          | Missing or invalid consumer credential |
| 400         | Bad Request           | Invalid query syntax or parameters |
| 500         | Internal Server Error | Server error - retry the request   |

Missing resources are **not** signalled with an HTTP 404. Single-item queries (`ipnft`, `ipt`, `user`, …) return HTTP 200 with a GraphQL error in the `errors[]` array carrying a machine-readable `code`:

| GraphQL error `code`        | Meaning                                                     |
| --------------------------- | ----------------------------------------------------------- |
| `NOT_FOUND`                 | Requested resource doesn't exist                            |
| `COMPLEXITY_LIMIT_EXCEEDED` | Query too complex — max depth **5**, max **100** selections |
| `INVALID_INPUT`             | Malformed arguments                                         |

### Troubleshooting

**401 Unauthorized Error:**

- Verify the `Authorization` header is included, with no `Bearer` prefix
- Check that your consumer credential is valid and not expired
- Ensure no typos in the consumer credential

**Empty Results:**

- Check filter criteria - may be too restrictive
- Verify the chainId if filtering by chain
- Try removing filters to see all results

**GraphQL Errors:**

- Check query syntax is valid
- Ensure field names match the schema
- Verify variable types match parameter types

---

## Getting Support

For questions or issues with the IPNFT API:

- Join our [Discord community](https://t.co/L0VEiy4Bjk)
- Check the [API Overview](README.md) for authentication help

---

## Recent Updates

The breaking changes, migration notes, and newly added fields for this API have moved to the [API Changelog & Migration](changelog.md#ipnft-api-deprecated) page (February 2026 changes).

---

_Last updated: February 2026_
