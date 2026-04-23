---
description: >-
  How Lab data is indexed, aggregated, served, and consumed by the platform, AI
  tools, and external integrations
icon: computer-classic
---

# Data API & Integration

### The Data Access Layer

The Data Storage and Data Privacy & Access pages describe how research files enter an Onchain Lab — how they are encrypted, stored, versioned, and access-controlled. This page describes the other side of the pipeline: how on-chain events are indexed into a queryable database, how that data is aggregated with research metadata and editorial content into a unified API layer, and how every consumer in the ecosystem — the frontend, MIRA, external developers, and AI agents — accesses it.

Every interaction a user has with Lab data on the Molecule platform — browsing project listings, viewing token prices, reading announcements, querying MIRA, or building an integration with the SDK — is mediated by the DeSci API. The API is the central data hub that sits between the raw data sources and the consumers that need them.

### Data Sources

The DeSci API aggregates data from multiple backends, each responsible for a different category of information.

**Amazon Aurora (AWS relational database)** is the primary data store for indexed on-chain and application data. Aurora is a cloud-native, fully managed relational database service from AWS. It stores IP-NFT ownership records, IPT contract state, token metadata, treasury balances, crowdsale contributions, Lab creation records, and transaction history — all pre-indexed into relational tables so the API can serve queries without reading directly from the blockchain at request time.

**Kamu** provides the research data layer. Every file uploaded to a Lab's data room — its content hash, version history, author DID, access level, encryption metadata, and provenance chain — is tracked in Kamu's provenance database. Kamu also records activity events: file access logs, metadata changes, and the complete append-only version graph for every dataset. The Labs API exposes this data through a GraphQL interface, enabling programmatic read and write access to data rooms.

**Sanity CMS** provides editorial and marketing content — project descriptions, team profiles, research category taxonomies, curated homepage content, and blog posts. This content is managed by the Molecule team and bioDAO partners through Sanity Studio, a customizable content editing interface. Sanity is a third-party headless CMS platform (sanity.io) that stores content on its hosted infrastructure and exposes it through its own API.

**GeckoTerminal** provides supplementary token price and market data. The MCP tools that power MIRA's market intelligence pull pricing data from GeckoTerminal in addition to the DeSci API's own indexed market data, providing OHLCV (open, high, low, close, volume) historical price charts and liquidity metrics.

### The Indexing Pipeline

On-chain events do not flow directly from the blockchain to the API. There is an indexing layer in between that parses blockchain events and writes them into the Aurora database in a structured, queryable format.

When a transaction occurs on-chain — an IP-NFT is minted, IPTs are transferred, a crowdsale receives a contribution, treasury funds are deployed, or any Lab transaction is executed — the blockchain records that event as a transaction receipt with associated logs. Service providers monitor the blockchain, detect relevant contract events from Molecule Protocol contracts, parse the event data (decoding function signatures, extracting parameter values, resolving token metadata), and write the resulting structured records into the Aurora database.

This indexing step is what makes the platform performant. Without it, every page load, every search query, and every MIRA conversation would require reading and parsing raw blockchain state — which is slow, expensive, and impractical for a real-time user experience. Instead, the indexer runs continuously in the background, keeping Aurora in sync with the on-chain state, and the DeSci API queries Aurora directly.

The indexing pipeline handles several data transformations during this process. Raw contract events are decoded into human-readable records. Token metadata (names, symbols, images) is resolved from on-chain tokenURI pointers. Pricing data is computed from DEX pool events (swaps, liquidity changes). Ownership graphs are maintained as tokens transfer between wallets. Treasury balances are aggregated across multiple token types. Crowdsale states are tracked through their lifecycle (created, running, settled, failed).

The result is a relational database where complex queries — "show me all Labs sorted by treasury size" or "find all IPTs with market cap above $100k" — can be resolved in milliseconds rather than requiring full blockchain scans.

**Note:** The Molecule Subgraph (powered by The Graph) is a separate, parallel indexing path. The Subgraph also indexes on-chain events into a queryable GraphQL API, but it serves a different purpose: it provides direct, unauthenticated access to raw on-chain event data for developers who want to query blockchain state without going through the DeSci API's aggregation layer. The Subgraph and the Aurora-backed DeSci API index the same on-chain events but serve different consumer needs. The Subgraph is documented separately in the References section.

### The API Layer

The DeSci API is a GraphQL API that sits on top of the Aurora database and serves aggregated data to all consumers. It is the primary interface through which the platform operates.

The API serves two broad categories of data. Market and token data — representing approximately 85% of current API traffic — includes IPT prices, market caps, liquidity depths, holder distributions, trading volumes, IP-NFT metadata, crowdsale states, treasury balances, project listings, and transaction histories. This data originates from on-chain events, indexed through the pipeline described above into Aurora.

