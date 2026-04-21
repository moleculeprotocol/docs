---
description: Discover, evaluate, and invest in tokenized scientific research
icon: user-tie
---

# Investors

### Who This Guide Is For

You're interested in scientific research as an investable asset class. You might be a retail supporter who wants to fund a longevity project for $50. You might be a biotech venture firm evaluating early-stage IP for licensing or acquisition. You might be a DAO looking to deploy treasury into DeSci. Whatever your scale and strategy, Molecule gives you direct access to scientific research at every stage — from an unproven hypothesis to a licensable drug candidate — with transparent onchain data and AI-powered evaluation tools to help you make informed decisions.

This guide covers three investor profiles: community supporters who fund Labs through token purchases, strategic investors who evaluate projects for larger commitments like licensing deals or Lab acquisitions, and institutional buyers exploring M\&A, technology transfer, or IP licensing at scale. The platform serves all three through different interaction surfaces.

### Understanding the Ecosystem

Before diving into platform mechanics, it helps to understand how the pieces fit together — this is one of the most common points of confusion for new investors.

Molecule is the protocol and platform layer. It provides the smart contract infrastructure (IP-NFTs, IP Tokens, Labs, data rooms) and the user-facing application where you discover, evaluate, and invest in research. Molecule the company also builds and maintains MIRA, the AI research assistant.

BIO Protocol is the network layer above Molecule. BIO coordinates capital and governance across the DeSci ecosystem through bioDAOs — community-governed organisations focused on specific therapeutic areas. VitaDAO (longevity), CerebrumDAO (neuroscience), and others each curate and fund research within their domain using Molecule's infrastructure. The BIO token governs the network; individual bioDAOs have their own governance tokens.

When you invest in IPTs on the Molecule platform, your economic exposure is to a specific Lab's intellectual property. That Lab may be affiliated with a bioDAO (e.g., funded through a VitaDAO grant), but your IPT position is tied to the Lab itself, not to VitaDAO or BIO. Revenue from licensing deals, milestone payments, or IP commercialisation flows to the Lab's treasury and is governed by IPT holders. BIO and bioDAO tokens represent exposure to their respective networks and treasuries — they are separate positions with separate governance.

This distinction matters because it determines where value accrues. If a Lab's drug candidate gets licensed, the licensing fees go to that Lab's treasury (benefiting IPT holders). If a bioDAO's portfolio of Labs collectively succeeds, that benefits the bioDAO's token holders. If the DeSci ecosystem grows broadly, that benefits BIO holders. You can participate at whichever level matches your thesis.

### Discovering Labs and Projects

The Molecule platform is your starting point for finding research worth supporting. It surfaces every active Lab in the ecosystem with its project description, data room activity, funding history, token market data, and AI-generated project scores.

Browse by research category to find projects in your areas of interest. Categories span therapeutic areas, research methodologies, and scientific domains. Each project listing shows the Lab's IP-NFT symbol, its current IPT price and market cap (if tokenized), treasury balance, data room size, and recent activity.&#x20;

The Investor Portfolio view — a dedicated interface designed specifically for investors — provides a consolidated view of your positions alongside discovery tools, letting you track your holdings and find new opportunities from a single dashboard. Activity feeds surface real-time updates: new data room uploads, announcements, milestone completions, and token market movements across Labs you follow.

The platform's search is semantic — you can query in natural language. Searching for something like "gene therapy for rare diseases" returns Labs whose data rooms, announcements, and metadata match your intent, not just keyword matches. This makes it practical to explore the ecosystem by research thesis rather than by browsing token listings.

Labs also publish Shareable Lab Updates — formatted progress reports that Lab owners push to their community. These updates appear in activity feeds and provide structured summaries of recent milestones, data room additions, and next steps. For investors monitoring a portfolio of positions, these updates are a reliable signal of continued research activity without needing to dig into raw data room contents each time.

### Evaluating Projects with MIRA

Before committing capital, you want to understand what a project is actually worth. This is where MIRA — Molecule's Insights & Research Assistant — becomes your primary evaluation tool.

MIRA is an AI-powered research assistant embedded directly into the platform. It combines a curated knowledge base about DeSci, real-time market data via MCP tools, and an AI-powered project scoring system. You can interact with it conversationally — ask it about a specific project, request a summary of recent activity, compare projects in the same therapeutic area, or get an explanation of a Lab's funding structure.

The most relevant feature for investors is MIRA's project scoring. MIRA analyses a Lab's data room contents and generates a Technology Readiness Level (TRL) assessment alongside weighted evaluation scores covering therapeutic relevance, optionality, and IP strength. These scores are generated by AI, reviewed by human experts, and published through a structured pipeline. The TRL score and rationale are displayed directly on each project's page, giving you a standardised framework for comparing projects at a glance.

