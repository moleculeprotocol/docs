---
description: Molecule Insights & Research Assistant
---

# ✨ MIRA

## Overview

[MIRA](https://mira.molecule.xyz/) is an AI-powered assistant designed to help users navigate Molecule and the broader DeSci ecosystem. It employs Retrieval‑Augmented Generation (RAG) with a domain‑specific knowledge base, supplemented with real‑time web search when necessary. Initially launched as [**MIRA v0**](https://mira.molecule.xyz/), it serves as a foundation for Molecule’s AI initiatives.

## Key Capabilities

* **Scoped Expertise**: Answers queries specifically about [Molecule’s products](https://docs.molecule.to/documentation/~/revisions/8DP0sNZ6SasJY2Dd9V34/introduction/readme), [IP‑NFTs](https://docs.molecule.to/documentation/~/revisions/8DP0sNZ6SasJY2Dd9V34/ip-nfts/intro-to-ip-nft), decentralized science (DeSci), bioDAOs, and funding mechanisms using curated local sources.
* **Safe and Transparent**: Called a _responsible_ AI design—it knows when to say “I don’t know” and defers to human experts to avoid misinformation.
* **Source Attribution**: Every response links back to its origin, ensuring traceability and accountability. Performance and feedback are tracked via **Langfuse** to continually improve response quality.
* **Fallback to web search**: If local knowledge is insufficient but relevant, MIRA augments its answers by retrieving current information from trusted online sources.

## Architecture & Workflow

### **Workflow:**

1. **User Query**
2. **Search Local Knowledge (LanceDB)**&#x20;
   * If sufficient → answer directly
   * If insufficient but relevant → perform web search
   * If off-topic → politely decline
3. **Generate Response** with linked sources
4. **User Feedback & Tracking** via Langfuse

### **Tech Stack Components:**

* **Chainlit**: Interactive chat UI with streaming responses and feedback buttons.
* **LanceDB**: Vector-based semantic search engine for local documents.
* **OpenAI GPT‑4o**: Powers both local logic (evaluation) and web‑augmented generation.
* **Langfuse**: Monitors performance metrics and feedback to refine [MIRA](https://mira.molecule.xyz/).

### Context & Strategy

* **Genesis**: Announced in July 2025, MIRA v0 marks Molecule’s first foray into AI‑driven tools to democratize access to DeSci knowledge.
* **Vision Forward**: Future iterations aim to integrate more agentic workflows—enhancing decision‑making, pattern recognition, and complex process automation. Plans include bringing MIRA into flagship products like [**Molecule Screener** ](https://molecule.xyz/)and [**Molecule Labs**](https://docs.molecule.to/documentation/~/revisions/8DP0sNZ6SasJY2Dd9V34/molecule-labs/intro-to-molecule-labs) to benefit researchers and funders.\


{% embed url="https://mira.molecule.xyz/" %}





