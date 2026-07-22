---
description: >-
  Build on Molecule: Integrate Labs, extend the protocol, and deploy autonomous
  research agents
icon: robot
---

# Developers/AI Agents

### Who This Guide Is For

You're a developer building on the Molecule ecosystem. You might be integrating Lab data into a front-end, writing a smart contract module that adds new capabilities to Labs, deploying an AI agent that operates on research data, or building a tool that queries ecosystem state for analytics or trading. This guide maps out the integration surfaces, explains what's available today versus what's on the roadmap, and shows you the fastest path to a working integration for each use case.

The reference pages (Contracts, Labs API, MCP Tools, Subgraph) contain the full API specifications, type definitions, and code examples. This guide is the narrative layer that explains when to use which tool, how the pieces connect, and what the architecture expects from you.

### The Integration Surface

Molecule exposes four primary integration layers, each serving different developer needs.

The Labs API is a GraphQL endpoint for reading and writing to Lab data rooms — the offchain encrypted storage where research files, announcements, and metadata live. This is the primary interface for applications that need to manage scientific data: uploading files, querying project activity, searching across Labs, and managing announcements. Authentication uses API keys for reads and service tokens for writes. The full specification, including every query and mutation, is documented in the Labs API reference.

The Subgraph is a GraphQL interface over indexed onchain events — IP-NFT activity, IPT balances, crowdsale participation, token transfers. It's the right tool for reading historical and current onchain state without running your own indexer: market data, ownership, crowdsale contributions, and event timelines. It requires no authentication and indexes Ethereum mainnet. For onchain writes, you call the contract ABIs directly with viem/wagmi. The Subgraph reference documents the full schema and example queries.

The Smart Contracts are the onchain layer. The V2 contracts (IPNFT, CrowdSale, SchmackoSwap) on Ethereum mainnet underpin the existing IP-NFT assets, token sales, and trading. The V3 contracts (OnChainLab, OnChainLabFactory, ERC7484Registry, OclTokenizer, and associated modules) are deployed on Base mainnet and Base Sepolia and introduce the modular account architecture plus Lab tokenization. Contract addresses, ABIs, and upgrade patterns are documented in the Contracts reference. The Architecture page provides the full implementation-level breakdown of how these contracts compose.

The MCP Server is a Model Context Protocol endpoint that lets AI assistants query Molecule ecosystem data in real time — IPT prices, project activity, categories, and summaries. It's the fastest way to give an LLM context about the DeSci ecosystem without building a custom integration. Setup takes one config file. The MCP Tools reference covers available tools, self-hosting, and programmatic integration against the MCP endpoint.

### Building a Front-End or Dashboard

If you're building an interface that displays Lab data, IPT markets, or research activity, your primary tools are the DeSci API, the Labs API, and the Subgraph.

Use the DeSci API for market and token data — IPTs with prices, project summaries, and activity feeds — and the Subgraph for indexed onchain state such as IP-NFT ownership, crowdsale contributions, and event timelines. For real-time onchain state that isn't indexed — for example, live token balances, allowances, or contract state — call the contracts directly via viem.