TRL is a well-established framework used across biotech and pharmaceuticals. A project at TRL 1–2 is at the concept or hypothesis stage. TRL 3–4 means experimental proof of concept. TRL 5–6 indicates validated results in relevant environments. Higher levels reflect clinical validation and commercialisation readiness. MIRA maps each Lab's actual research output — papers, datasets, experimental results, IP filings — to this scale, so the score reflects real scientific progress rather than marketing claims.

Beyond scoring, MIRA provides real-time market intelligence. Ask it for a project's price history, holder distribution, liquidity depth, or recent data room changes. It knows which token you're currently viewing and tailors its responses to that context. It will not provide investment advice — that boundary is deliberate — but it will give you the data and analysis to form your own view.

MIRA is also available as a Discord bot for community-level discussions and analysis, and its capabilities are being continuously expanded. The current v2 focuses on scoring and data room analysis. The planned v3 introduces knowledge graphs that map relationships between projects, researchers, and institutions across the ecosystem — enabling you to trace collaboration networks, identify shared methodologies, and discover related research you might otherwise miss.

### Funding Labs: Community Participation

For community supporters and retail investors, the primary way to participate is through IP Token (IPT) purchases on the secondary market or through structured fundraising events.

When a Lab tokenizes its IP-NFT, it creates IP Tokens that represent fractional ownership and governance rights over the underlying intellectual property. Buying IPTs gives you economic exposure to the research's success, access to the Lab's token-gated data room (where premium research data lives), governance rights over key decisions about the IP's direction, and a liquid position that you can trade on secondary markets at any time.

IPTs trade on decentralised exchanges with real-time price discovery. You can buy or sell at any time, meaning you're not locked into a multi-year commitment. If a project hits a major milestone — a successful experiment, a licensing deal, a regulatory filing — the market reprices the token in real time. If your investment thesis changes, you can exit.

The platform supports structured fundraising through Catalyst campaigns, which use a bonding curve mechanism to fund early-stage research. Contributors purchase seed tokens that convert to IPTs if and when the project reaches a negotiated IP agreement. If the negotiation doesn't succeed, contributors are refunded through the curve. This structure lets you back promising research directions before formal IP exists, with built-in downside protection.

All contributions flow directly into the Lab's treasury. You can see exactly how funds are deployed — treasury balances, transaction history, and spending patterns are all visible onchain. There's no opacity about where the money went.

For a technical reference on token mechanics, see the IPT and SynthesisModule contract pages in the References section.

### Strategic Investment: Licensing and Partnerships

For venture firms, pharmaceutical companies, and bioDAOs evaluating larger commitments, the relevant interaction surfaces go beyond token purchases.

Every Lab's onchain history is a due diligence asset. You can inspect the complete audit trail: when the Lab was created, what data has been uploaded and how often, which IP-NFTs have been minted, how treasury funds have been deployed, what agent activity has occurred, and the full history of announcements and file updates. This replaces the fragmented, document-heavy due diligence process in traditional biotech with a single inspectable onchain record. MIRA's TRL scoring provides a standardised starting point, and the raw data room contents let you drill into the science itself.

IP licensing on Molecule is designed to be executed onchain. The architecture supports rentable IP-NFTs using the ERC-4907 standard — a pharmaceutical company or research institution can license an IP-NFT for a defined period by executing a single transaction. Rental fees are paid in stablecoins and automatically routed to the Lab's treasury. Royalties are distributed to the original scientist, the institution, and other stakeholders via smart contracts. This removes months of legal negotiation for standard licensing terms, though complex deals may still require off-chain agreements.

For investors evaluating M\&A or technology transfer, the Lab NFT itself is the acquisition mechanism. Under the V3 architecture, each Lab is an ERC-721 NFT with an ERC-6551 Token Bound Account (TBA) — meaning the Lab is a self-contained onchain entity that owns its own assets. Purchasing a Lab's NFT transfers control of everything inside it in a single transaction: the IP-NFTs, the token treasury, the data room references, the onchain history, and any installed modules or agent configurations. This is the onchain equivalent of acquiring a biotech company, but without the administrative overhead of entity transfers, shareholder votes, or escrow processes. The Lab's identity (its permanent TBA address) stays the same; only the controller changes.

For any of these larger-scale interactions, the Molecule team can facilitate introductions, structure deals, and provide technical support. Reach out through the Molecule Discord or directly to the team.

### Governance and Stakeholder Rights

