---
icon: webhook
---

# Subgraph

## Molecule Subgraph Documentation

The Molecule Subgraph indexes all on-chain events from Molecule Protocol contracts, providing a fast GraphQL API for querying IP-NFTs, IPTs, crowdsales, and marketplace activity. It's powered by The Graph and requires no authentication.

### Quick Start

Query the subgraph directly with any GraphQL client:

```javascript
const response = await fetch(
  'https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-mainnet/api',
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: `{ ipts(first: 5) { id metadata { symbol name } } }`
    })
  }
)
const { data } = await response.json()
```

Or explore interactively in the [Playground](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-mainnet/playground).

#### Endpoints

* **Network: Mainnet**
  * API Endpoint: [https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-mainnet/api](https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-mainnet/api)
  * Playground: [https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-mainnet/playground](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-mainnet/playground)
* **Network: Sepolia**
  * API Endpoint: [https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-sepolia/api](https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-sepolia/api)
  * Playground: [https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-sepolia/playground](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-sepolia/playground)

### Entities

#### Core Entities

* **IPNFT**: IP-NFT tokens
  * Key Fields: `id`, `tokenId`, `owner`, `tokenURI`, `symbol`, `createdAt`
* **IPT**: Tokenized IP-NFTs (ERC-20)
  * Key Fields: `id`, `tokenContract`, `ipnft`, `metadata`, `totalIssued`, `capped`
* **IPTBalance**: User balances of IPTs
  * Key Fields: `id`, `ipt`, `holder`, `balance`
* **IpnftMetadata**: IPFS metadata
  * Key Fields: `id`, `name`, `image`, `external_url`, `properties`

#### Fundraising

* **CrowdSale**: Fundraising campaigns
  * Key Fields: `id`, `state`, `auctionToken`, `biddingToken`, `fundingGoal`, `closingTime`, `totalBids`
* **Contribution**: Individual bids
  * Key Fields: `id`, `crowdSale`, `contributor`, `amount`, `claimed`

#### Marketplace

* **Listing**: SchmackoSwap listings
  * Key Fields: `id`, `tokenContract`, `tokenId`, `creator`, `askPrice`, `state`

#### Vesting

* **TimelockedToken**: Vested token contracts
  * Key Fields: `id`, `underlyingToken`, `ipt`
* **LockedSchedule**: Vesting schedules
  * Key Fields: `id`, `beneficiary`, `amount`, `releaseTime`, `released`

### Common Queries

#### Finding IPToken Addresses

IPTokens are dynamically created through the Tokenizer. It is recommended not to hardcode IPToken addresses. Instead, use the methods outlined below to retrieve them.

#### On-Chain Query

To find an IPToken address on-chain, use the following method:

```solidity
IIPToken ipToken = Tokenizer.synthesized(ipnftId);
```

#### Subgraph Query

Alternatively, query via a subgraph using the query structure below:

```graphql
{
  ipt(where: { ipnftId: "123" }) {
    id
    tokenContract
  }
}
```

Follow the provided methods to dynamically obtain IPToken addresses, ensuring accuracy and adaptability in your implementations.

#### Get all IPTs

```graphql
query GetAllIPTs {
  ipts(first: 100, orderBy: createdAt, orderDirection: desc) {
    id
    tokenContract
    totalIssued
    capped
    ipnft {
      id
      owner
      tokenURI
    }
    metadata {
      name
      symbol
      image
    }
  }
}
```

#### Get IP-NFTs by owner

```graphql
query GetIPNFTsByOwner($owner: String!) {
  ipnfts(where: { owner: $owner }) {
    id
    tokenId
    owner
    tokenURI
    symbol
    createdAt
  }
}
```

#### Get user's IPT balances

```graphql
query GetUserBalances($holder: String!) {
  iptbalances(where: { holder: $holder, balance_gt: "0" }) {
    balance
    ipt {
      tokenContract
      metadata {
        symbol
        name
      }
    }
  }
}
```

#### Get crowdsale with contributions

```graphql
query GetCrowdSale($id: ID!) {
  crowdSale(id: $id) {
    id
    state
    fundingGoal
    salesAmount
    closingTime
    totalBids
    auctionToken
    biddingToken
    contributions(first: 100, orderBy: amount, orderDirection: desc) {
      contributor
      amount
      claimed
    }
  }
}
```

#### Get active marketplace listings

```graphql
query GetActiveListings {
  listings(where: { state: LISTED }, orderBy: askPrice, orderDirection: asc) {
    id
    tokenId
    creator
    askPrice
    paymentToken
    tokenContract
  }
}
```

### JavaScript Client

A minimal client for querying the subgraph:

{% code overflow="wrap" %}
```javascript
const MAINNET = 'https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-mainnet/api'
const SEPOLIA = 'https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-sepolia/api'

async function query(endpoint, graphql, variables = {}) {
  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: graphql, variables }),
  })
  const { data, errors } = await response.json() 
  if (errors) throw new Error(errors[0].message) 
  return data
}

// Examples
const { ipts } = await query(MAINNET, `{ ipts(first: 10) { id metadata { symbol } } }`)
const { ipnfts } = await query(MAINNET,
  `query($owner: String!) { ipnfts(where: { owner: $owner }) { id tokenId } }`,
  { owner: '0x...' }
)
```
{% endcode %}

### Pagination

The subgraph limits results to 1000 items per query. For larger datasets, use cursor-based pagination:

```graphql
query GetAllIPTs($lastId: String) {
  ipts(
    first: 1000
    where: { id_gt: $lastId }
    orderBy: id
    orderDirection: asc
  ) {
    id
    metadata { symbol }
  }
}
```

```javascript
async function getAllIPTs() {
  let all = []
  let lastId = ''
  while (true) {
    const { ipts } = await query(MAINNET, GET_ALL_IPTS, { lastId })
    if (ipts.length === 0) break
    all = all.concat(ipts)
    lastId = ipts[ipts.length - 1].id
  }
  return all
}
```

### Filtering & Sorting

#### Filter operators

* **Exact match**: `ipts(where: { capped: true })`
* **Comparison**: `crowdSales(where: { fundingGoal_gte: "1000000000000000000" })`
* **Multiple conditions (AND)**: `contributions(where: { claimed: false, amount_gt: "0" })`
* **Pattern matching**: `ipnfts(where: { symbol_contains: "BIO" })`

**Available operators**

| Operator    | Example                        | Description      |
| ----------- | ------------------------------ | ---------------- |
| `_eq`       | owner: "0x..."                 | Equals (default) |
| `_not`      | state\_not: FAILED             | Not equals       |
| `_gt/_gte`  | amount\_gte: "1000"            | Greater than     |
| `_lt/_lte`  | closingTime\_lt: 1700000000    | Less than        |
| `_in`       | state\_in: \[RUNNING, SETTLED] | In list          |
| `_contains` | symbol\_contains: "VITA"       | String contains  |

#### Sorting

* `ipts(orderBy: createdAt, orderDirection: desc)`
* `contributions(orderBy: amount, orderDirection: desc)`

### Rate Limits

| Limit            | Value                |
| ---------------- | -------------------- |
| Requests         | 1,000/minute         |
| Max results      | 1,000/query          |
| Query complexity | Limited by The Graph |

No authentication required.

### Resources

* [Playground (Mainnet)](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-mainnet/playground) — Interactive query explorer
* [Playground (Sepolia)](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-sepolia/playground) — Testnet explorer
* [SDKS/Client-SDK](https://sdks/client-sdk) — Typed wrapper with caching
* [The Graph Documentation](https://thegraph.com/docs/) — GraphQL query reference