Research and Lab data — representing the remaining 15% — includes data room file listings, file versions, announcements, project activity feeds, and semantic search results. This data is served through the Labs API, which reads from Kamu for file metadata and provenance, and generates presigned S3 URLs through Filebase for file uploads and downloads.

The API uses a two-tier authentication model. Read operations (queries) require an API key, issued by the Molecule team upon request. Write operations (mutations) — file uploads, metadata updates, announcements — require both an API key and a service token, which is scoped to a specific wallet address and Lab. Service tokens have configurable expiration and can be extended or revoked through the API. For detailed authentication setup, credential management, and rate limits, see the Labs API reference page.

### How Consumers Access Data

Different consumers interact with the data layer through different interfaces, depending on their use case.

**The Molecule frontend (DeSci Screener)** consumes the DeSci API's GraphQL endpoints for project listings, token market data, Lab metadata, and activity feeds. It calls the Labs API for data room operations — file uploads, downloads, and encrypted file retrieval. It currently also calls Sanity's API directly for editorial content (project descriptions, blog posts, categories). It performs client-side decryption when users access encrypted files: for new files, it calls `decryptDataKey` to retrieve the unwrapped DEK after the backend re-verifies access conditions against live chain state, and decrypts locally via Web Crypto; for legacy files encrypted before the migration, it falls back to the Lit Protocol SDK.

**MIRA** accesses data through MCP (Model Context Protocol) tools — structured function calls that the AI model invokes during conversations. The current MCP tool suite provides five tools: get-ipts (token listings with market data), get-ipt-categories (research categories with statistics), get-project-summary (comprehensive project overview), get-ipt-historic-prices (OHLCV price history via GeckoTerminal), and get-project-activity (recent file and announcement events). MIRA's knowledge base — a curated corpus about DeSci, Molecule's architecture, and the broader ecosystem — is maintained separately and updated through an automated crawl pipeline.

**External developers and AI agents** access Lab data through two primary interfaces. The Labs API provides full GraphQL access to data room operations: listing projects, querying files, uploading and versioning research data, creating announcements, and performing semantic search across Labs. The Client SDK (@moleculexyz/client-sdk) wraps the Labs API in a type-safe TypeScript interface with convenience methods for common workflows like file upload, encrypted file handling, and market data queries. Both interfaces are documented in the References section.

**The Subgraph** provides direct access to indexed on-chain events — IP-NFT mints, token transfers, crowdsale contributions, and marketplace activity — for consumers that need raw blockchain data without the API's aggregation layer. It is powered by The Graph, requires no authentication, and is documented separately.

### The Integration Pipeline

When a researcher uploads a file to a Lab, the data flows through the full pipeline and becomes available to every consumer in the ecosystem. The file is AES-256-GCM encrypted client-side with a per-file wrapped data encryption key (Onchain-Verified Envelope Encryption), uploaded through the Labs API to Filebase, committed to IPFS and Arweave by Kamu (which records the version, content hash, author DID, and provenance metadata), and referenced on-chain in the Lab's Token Bound Account. The on-chain reference event is then picked up by the indexer and written to Aurora. At that point, the file is retrievable through the Labs API by any authenticated consumer with the appropriate access level, searchable via the semantic search endpoint, visible in MIRA's project activity responses, and reflected in the Lab's data room size and activity metrics on the frontend.

Market data flows through a different path. When a token event occurs on-chain — an IPT trade on a DEX, a crowdsale contribution, a treasury deployment — the indexer picks up the event, computes the relevant metrics (new price, updated volume, changed holder count), and writes the results to Aurora. The DeSci API serves the updated data on the next query. MIRA's MCP tools access this data in real time during conversations, supplemented by GeckoTerminal for historical OHLCV charts.

Announcements follow the Labs API path. A Lab owner creates an announcement through the Labs API (or the platform UI), optionally attaching data room files. The announcement is stored via Kamu with its timestamp, author, and content, and immediately appears in the project's activity feed, the global activity feed, and MIRA's context when users ask about the project.

### Semantic Search

The API exposes a semantic search endpoint that queries across all Labs, files, and announcements in the ecosystem. Queries are processed as natural language — searching for "gene therapy for rare diseases" returns Labs whose data room contents, announcements, and metadata are semantically relevant, not just keyword matches.

Search results can be filtered by IP-NFT UIDs, tags, categories, access levels, and content kinds (files or announcements). Each result includes the matching entity, its parent Lab, and a relevance score. This powers both the platform's search interface and MIRA's ability to discover related projects during conversations.

