---
description: >-
  Build on Molecule: Integrate Labs, extend the protocol, and deploy autonomous
  research agents
icon: robot
---

# Developers/AI Agents

### Who This Guide Is For

You're a developer building on the Molecule ecosystem. You might be integrating Lab data into a front-end, writing a smart contract module that adds new capabilities to Labs, deploying an AI agent that operates on research data, or building a tool that queries ecosystem state for analytics or trading. This guide maps out the integration surfaces, explains what's available today versus what's on the roadmap, and shows you the fastest path to a working integration for each use case.

The reference pages (Contracts, SDKs, Labs API, MCP Tools, Subgraph) contain the full API specifications, type definitions, and code examples. This guide is the narrative layer that explains when to use which tool, how the pieces connect, and what the architecture expects from you.

### The Integration Surface

Molecule exposes four primary integration layers, each serving different developer needs.

The Labs API is a GraphQL endpoint for reading and writing to Lab data rooms — the off-chain encrypted storage where research files, announcements, and metadata live. This is the primary interface for applications that need to manage scientific data: uploading files, querying project activity, searching across Labs, and managing announcements. Authentication uses API keys for reads and service tokens for writes. The full specification, including every query and mutation, is documented in the Labs API reference.

The Client SDK is a TypeScript package that wraps the Labs API and subgraph queries into a type-safe interface. It handles authentication, file uploads (including encrypted uploads via Lit Protocol), IPT market data, crowdsale queries, and project management. The SDK is read-focused for on-chain data — for on-chain writes (minting, tokenizing, bidding), you use the contract ABIs directly with viem/wagmi. The SDK documentation includes quickstart examples and every available method.

The Smart Contracts are the on-chain layer. The V2 contracts (IPNFT, Tokenizer, CrowdSale, SchmackoSwap) are deployed on Ethereum mainnet and handle IP-NFT minting, IPT creation, token sales, and trading. The V3 contracts (Modular\_OnChainLab, OnChainLabFactory, ERC7484Registry, and associated modules) are deployed on Sepolia and introduce the modular account architecture. Contract addresses, ABIs, and upgrade patterns are documented in the Contracts reference. The Architecture page provides the full implementation-level breakdown of how these contracts compose.

The MCP Server is a Model Context Protocol endpoint that lets AI assistants query Molecule ecosystem data in real time — IPT prices, project activity, categories, and summaries. It's the fastest way to give an LLM context about the DeSci ecosystem without building a custom integration. Setup takes one config file. The MCP Tools reference covers available tools, self-hosting, and programmatic integration via the @moleculexyz/ai package.

### Building a Front-End or Dashboard

If you're building an interface that displays Lab data, IPT markets, or research activity, your primary tools are the Client SDK and the Subgraph.

Start with the SDK for most queries. It provides pre-built methods for fetching IPTs with market data, querying IP-NFTs by owner or ID, retrieving crowdsale details and contribution status, listing projects, and searching across Labs. For on-chain state that isn't indexed by the API — for example, real-time token balances, allowances, or contract state — query the subgraph directly or call the contracts via viem.