For data room interactions (showing a Lab's files, uploading research data on behalf of a user), use the Labs API. File uploads follow a three-step flow: call `initiateCreateOrUpdateFile` to get a presigned S3 URL, PUT the file to that URL, then call `finishCreateOrUpdateFile` with metadata. If the file needs encryption, first request a key via `generateDataEncryptionKey` — the backend returns a one-shot plaintext DEK and its encrypted form — and AES-256-GCM encrypt the file locally via Web Crypto before uploading, attaching the encryption metadata on finish. On download, `decryptDataKey` returns the unwrapped DEK after the backend re-verifies the file's access conditions against live onchain state. Legacy files encrypted before the migration to Onchain-Verified Envelope Encryption continue to decrypt via the Lit Protocol path. The [Data Privacy & Access](../core-infrastructure/data/data-privacy-and-access.md) page walks through both flows end-to-end.

Authentication splits into two paths. Unauthenticated calls work for all read operations against public data — IPT listings, market data, project summaries. Authenticated calls require a Privy JWT token and wallet address, and are needed for any write operation or access to private data rooms.

### Extending Labs with Smart Contract Modules

The module system is Molecule's primary extensibility mechanism. If you want to add new capabilities to Labs — automated royalty distribution, governance voting, milestone-based fund release, AI agent execution boundaries, licensing logic, oracle integrations — you build a module.

The module architecture is documented in detail in the Module Registry section, including the security model, attestation flow, and installation process. Here, we focus on what you need to know to actually build one.

There are two module types. Executor modules initiate transactions from a Lab account — they call executeFromExecutor on the Lab to perform actions on its behalf. This is the right pattern when external logic needs to trigger Lab actions: distributing funds, releasing escrowed payments, executing agent workflows, or responding to oracle events. Fallback modules extend a Lab's interface by registering new function selectors, allowing the Lab to respond to calls it doesn't natively support. This is the right pattern when you want to add queryable state or new interaction surfaces to a Lab — governance interfaces, custom data accessors, or protocol-specific compatibility layers.

Your module is a standalone Solidity contract. It does not inherit from the Lab contract. It interacts with Labs through the defined execution interfaces. The development cycle looks like this: write your module contract, test it against the Lab's execution interface using Foundry (the poc-protocol-modular-onchain-labs repo provides test fixtures), deploy it to the target chain, and submit it for attestation through the Molecule team. Once attested in the ERC-7484 Registry, any Lab owner can install your module via a UserOperation through the ERC-4337 EntryPoint.

A critical detail: modules never run via delegatecall — the account dispatches module calls as regular external calls, so a module cannot touch the Lab's storage directly. Your executor triggers Lab actions through `executeFromExecutor`, and the attestation process reviews what those actions can do. Reentrancy and unauthorized asset access remain the key risks the review looks for.

The Base Sepolia deployment includes all the infrastructure you need for testing: the OnChainLabFactory at `0xd629FE2310b4309a212495F10A47f8436dcEfD90`, the ERC7484Registry at `0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5`, and the full set of supporting contracts listed in the [Contracts reference](../references/contracts/).

### Deploying an AI Research Agent

AI agents that operate on Lab data — reading files, running analyses, writing findings back — interact through the Labs API. The protocol treats agent outputs the same as any other data: versioned records with content identifiers, permanent onchain references, and configurable access control.

The simplest agent integration is read-only: query a Lab's data room for files, download them, perform analysis, and present results. This requires only an API key. Querying `labWithDataRoomAndFiles` gives you the complete file list with download URLs, content types, and encryption metadata. For encrypted files, the agent calls `decryptDataKey` with its service token; the backend evaluates the file's onchain access conditions and, if satisfied, returns the plaintext DEK. Access commonly resolves through a [Viewer or Contributor role grant](../core-infrastructure/roles-and-permissions.md) on the Lab — granted by the Lab owner with `isAgent = true` and a bounded `expiry` matching the agent's session-key lifetime.

A write-enabled agent goes further: it reads data, performs analysis, and writes results back as new files in the Lab's data room. This requires both an API key and a service token. Two paths to a service token:

* **Long-lived service token** — mint one via the `generateServiceToken` mutation (wallet signature or Privy session), or contact the Molecule team. The token is a JWT tied to your wallet; write authorization is resolved from that wallet's onchain role on the target Lab.
* **Pay-per-call via the** [**x402 Gateway**](../api-reference/x402-gateway.md) — for agents that serve external users, charge per request, or don't have pre-provisioned credentials. The gateway settles a USDC payment on Base per call and mints a short-lived (default 5-minute) service token scoped to one mutation. Available for `initiateCreateOrUpdateFile`, `finishCreateOrUpdateFile`, `createAnnouncement`, `createLab`, `generateDataEncryptionKey`, and `decryptDataKey`.

The three-step upload flow (initiate, PUT, finalize) lets you write any file type with metadata including descriptions, tags, categories, and searchable content text. If the file should be confidential, first request a key via `generateDataEncryptionKey` — the backend returns a one-shot plaintext DEK (plus its encrypted form) you use to encrypt locally before upload, attaching the encryption metadata on finish. Every file the agent writes becomes a permanent, versioned record in the Lab's history.

For agents that need to operate autonomously within an Onchain Lab's smart contract context — executing treasury operations, managing permissions, or interacting with DeFi protocols — the path is through executor modules. The agent's logic is deployed as a module contract, attested in the ERC-7484 Registry, and installed on the target Lab by its owner. At that point, the agent can call executeFromExecutor to perform Lab actions within whatever boundaries the module enforces (spending limits, allowed function selectors, time windows). The V3 whitepaper describes two operational modes for this: Human-Directed (agents assist, humans approve) and Fully Autonomous (agents operate independently within onchain constraints).

### Building with BioAgents

BioAgents is an open-source AI scientist framework for biological research, and it's the reference implementation for autonomous research agents on Molecule. If you're building a research agent, starting from BioAgents gives you a proven architecture and a set of specialized agents you can extend or replace.

The framework provides a modular agent architecture where each agent is an independent function that performs a specific research task. The Planning Agent creates research plans from user questions and available data. The Literature Agent searches scientific literature with inline citations via multiple backends (a built-in semantic search with LLM reranking, OpenScholar, or Edison). The Analysis Agent runs data analysis on uploaded datasets. The Hypothesis Agent synthesizes findings into testable hypotheses. The Reflection Agent reviews overall research progress and adjusts methodology. The Reply Agent generates user-facing responses with preserved citations.

BioAgents operates through two routes: /api/chat for conversational research questions with automatic literature search, and /api/deep-research for iterative hypothesis-driven investigation with multi-step planning and reflection cycles.

To integrate BioAgents with a Molecule Lab, you connect the agent to the Labs API. The agent reads datasets from the Lab's data room, processes them through its analysis pipeline, and writes findings back as new versioned files. The Labs API's semantic search (searchLabs) lets agents discover relevant data across the entire ecosystem, not just a single Lab. The three-step file upload flow lets agents write results back with full metadata — descriptions, tags, categories — making agent outputs discoverable and attributable.

The framework supports two authentication systems. JWT authentication for production deployments where your backend authenticates users and issues signed tokens. And x402 micropayments for pay-per-request access using USDC on Base, which is relevant for agents that serve external users or charge for analysis — see the [x402 Gateway](../api-reference/x402-gateway.md) reference for the full protocol and endpoint list. The codebase also includes a job queue system (BullMQ + Redis) for production deployments that need reliable background processing, horizontal scaling, and automatic retries.

BioAgents also includes a customizable knowledge base backed by a vector database with semantic search and Cohere reranking. You can load domain-specific documents (PDFs, Markdown, DOCX) that the Literature Agent will search alongside public scientific literature — useful for giving your agent proprietary context about a specific research domain.

The full setup guide, architecture diagrams, and agent implementation details are in the BioAgents repository at github.com/bio-xyz/BioAgents.

### Giving AI Assistants Molecule Context via MCP

If you want existing AI assistants (Claude, GPT, or any MCP-compatible client) to have real-time access to Molecule ecosystem data, the MCP server is the lowest-friction path. No custom code required — just a configuration entry pointing at the public endpoint.

The MCP server exposes five tools: querying available IPTs with market data, fetching project activity for a specific token, listing IPT categories, getting comprehensive project summaries, and retrieving historical OHLCV price data. These cover the most common questions an AI assistant needs to answer about the ecosystem.

For programmatic integration — embedding Molecule tools into your own AI application — connect an MCP client to the endpoint and pass its tools to your LLM call, so the model can query Molecule data as part of its reasoning process. Any MCP-compatible client works, including the Vercel AI SDK's MCP client. The MCP Tools reference includes the full setup, self-hosting instructions for private deployments, and caching configuration.

### Querying Onchain State via Subgraph

For indexed onchain data — IP-NFT activity, IPT balances, crowdsale participation, token transfers — the subgraph provides a GraphQL interface over historical blockchain data. This is the right tool for analytics dashboards, portfolio trackers, historical queries, and any application that needs to reconstruct the full timeline of protocol events.

The subgraph indexes Ethereum mainnet and is documented in the Subgraph reference. For queries that combine onchain and offchain data (for example, an IPT's market data alongside its Lab's data room contents), combine subgraph queries with DeSci API and Labs API calls in your application.

### Where to Start

Your starting point depends on what you're building.

If you're building a front-end or dashboard, start with the DeSci API and Subgraph. Follow the query examples in those references and you'll have IPT market data rendering in minutes; add the Labs API when you need data room contents.

If you're building a smart contract module, start with the poc-protocol-modular-onchain-labs repository. Clone it, run the Foundry tests to understand the execution model, then write your module against the test fixtures. Deploy to Sepolia for testing and contact the Molecule team for attestation when you're ready for production.

If you're building an AI research agent, start with BioAgents. Fork the repository, configure your LLM providers, and connect it to a Lab's data room via the Labs API. The /api/deep-research route gives you a working multi-agent research pipeline out of the box.

If you just want to give an AI assistant Molecule context, add the MCP server URL to your client's config and you're done in sixty seconds.

For API keys, service tokens, attestation requests, or any integration support, reach out on the Molecule Discord.