### Access Levels and Gating

The API enforces access levels at the file level, consistent with the access control model described in the Data Privacy & Access page. Public files are accessible to any API consumer without authentication. Admin-restricted files require a valid API key and service token tied to an authorised wallet. Token-holder-gated files are accepted by the API but require on-chain verification for decryption — the API serves the encrypted blob and encryption metadata, and only wallets that satisfy the stored access conditions (re-verified against live chain state) can obtain the unwrapped data encryption key (through `decryptDataKey` for current-flow files, or through the Lit SDK for legacy files).

This means the API can serve file metadata (path, version, content type, access level) for any file regardless of access level, but the actual file content for encrypted files is only accessible to authorised parties who satisfy the stored access conditions and then decrypt client-side.

### Current Limitations

The current API and MCP tool suite is weighted toward market and token data. Several research-oriented capabilities are not yet exposed through the API or MCP tools. These include structured scientific data queries (querying dataset contents by schema or field values), dataset version diffs (comparing what changed between two versions of a file), cross-Lab provenance queries (tracing how a dataset or methodology was shared or derived across multiple Labs), and research file metadata queries for MIRA (the MCP tools currently cannot access data room contents, search across file metadata, or retrieve dataset version histories on behalf of the AI).

These gaps mean that MIRA can tell you a project's token price, market cap, and recent announcements, but cannot yet directly inspect the contents of a Lab's data room or compare the scientific substance of two projects' research outputs. Developers building integrations should be aware that the Labs API provides richer data room access than the MCP tools currently expose.

The Sanity CMS integration is also in transition. The frontend currently calls Sanity's API directly for editorial content, bypassing the DeSci API. The planned architecture consolidates Sanity content into the DeSci API so the frontend has a single data source. This migration is actively in progress.

### Data Architecture Summary

<table><thead><tr><th width="156.0859375">Layer</th><th>Component</th><th>Role</th></tr></thead><tbody><tr><td><strong>Data Origins</strong></td><td>Blockchain (Eth/Base)</td><td>On-chain events: mints, transfers, crowdsales, treasury</td></tr><tr><td></td><td>Researcher uploads</td><td>Research files via Labs API / platform UI</td></tr><tr><td></td><td>Editorial team</td><td>Project content via Sanity Studio</td></tr><tr><td><strong>Indexing</strong></td><td>Indexer / Service Provider</td><td>Parses blockchain events, writes to Aurora</td></tr><tr><td></td><td>The Graph (Subgraph)</td><td>Parallel on-chain index, separate consumer path</td></tr><tr><td><strong>Storage</strong></td><td>Amazon Aurora (AWS)</td><td>Relational DB for indexed on-chain + application data</td></tr><tr><td></td><td>Kamu v2</td><td>Provenance DB for file versions, DIDs, activity events</td></tr><tr><td></td><td>IPFS + Arweave</td><td>Encrypted research file blobs (ciphertext)</td></tr><tr><td></td><td>Sanity CMS</td><td>Editorial content, categories, blog posts</td></tr><tr><td></td><td>Onchain-Verified Envelope Encryption + AccessResolver</td><td>Wrapped per-file DEKs with conditions stored on Kamu (ODF) and re-verified against live on-chain state (legacy files still resolve via Lit Protocol)</td></tr><tr><td><strong>API</strong></td><td>DeSci API (GraphQL)</td><td>Aggregates Aurora data → serves market/token/project data</td></tr><tr><td></td><td>Labs API (GraphQL)</td><td>Aggregates Kamu data → serves data room operations</td></tr><tr><td></td><td>MCP Server</td><td>Exposes 5 tools for AI assistants (market data focused)</td></tr><tr><td><strong>Consumers</strong></td><td>DeSci Screener (frontend)</td><td>Platform UI — queries DeSci API + Labs API + Sanity</td></tr><tr><td></td><td>MIRA</td><td>AI assistant — queries via MCP tools</td></tr><tr><td></td><td>External devs / agents</td><td>Programmatic access via Labs API + Client SDK</td></tr><tr><td></td><td>Subgraph consumers</td><td>Direct on-chain event queries via The Graph</td></tr></tbody></table>

### Related Pages

For the storage pipeline (how research files are stored and versioned): see **Data Storage**.

For encryption and access control (how data is protected): see **Data Privacy & Access**.

For the on-chain event index (direct blockchain queries): see **Subgraph**.

For the GraphQL API reference (data room endpoints, queries, mutations): see **Labs API**.

For the TypeScript SDK (client library for integrations): see **SDKs**.

For MIRA's tool capabilities (what the AI can query): see **MCP Tools**.
