---
description: Molecule Insights & Research Assistant
icon: brain
---

# MIRA

### Overview <a href="#overview" id="overview"></a>

[MIRA](https://mira.molecule.xyz/) serves as the connective layer between complex scientific research and the growing community participating in decentralized science. It combines a curated knowledge base, real-time market data via Model Context Protocol (MCP), and AI-powered project scoring to serve researchers, traders, and institutions. The name—Molecule Insights & Research Assistant—reflects its dual purpose: delivering deep insights while guiding users through the complexities of IP-NFTs, IPTs, and decentralized funding.

MIRA serves three core user groups:

* **Researchers** — Understanding DeSci, structuring IP, navigating the minting process
* **Traders/Investors** — Accessing real-time token data, project insights, and market intelligence
* **DAOs & Institutions** — Evaluating project maturity through standardized scoring frameworks

#### Key Capabilities

* **Scoped Expertise:** Answers queries specifically about Molecule's products, IP-NFTs, IPTs,\
  decentralized science (DeSci), bioDAOs, and funding mechanisms using curated local sources and real-time API data.
* **Real-Time Market Intelligence:** Retrieves live token prices, holder distributions, liquidity\
  metrics, project activity, and historical price data through MCP tools connected to Molecule's\
  APIs.
* **AI-Powered Project Scoring:** Analyzes Data Room files to generate Technology Readiness Level (TRL) assessments and weighted evaluation scores covering therapeutic relevance, optionality, and IP strength.
* **Context Awareness:** Detects the user's current page and viewing context, tailoring responses to the specific token or project being explored.
* **Safe and Transparent:** Employs responsible AI design—it knows when to say "I don't know" and defers to human experts to avoid misinformation. Never provides investment advice.
* **Source Attribution:** Every response links back to its origin, ensuring traceability and\
  accountability. Performance and feedback are tracked via Langfuse to continually improve response quality.
* **Web Search Fallback:** If local knowledge is insufficient but relevant, MIRA augments its answers by retrieving current information from trusted online sources.

### Architecture & Workflow <a href="#architecture-and-workflow" id="architecture-and-workflow"></a>

1. User Query
2. Extract Context (current token/page being viewed)
3. Search Local Knowledge (LanceDB)
4. If sufficient → answer directly
5. If real-time data needed → execute MCP tools
6. If insufficient but relevant → perform web search
7. If off-topic → politely decline
8. Generate Response with linked sources
9. User Feedback & Tracking via Langfuse

**Tech Stack:**

<table><thead><tr><th width="225.05859375">Component</th><th>Purpose</th></tr></thead><tbody><tr><td>OpenAI GPT-4o</td><td>Powers reasoning, tool selection, and response generation</td></tr><tr><td>Model Context Protocol</td><td>Connects to Molecule APIs for real-time token and project data</td></tr><tr><td>LanceDB</td><td>Vector-based semantic search for curated documentation</td></tr><tr><td>Vercel AI SDK</td><td>Streaming responses and tool orchestration</td></tr><tr><td>Upstash Redis</td><td>Caching layer for API responses</td></tr><tr><td>Sanity CMS</td><td>Stores and manages AI-generated project scores</td></tr><tr><td>Langfuse</td><td>Monitors performance metrics and feedback</td></tr><tr><td>Tavily API</td><td>Web search for current information beyond local knowledge</td></tr></tbody></table>

<table><thead><tr><th width="289.67578125">MCP Tool</th><th>Function</th></tr></thead><tbody><tr><td>molecule-get-ipts</td><td>Token listings with market data, holders, and liquidity</td></tr><tr><td>molecule-get-project-activity</td><td>Announcements, file updates, and data room changes</td></tr><tr><td>molecule-get-ipt-categories</td><td>Research categories and associated assets</td></tr><tr><td>molecule-get-project-summary</td><td>Comprehensive overviews with price history</td></tr><tr><td>molecule-get-ipt-historic-prices</td><td>Daily OHLCV data up to 90 days</td></tr></tbody></table>

**Evolution & Strategy**

* **v0 — Knowledge Foundation:** Launched as Molecule's first AI-driven tool, MIRA v0 established a\
  context-aware assistant built on curated knowledge from official documentation, protocol blogs, and DeSci standards. It responds only from verified sources, preventing hallucination and ensuring reliability.
* **v1 — Real-Time Integration:** Introduced MCP tool integration for live market data and was embedded directly into the Molecule Screener. Added context awareness to understand which token users are viewing, enabling natural conversational flows with quick actions for common queries.
* **v2 — Project Scoring (Current):** Expanded to analyze Data Room contents and generate standardized project assessments. Scores flow through a review pipeline—from AI generation through human validation to publication via Sanity CMS—ensuring appropriate oversight.
* **v3 — Knowledge Graphs (Planned):** Future iterations will introduce relationship mapping across the ecosystem, connecting projects, researchers, and institutions through interactive visualizations. This phase also addresses confidential data handling and explores token-gated access models.

Check our [MIRA Github Repo](https://github.com/moleculeprotocol/mira-ai-prototype-v0) for more info.