Holding IPTs is not a passive position. Token holders participate in governance decisions that affect the Lab's direction and their investment.

IPT holders can vote on key decisions: approving or rejecting milestone completions, setting terms for licensing deals, directing treasury allocation, and — in cases where a project stalls — initiating clawback proceedings to recover treasury funds. The clawback mechanism is specifically designed to protect investors: if a Lab demonstrates prolonged inactivity or fails to meet minimum progress thresholds, IPT holders can vote to reallocate treasury funds — either returning them pro-rata to holders or redirecting them to an alternative Lab pursuing related research.

Clawback proposals require a supermajority to pass (preventing hostile takeovers of temporarily dormant projects), and the Lab owner receives a grace period to demonstrate renewed activity before execution. This structure ensures that capital can't be permanently locked in non-productive projects while still giving researchers reasonable time to work.

Beyond formal governance, IPT holders get access to token-gated data rooms. The minimum token holding required for data room access is set by the Lab owner — this threshold is visible on the project page so you know exactly what position size grants you access to premium research data. Inside the data room, you can see actual research progress: raw datasets, experimental results, methodological notes, and confidential files that aren't visible to non-holders. This gives you a genuine information advantage over participants who only see public data.

### Understanding Risk

Investing in early-stage science carries inherent uncertainty. Most research hypotheses don't pan out. Drug candidates fail clinical trials. Datasets turn out to be less valuable than expected. Molecule's infrastructure makes these risks more transparent and manageable — you can see exactly what a project is doing, how it's spending funds, and where it stands scientifically — but it doesn't eliminate them.

**Research risk.** The fundamental risk in any scientific investment is that the research doesn't produce the expected results. Even well-designed experiments fail, and even promising compounds fail in clinical trials. TRL scores reflect current scientific progress, not probability of future success.

**Market and liquidity risk.** IPT prices reflect market sentiment and can be volatile, particularly for early-stage projects with small float. Liquidity varies significantly across projects — larger, more established Labs will have deeper order books than newly tokenized projects. Check holder distribution and trading volume before sizing a position.

**Regulatory risk.** DeSci operates at the intersection of crypto and biotech, both of which face evolving regulatory landscapes. IP token classification, securities considerations, and cross-jurisdictional differences are active areas of development across the industry.

**Current limitations.** IPT ownership currently provides governance and economic exposure but not legal equity in a traditional sense. The RWA Equity module — which will bridge IPT ownership to real-world shareholder rights and regulatory-compliant equity structures — is actively under development and represents a key part of the roadmap for institutional investors. Until it's live, IPT positions are best understood as onchain governance tokens with economic exposure to the Lab's IP commercialisation, not as shares in a legal entity.

The onchain track record is your best tool for managing these risks. Labs that consistently upload data, hit milestones, deploy treasury responsibly, and maintain active communities are demonstrably lower-risk than Labs with sparse activity. MIRA's scoring reflects this. Use it as a starting point, dive into the data room for substance, and let the onchain history speak for itself.

### What's Coming for Investors

The platform is actively evolving. Several features in development are directly relevant to investors:

The **RWA Equity module** will create a bridge between onchain IPT positions and real-world equity rights, making Molecule investable for institutional allocators who require legal ownership structures. This is currently in the design and early implementation phase.

**Group Tokens** (part of the V3 architecture) will enable Lab-level tokenization — instead of buying into individual IP-NFTs, you'll be able to hold tokens representing an entire Lab's portfolio of IP assets. This is analogous to buying into a fund rather than a single asset, providing diversified exposure to a research team's full output.

**MIRA v3's knowledge graphs** will map the relationships between projects, researchers, institutions, and funding sources across the ecosystem, giving you tools to identify emerging research clusters, track researcher track records across Labs, and discover investment opportunities based on network analysis rather than just individual project metrics.

**Enhanced licensing infrastructure** using ERC-4907 will make IP licensing transactions executable in a single onchain step, with automatic royalty distribution to all stakeholders — reducing the time and cost of deal execution for strategic investors.

### Getting Started

Connect a wallet to the Molecule platform and start browsing Labs. No tokens or prior setup required to explore — project pages, public data rooms, MIRA scores, and market data are all accessible immediately.

When you find a project worth supporting, you can purchase IPTs on secondary markets instantly or participate in a Catalyst campaign when one is active. For larger commitments — licensing inquiries, Lab acquisitions, or strategic partnerships — reach out to the Molecule team through Discord.

Use MIRA as your research assistant throughout the process. Ask it anything about a project, a token, a research area, or the ecosystem as a whole. It's designed to give you the context you need to make informed decisions.