For data room interactions (showing a Lab's files, uploading research data on behalf of a user), use the SDK's labs module. File uploads follow a three-step flow: initiate the upload to get a presigned S3 URL, PUT the file to that URL, then finalize the upload with metadata. The SDK abstracts this for you. If the file needs encryption, the @moleculexyz/storage package provides Lit Protocol integration — encrypt the file client-side, then upload the ciphertext with encryption metadata attached. The Labs API reference walks through both flows with full code examples.

Authentication splits into two paths. Unauthenticated calls work for all read operations against public data — IPT listings, market data, project summaries. Authenticated calls require a Privy JWT token and wallet address, and are needed for any write operation or access to private data rooms.

### Extending Labs with Smart Contract Modules

The module system is Molecule's primary extensibility mechanism. If you want to add new capabilities to Labs — automated royalty distribution, governance voting, milestone-based fund release, AI agent execution boundaries, licensing logic, oracle integrations — you build a module.

The module architecture is documented in detail in the Module Registry section, including the security model, attestation flow, and installation process. Here, we focus on what you need to know to actually build one.

There are two module types. Executor modules initiate transactions from a Lab account — they call executeFromExecutor on the Lab to perform actions on its behalf. This is the right pattern when external logic needs to trigger Lab actions: distributing funds, releasing escrowed payments, executing agent workflows, or responding to oracle events. Fallback modules extend a Lab's interface by registering new function selectors, allowing the Lab to respond to calls it doesn't natively support. This is the right pattern when you want to add queryable state or new interaction surfaces to a Lab — governance interfaces, custom data accessors, or protocol-specific compatibility layers.

Your module is a standalone Solidity contract. It does not inherit from the Lab contract. It interacts with Labs through the defined execution interfaces. The development cycle looks like this: write your module contract, test it against the Lab's execution interface using Foundry (the poc-protocol-modular-onchain-labs repo provides test fixtures), deploy it to the target chain, and submit it for attestation through the Molecule team. Once attested in the ERC-7484 Registry, any Lab owner can install your module via a UserOperation through the ERC-4337 EntryPoint.

A critical detail: executor modules that use delegatecall operate within the Lab's storage context. This means they can read and write the Lab's state, which is powerful but demands careful development. Storage collisions, reentrancy, and unauthorized asset access are all risks that the attestation process is designed to catch. If your module only needs to trigger Lab actions without accessing its storage directly, prefer the call-based execution pattern over delegatecall.

The Sepolia deployment includes all the infrastructure you need for testing: a Factory at 0x3F4EBF8e326c807B7D075EaB5875E54B9E3206Bd, a ModuleRegistry at 0x005ad978548657Bd208aFC238b24a17311365b5e, and the full set of supporting contracts listed in the Architecture page.

### Deploying an AI Research Agent

AI agents that operate on Lab data — reading files, running analyses, writing findings back — interact through the Labs API. The protocol treats agent outputs the same as any other data: versioned records with content identifiers, permanent onchain references, and configurable access control.

The simplest agent integration is read-only: query a Lab's data room for files, download them, perform analysis, and present results. This requires only an API key. Querying projectWithDataRoomAndFilesV2 gives you the complete file list with download URLs, content types, and encryption metadata. For encrypted files, you'll need to satisfy the Lit Protocol access conditions to decrypt — which may require holding specific tokens or having a specific wallet address.

A write-enabled agent goes further: it reads data, performs analysis, and writes results back as new files in the Lab's data room. This requires both an API key and a service token, obtained by contacting the Molecule team with your wallet address, use case, and target Lab. The three-step upload flow (initiate, PUT, finalize) lets you write any file type with metadata including descriptions, tags, categories, and searchable content text. Every file the agent writes becomes a permanent, versioned record in the Lab's history.

For agents that need to operate autonomously within an Onchain Lab's smart contract context — executing treasury operations, managing permissions, or interacting with DeFi protocols — the path is through executor modules. The agent's logic is deployed as a module contract, attested in the ERC-7484 Registry, and installed on the target Lab by its owner. At that point, the agent can call executeFromExecutor to perform Lab actions within whatever boundaries the module enforces (spending limits, allowed function selectors, time windows). The V3 whitepaper describes two operational modes for this: Human-Directed (agents assist, humans approve) and Fully Autonomous (agents operate independently within onchain constraints).

### Building with BioAgents

BioAgents is an open-source AI scientist framework for biological research, and it's the reference implementation for autonomous research agents on Molecule. If you're building a research agent, starting from BioAgents gives you a proven architecture and a set of specialized agents you can extend or replace.

The framework provides a modular agent architecture where each agent is an independent function that performs a specific research task. The Planning Agent creates research plans from user questions and available data. The Literature Agent searches scientific literature with inline citations via multiple backends (a built-in semantic search with LLM reranking, OpenScholar, or Edison). The Analysis Agent runs data analysis on uploaded datasets. The Hypothesis Agent synthesizes findings into testable hypotheses. The Reflection Agent reviews overall research progress and adjusts methodology. The Reply Agent generates user-facing responses with preserved citations.

BioAgents operates through two routes: /api/chat for conversational research questions with automatic literature search, and /api/deep-research for iterative hypothesis-driven investigation with multi-step planning and reflection cycles.

To integrate BioAgents with a Molecule Lab, you connect the agent to the Labs API. The agent reads datasets from the Lab's data room, processes them through its analysis pipeline, and writes findings back as new versioned files. The Labs API's semantic search (searchLabs) lets agents discover relevant data across the entire ecosystem, not just a single Lab. The three-step file upload flow lets agents write results back with full metadata — descriptions, tags, categories — making agent outputs discoverable and attributable.

The framework supports two authentication systems. JWT authentication for production deployments where your backend authenticates users and issues signed tokens. And x402 micropayments for pay-per-request access using USDC on Base, which is relevant for agents that serve external users or charge for analysis. The codebase also includes a job queue system (BullMQ + Redis) for production deployments that need reliable background processing, horizontal scaling, and automatic retries.

BioAgents also includes a customizable knowledge base backed by a vector database with semantic search and Cohere reranking. You can load domain-specific documents (PDFs, Markdown, DOCX) that the Literature Agent will search alongside public scientific literature — useful for giving your agent proprietary context about a specific research domain.

The full setup guide, architecture diagrams, and agent implementation details are in the BioAgents repository at github.com/bio-xyz/BioAgents.

### Giving AI Assistants Molecule Context via MCP

If you want existing AI assistants (Claude, GPT, or any MCP-compatible client) to have real-time access to Molecule ecosystem data, the MCP server is the lowest-friction path. No custom code required — just a configuration entry pointing at the public endpoint.

The MCP server exposes five tools: querying available IPTs with market data, fetching project activity for a specific token, listing IPT categories, getting comprehensive project summaries, and retrieving historical OHLCV price data. These cover the most common questions an AI assistant needs to answer about the ecosystem.

For programmatic integration — embedding Molecule tools into your own AI application — the @moleculexyz/ai package provides a tool registry that works directly with the Vercel AI SDK. You instantiate a registry pointed at the MCP endpoint, pass the tools to your LLM call, and the model can query Molecule data as part of its reasoning process. The MCP Tools reference includes the full setup, self-hosting instructions for private deployments, and caching configuration.

### Querying On-Chain State via Subgraph

For indexed on-chain data — IP-NFT minting events, IPT creation, crowdsale participation, token transfers — the subgraph provides a GraphQL interface over historical blockchain data. This is the right tool for analytics dashboards, portfolio trackers, historical queries, and any application that needs to reconstruct the full timeline of protocol events.

The subgraph indexes Ethereum mainnet and is documented in the Subgraph reference. For queries that combine on-chain and off-chain data (for example, an IPT's market data alongside its Lab's data room contents), use the Client SDK, which unifies subgraph queries and API calls into a single interface.

### Where to Start

Your starting point depends on what you're building.

If you're building a front-end or dashboard, start with the Client SDK. Install @moleculexyz/client-sdk, follow the quickstart in the SDK reference, and you'll have IPT market data rendering in minutes.

If you're building a smart contract module, start with the poc-protocol-modular-onchain-labs repository. Clone it, run the Foundry tests to understand the execution model, then write your module against the test fixtures. Deploy to Sepolia for testing and contact the Molecule team for attestation when you're ready for production.

If you're building an AI research agent, start with BioAgents. Fork the repository, configure your LLM providers, and connect it to a Lab's data room via the Labs API. The /api/deep-research route gives you a working multi-agent research pipeline out of the box.

If you just want to give an AI assistant Molecule context, add the MCP server URL to your client's config and you're done in sixty seconds.

For API keys, service tokens, attestation requests, or any integration support, reach out on the Molecule Discord.
