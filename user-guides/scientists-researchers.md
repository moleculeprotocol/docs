---
description: Your Path from Research to IP Tokens
icon: user-magnifying-glass
---

# Scientists/Researchers

### From Idea to IP Tokens

<figure><img src="../.gitbook/assets/Onchain Lab IP-NFT Flow-2026-01-25-073835.png" alt=""><figcaption></figcaption></figure>

### Who This Guide Is For

You're a researcher — at a university, an independent lab, or somewhere in between — and you have scientific work that deserves funding, recognition, and a path to real-world impact. Maybe you have preliminary data sitting on a hard drive. Maybe you have a patented compound stuck in a tech transfer queue. Maybe you just have a hypothesis and the skills to test it. Molecule gives you a way to take that work onchain, own it, fund it, and run it — without waiting for institutional approval or traditional grant cycles.

This guide walks you through the journey from your perspective. For the technical details behind each step, the Core Concepts section covers the architecture in depth. Here, we focus on what you actually do at each stage, why it matters for your research, and the choices you'll make along the way.

### Starting Point: Where Are You?

Researchers come to Molecule from different places, and the platform is designed to meet you wherever you are.

If you have an idea but no IP yet — perhaps an untested hypothesis, preliminary observations, or a research direction you want to explore — your first step is simply creating a Lab and uploading whatever you have. There are no prerequisites. The Lab is your starting line, not the finish.

If you have existing research through a university or company — datasets, publications, maybe a provisional patent — you can bring that work onchain by creating a Lab, uploading your data, and minting an IP-NFT once you've arranged the legal assignment. The legal frameworks section in DeSci.Codes provides model agreements for exactly this situation.

If you have existing IP or a patent as an independent researcher — you already own the rights. Create a Lab, mint an IP-NFT anchored to your existing IP, and you're immediately ready to tokenize and fundraise.

No matter where you start, the entry point is the same: create a Lab.

### Step 1: Create Your Onchain Lab

Creating a Lab is a single action. You mint a Lab NFT and instantly receive a permanent onchain identity — a wallet address that belongs to your research project. This address is yours. It doesn't belong to a university, a funding body, or a company. If you change institutions, the Lab comes with you. If you sell the project, the entire Lab transfers in one transaction.

The practical experience is straightforward: connect your wallet, click create, and your Lab exists. There's no gas cost — Molecule sponsors the transaction fees for Lab creation. From this moment, your Lab can hold assets, store data, and accumulate a track record.

A few things worth knowing early on. Your Lab address is permanent and deterministic, meaning it works across any EVM chain without needing to be bridged or migrated. Everything you do — uploading a file, receiving funds, minting IP — is recorded as a transaction on this address. Over time, this becomes your project's verifiable CV. Refer to the On-Chain Lab section for the full technical picture of how this works under the hood.

### Step 2: Upload Your Research Data

Once your Lab is live, you can start populating it with your research. Upload datasets, protocols, lab notebooks, images, analysis scripts — anything that constitutes the scientific foundation of your project. Files are encrypted and stored on decentralized infrastructure, with content identifiers recorded onchain so that every version is permanent and traceable.

The key decision here is access control. For each file, you choose who can see it.

Public files are visible to anyone. Use this for data that builds credibility — published datasets, methodology descriptions, summary results. This is how you signal to the community and potential funders that your Lab is active and producing real work.

Token-gated files require holding your project's IP Tokens to access. This is your premium data room. Funders who hold IPTs get access to the detailed research — raw experimental data, unpublished results, proprietary methods. This creates a direct link between financial participation and information access.

Private files remain encrypted and visible only to you (and any roles you delegate). Pre-publication results, sensitive patient data, or trade secrets stay protected until you choose otherwise.

The data you upload is not just storage — it becomes an asset. As your Lab accumulates data, its value grows. Investors evaluate Labs partly by the depth and quality of their data rooms. AI agents (covered below) can operate on your data continuously. And every file upload is a recorded event in your Lab's onchain history, contributing to your project's verifiable track record. For details on the underlying storage and encryption infrastructure, see Data Storage and Data Privacy & Access.

### Step 3: Register and Tokenize Your IP

When your research reaches a point where you want to formalise ownership — a novel compound, a validated methodology, a curated dataset — you mint an IP-NFT into your Lab. The IP-NFT is an onchain asset that represents ownership of your intellectual property, anchored to a legal agreement that defines the rights included.

After minting, you have a choice: keep the IP-NFT whole, or tokenize it.

Keeping it whole means you retain full ownership. You can license it directly to a pharma company, sell it outright via an OTC swap, or hold it while the research matures. This path makes sense when you want maximum control or when you're negotiating a single large deal.

