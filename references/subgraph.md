---
icon: webhook
---

# Subgraph (Deprecated)

## Molecule Subgraph Documentation

The Molecule Subgraph indexes onchain events from the legacy IP-NFT protocol contracts (IP-NFTs, IPTs, crowdsales, marketplace activity). It's powered by The Graph and requires no authentication. The subgraph source lives in the [IPNFT repository](https://github.com/moleculeprotocol/IPNFT/tree/main/subgraph).

{% hint style="warning" %}
**Service status:** This service is deprecated. For indexed IP-NFT, IPT, and market data, use the actively maintained [Data API](../api-reference/data-api.md) instead. The entity reference below matches the subgraph schema source and remains valid for self-hosted deployments.
{% endhint %}

### Quick Start

Query the subgraph directly with any GraphQL client:

```javascript
const response = await fetch(
  'https://subgraph.satsuma-prod.com/742d8952ab24/molecule--4039244/ip-nft-mainnet/api',
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: `{ ipts(first: 5) { id symbol name } }`
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

* **Ipnft**: IP-NFT tokens
  * Key Fields: `id` (the tokenId), `owner`, `tokenURI`, `symbol`, `createdAt`, `metadata`, `ipToken`
* **IPT**: IP Tokens (ERC-20)
  * Key Fields: `id` (the IPT contract address), `name`, `symbol`, `agreementCid`, `totalIssued`, `circulatingSupply`, `capped`, `ipnft`, `createdAt`
* **IPTBalance**: User balances of IPTs
  * Key Fields: `id`, `ipt`, `owner`, `balance`, `agreementSignature`
* **IpnftMetadata**: IPFS metadata
  * Key Fields: `id`, `name`, `description`, `image`, `externalURL`, `organization`, `topic`, `fundingAmount_*`

#### Fundraising

* **CrowdSale**: Fundraising campaigns
  * Key Fields: `id`, `state`, `type`, `ipt`, `biddingToken`, `salesAmount`, `fundingGoal`, `amountRaised`, `closingTime`, `contract`
* **Contribution**: Individual bids
  * Key Fields: `id`, `crowdSale`, `contributor`, `amount`, `stakedAmount`, `claimedAt`

#### Marketplace

* **Listing**: SchmackoSwap listings
  * Key Fields: `id`, `ipnft`, `creator`, `paymentToken`, `beneficiary`, `askPrice`, `createdAt`, `unlistedAt`, `purchasedAt`, `buyer`

#### Vesting

* **TimelockedToken**: Vested token contracts
  * Key Fields: `id`, `underlyingToken`, `ipt`, `symbol`, `decimals`
* **LockedSchedule**: Vesting schedules
  * Key Fields: `id`, `tokenContract`, `beneficiary`, `amount`, `expiresAt`, `claimedAt`

### Common Queries

#### Finding IPToken Addresses

IPToken contract addresses are dynamic. It is recommended not to hardcode IPToken addresses. Instead, retrieve them via the subgraph:

#### Subgraph Query

```graphql
{
  ipts(where: { ipnft: "123" }) {
    id
  }
}
```

The IPT's `id` **is** its ERC-20 contract address — there is no separate `tokenContract` field.

#### Get all IPTs

```graphql
query GetAllIPTs {
  ipts(first: 100, orderBy: createdAt, orderDirection: desc) {
    id
    name
    symbol
    totalIssued
    circulatingSupply
    capped
    ipnft {
      id
      owner
      tokenURI
    }
  }
}
```

#### Get IP-NFTs by owner

```graphql
query GetIPNFTsByOwner($owner: String!) {
  ipnfts(where: { owner: $owner }) {
    id
    owner
    tokenURI
    symbol
    createdAt
  }
}
```

#### Get user's IPT balances

```graphql
query GetUserBalances($owner: String!) {
  iptbalances(where: { owner: $owner, balance_gt: "0" }) {
    balance
    ipt {
      id
      symbol
      name
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
    type
    fundingGoal
    amountRaised
    salesAmount
    closingTime
    ipt { id symbol }
    biddingToken { id symbol }
    contributions(first: 100, orderBy: amount, orderDirection: desc) {
      contributor
      amount
      claimedAt
    }
  }
}
```

#### Get open marketplace listings

Listings have no `state` field — an open listing is one that has been neither purchased nor unlisted:

```graphql
query GetOpenListings {
  listings(
    where: { purchasedAt: null, unlistedAt: null }
    orderBy: askPrice
    orderDirection: asc
  ) {
    id
    ipnft { id }
    creator
    askPrice
    paymentToken
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
const { ipts } = await query(MAINNET, `{ ipts(first: 10) { id symbol } }`)
const { ipnfts } = await query(MAINNET,
  `query($owner: String!) { ipnfts(where: { owner: $owner }) { id tokenURI } }`,
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
    symbol
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
* **Multiple conditions (AND)**: `contributions(where: { claimedAt: null, amount_gt: "0" })`
* **Pattern matching**: `ipnfts(where: { symbol_contains: "BIO" })`

**Available operators**

| Operator    | Example                        | Description                       |
| ----------- | ------------------------------ | --------------------------------- |
| (none)      | owner: "0x..."                 | Equals — the bare field name      |
| `_not`      | state\_not: FAILED             | Not equals                        |
| `_gt/_gte`  | amount\_gte: "1000"            | Greater than                      |
| `_lt/_lte`  | closingTime\_lt: 1700000000    | Less than                         |
| `_in`       | state\_in: \[RUNNING, SETTLED] | In list                           |
| `_contains` | symbol\_contains: "VITA"       | String contains                   |

#### Sorting

* `ipts(orderBy: createdAt, orderDirection: desc)`
* `contributions(orderBy: amount, orderDirection: desc)`

### Rate Limits

| Limit            | Value                |
| ---------------- | -------------------- |
| Max results      | 1,000/query          |
| Query complexity | Limited by The Graph |

No authentication required.

### Resources

* [Playground (Mainnet)](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-mainnet/playground) — Interactive query explorer
* [Playground (Sepolia)](https://subgraph.satsuma-prod.com/molecule--4039244/ip-nft-sepolia/playground) — Testnet explorer
* [The Graph Documentation](https://thegraph.com/docs/) — GraphQL query reference