Tokenizing means you create IP Tokens (IPTs) — fungible tokens that represent fractional ownership and governance rights over the underlying IP. This is the path to community funding. IPT holders get economic exposure to your research, access to token-gated data, and a voice in governance decisions. You decide how many tokens to create and how they're distributed.

If you tokenize, you can run a crowdsale. Molecule supports several sale types, each with different trade-offs between simplicity, investor commitment, and alignment incentives. The sale types are shown in the flowchart above. Funds raised go directly into your Lab's treasury — fully transparent, fully under your control. For the detailed mechanics of IP-NFTs, IPTs, and crowdsale options, see the Tokenized IPs section.

### Step 4: Use AI Agents to Accelerate Your Research

This is where Molecule diverges from a traditional funding platform. Your Lab isn't just a container for assets — it's an environment where AI agents can actively work on your research.

Molecule integrates BioAgents, an open-source AI scientist framework built specifically for biological and life sciences research. BioAgents is a multi-agent system where specialised agents handle different parts of the research workflow, and you direct their focus.

**What the agents actually do:**

A Literature Agent searches and synthesises scientific literature relevant to your research question. It returns findings with inline citations, so you can trace every claim back to its source paper. You can point it at a question like "What compounds have shown efficacy against target X in the last three years?" and receive a structured synthesis — not a summary, but a cited analysis you can build on.

An Analysis Agent operates on data you've uploaded to your Lab. Upload a dataset — genomic data, assay results, clinical measurements — and the agent runs analysis, generates visualisations, and identifies patterns. The results are written back to your Lab as new versioned records, each with a content identifier and permanent onchain reference.

A Hypothesis Agent takes findings from literature searches and data analyses and generates testable hypotheses. It considers the current state of your research, what has and hasn't worked, and proposes next steps with supporting evidence.

A Planning Agent coordinates multi-step research workflows. It looks at your research objectives, available data, and agent capabilities, then generates a sequence of literature and analysis tasks to execute.

A Reflection Agent periodically reviews the overall progress of your research — what's been discovered, what methodology is working, where gaps remain — and updates the research direction accordingly.

**How this fits into your Lab:**

The agents read from your Lab's data room through the Labs API, perform their work, and write findings back as new data. Every agent output — a literature synthesis, an analysis result, a generated hypothesis — becomes a permanent, versioned record in your Lab. This means agent work compounds over time. Today's analysis feeds tomorrow's hypothesis, which drives next week's experiment, which produces new data for the agent to analyse.

You can run agents in two modes. In Human-Directed mode, agents assist while you retain strategic control — you review outputs, approve directions, and decide what to pursue. In Fully Autonomous mode, agents operate independently within predefined safety boundaries (spending limits, scope restrictions, approval requirements for high-value actions). The mode you choose is recorded onchain and visible to your stakeholders.

The practical implication is that your Lab can produce research output around the clock — scanning literature while you sleep, analysing data over weekends, identifying promising directions while you're focused on wet-lab work. The agent's compute costs come directly from your Lab's treasury, so there's no separate billing system. Research expenditure is a transparent line item in your Lab's onchain history.

For more details on the BIOS AI Scientist and MIRA, see the AI Tooling section.

### Step 5: Grow, License, and Compound

With data accumulating, IP registered, a community of IPT holders, and agents running continuous research cycles, your Lab enters a compounding phase. Every action feeds the next.

New data improves agent analyses. Better analyses strengthen IP positions. Stronger IP attracts more funding. More funding enables larger experiments. Larger experiments produce more data. The Lab's onchain track record grows with every cycle, making it progressively easier to attract collaborators, licensees, and capital.

Revenue can come from multiple directions: licensing IP to industry partners for upfront or royalty payments, charging access fees for premium datasets, receiving milestone payouts from funding organisations, or eventual acquisition of the Lab itself. Smart contracts handle royalty distributions and revenue sharing automatically — you set the terms, the protocol enforces them.

When the time comes, your Lab is fully portable. Sell it to a pharmaceutical company, pass it to a successor researcher, merge it into a larger research DAO, or nest it under a parent Lab that represents a multi-program research initiative. Because the Lab is an NFT, everything it holds transfers in a single transaction — data, IP, tokens, treasury, history, and reputation.

### Summary: What Changes for You

The traditional path: apply for grants, wait months, do research within rigid institutional frameworks, file patents through tech transfer offices that take years and large equity cuts, hope for licensing interest from industry, and repeat.

The Molecule path: create a Lab in seconds, upload your data, register IP you actually own, raise funds directly from a global community, put AI agents to work on your research continuously, and build a verifiable track record that follows you — not your institution — for your entire career.

Your science. Your Lab. Your terms.

{% embed url="https://desci-codes.gitbook.io/desci.codes/templates/model-agreements" %}

Refer to DeSci codes for more Legal and governance templates
