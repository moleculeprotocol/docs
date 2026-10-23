---
description: Molecule Insights & Research Assistant
icon: brain
---

# MIRA

## MIRA — Molecule Insights & Research Assistant <a href="#mira-molecule-insights-research-assistant" id="mira-molecule-insights-research-assistant"></a>

Oct 23, 2026 By [María Sanmartín](https://app.gitbook.com/u/zSGZqrXHbJODprA0p5ATmzNzP273 "mention")

### Overview <a href="#overview" id="overview"></a>

MIRA (Molecule Insights & Research Assistant) is Molecule's real-time AI assistant for the Molecule Protocol, working alongside Molecule Labs. It answers from the project and market data made public through the Molecule API, plus Molecule's own documentation, so answers are specific to Molecule rather than generic. MIRA has no access to data that is not public through the API.

MIRA covers two domains:

* **Public queries** — the protocol, the ecosystem and individual tokens, for anyone exploring Molecule.
* **Project analysis** — reading the scientific content in a Lab's data room, including the Technology Readiness evaluation.

It serves three groups:

| Audience              | What they use MIRA for                                     |
| --------------------- | ---------------------------------------------------------- |
| Researchers           | Understanding DeSci and navigating Labs and projects       |
| Traders and investors | Live token data, project insights and market intelligence  |
| Institutions          | Judging project maturity with a standard scoring framework |

MIRA is available at [mira.molecule.xyz](https://mira.molecule.xyz/), on molecule.xyz, and as a connector for Claude.

The current version extends the [first MIRA release](https://molecule.xyz/blog/genesis-of-molecules-ai-systems) with more live tools, ecosystem-wide queries next to project-level ones, and a standalone project evaluation framework.

### What MIRA can do <a href="#what-mira-can-do" id="what-mira-can-do"></a>

MIRA handles five kinds of request today; the AI picks the right tool for each question on its own.

| Capability             | What you get                                                                                                  | Example prompt                            |
| ---------------------- | ------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| Project briefing       | One summary of a project: data room content, price and market data, last month's events, recent team activity | "Give me a briefing on VITA-RNA"          |
| Ecosystem overview     | Every tracked token with current price, liquidity and holder count                                            | "What's live on Molecule right now?"      |
| Recent project updates | New files, announcements and team updates from a Lab's data room                                              | "What has VITA-FAST progress this month?" |
| Protocol questions     | Answers from vetted Molecule documentation, not generic web results                                           | "How does the API work?"                  |
| Buying a token         | A direct link to the right purchase page                                                                      | "I want to buy this project token"        |

#### Safeguards <a href="#safeguards" id="safeguards"></a>

* **Scoped answers.** MIRA answers only about Molecule's products and funding mechanisms, and politely declines off-topic questions.
* **Honest limits.** It says "I don't know" when unsure, defers to human experts, and never gives investment advice.
* **Source attribution.** Every answer links back to its sources.
* **Web search fallback.** When local knowledge is relevant but not enough, MIRA adds current information from trusted online sources.
* **Quality tracking.** Performance and user feedback are logged in Langfuse to improve answers.

### The MIRA stack <a href="#the-mira-stack" id="the-mira-stack"></a>

Questions arrive from molecule.xyz or a connected Claude project; the MIRA server routes them to its assistant tools or the TRL evaluation, which draw on Labs data rooms, market data and Molecule's own knowledge base.

<figure><img src="../.gitbook/assets/Screenshot 2026-10-23 at 10.22.51.png" alt=""><figcaption></figcaption></figure>

#### How a question is answered <a href="#how-a-question-is-answered" id="how-a-question-is-answered"></a>

MIRA checks its curated knowledge first, calls live tools or the web only when needed, and declines anything outside Molecule's scope.

<figure><img src="../.gitbook/assets/Screenshot 2026-10-23 at 10.23.16.png" alt=""><figcaption></figcaption></figure>

#### Tech stack <a href="#tech-stack" id="tech-stack"></a>

| Component                    | Role                                                      |
| ---------------------------- | --------------------------------------------------------- |
| OpenAI GPT-4o                | Reasoning, tool selection and response generation         |
| Model Context Protocol (MCP) | Connects to Molecule APIs for live token and project data |
| LanceDB                      | Vector search over curated documentation                  |
| Vercel AI SDK                | Response streaming and tool orchestration                 |
| Upstash Redis                | Cache for API responses                                   |
| Sanity CMS                   | Stores and publishes AI-generated project scores          |
| Langfuse                     | Performance metrics and user feedback                     |
| Tavily API                   | Web search beyond local knowledge                         |

### Project evaluation (Technology Readiness) <a href="#project-evaluation-technology-readiness" id="project-evaluation-technology-readiness"></a>

A standalone evaluation tool reads a project's data room files and tells funders and science teams where the project stands and what should happen next. The result appears on project pages as a Technology Readiness Level (TRL) score, using the scale NASA created. Code: [MIRA-trl-cron](https://github.com/moleculeprotocol/MIRA-trl-cron).

The evaluation runs in four steps, each building on the last, and can run in one session once the documents are in the data room.

1. **Classify maturity** — places the project at a TRL from what the documents show, not how the team describes it. Takes about 5–10 minutes. Output: Pre-TRL, TRL 1, 2 or 3.
2. **Score strength** — scores the project against weighted criteria for its stage. Output: a 0–5 score with a per-criterion breakdown.
3. **Recommend** — maps the score to a decision for funders and evaluators. Output: Promote, Supplement, Pivot or Pause, with a written rationale.
4. **Plan (optional)** — builds an advancement plan around the lowest-scoring criteria. Output: tasks, resources, timelines and risks.

#### Step 1: TRL levels <a href="#step-1-trl-levels" id="step-1-trl-levels"></a>

| Level   | Meaning                                                                                                            |
| ------- | ------------------------------------------------------------------------------------------------------------------ |
| Pre-TRL | Scientific rationale on paper; no experimental work yet; pure research with no technology lens.                    |
| TRL 1   | Basic research, reframed around how the science could underpin a technology.                                       |
| TRL 2   | Applied research: experiments underway or early data, not yet validated at scale.                                  |
| TRL 3   | Proof of concept shown in a controlled setting; enough evidence to move toward clinical or commercial development. |

#### Step 2: Scoring criteria (early feasibility stage) <a href="#step-2-scoring-criteria-early-feasibility-stage" id="step-2-scoring-criteria-early-feasibility-stage"></a>

Weights differ by stage, because what separates strong from weak projects changes over time. The weights below apply at early feasibility.

| Criterion                                    | Weight | Why                                                                                                         |
| -------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| Relevance of therapeutic mechanism of action | 45%    | The key question early on: does the mechanism work in the lab?                                              |
| Utility of candidates                        | 20%    | Specific molecules with reproducible activity start to matter, but stay secondary.                          |
| Intellectual property position               | 15%    | Becomes consequential by TRL 2, not yet decisive.                                                           |
| Prospects for safety                         | 15%    | Becomes consequential by TRL 2, not yet decisive.                                                           |
| Therapeutic optionality                      | 5%     | Multi-indication potential is speculative this early; a higher weight would reward narrative over evidence. |

#### Step 3: Recommendations <a href="#step-3-recommendations" id="step-3-recommendations"></a>

| Recommendation | When it applies                                                                                                        |
| -------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Promote        | Strong across key criteria; ready for the next TRL, a funding round or a formal development programme.                 |
| Supplement     | Sound science with a specific blocker (missing data, open IP question, team gap, regulatory uncertainty) to fix first. |
| Pivot          | The science or asset has merit, but the approach, indication or market thesis needs reframing.                         |
| Pause          | Fundamental weaknesses across several criteria that time or money won't fix; reallocate resources.                     |

#### Step 4: Advancement plan <a href="#step-4-advancement-plan" id="step-4-advancement-plan"></a>

The optional plan lists the experiments or milestones to reach the next TRL, the resources and expertise needed, rough timelines, the main risks with mitigations, and suggested partners. It is structured to drop straight into a grant application, an investor update or internal planning.

Teams raise their TRL score by uploading the relevant documentation to their data room.

#### Review before publication <a href="#review-before-publication" id="review-before-publication"></a>

Scores are not published straight from the model. Each one moves through a review pipeline: AI generation, then human validation, then publication through Sanity CMS.

### Version history <a href="#version-history" id="version-history"></a>

MIRA is on v2, Lab intel: the Molecule MCP, Lab data room analysis and TRL scoring. v1 was the chatbot.

| Version      | Focus            | What it added                                                                                                                                                                 |
| ------------ | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v3 (planned) | Knowledge graphs | Interactive maps linking projects, researchers and institutions; confidential data handling; token-gated access                                                               |
| v2 (current) | Lab intel        | Molecule MCP with Lab tools, data room analysis, four-step TRL evaluation with review pipeline, Claude connector                                                              |
| v1           | Chatbot          | Assistant grounded in official docs, protocol blogs and DeSci standards; live market data via MCP; embedded in the Molecule Screener with awareness of the token being viewed |

### Using MIRA <a href="#using-mira" id="using-mira"></a>

MIRA is live on [molecule.xyz](https://molecule.xyz) and can also be added to Claude.

**On molecule.xyz.** Ask about any project in the ecosystem, browse tokens by research area, or request a full briefing on a specific project.

**In Claude.** Connect MIRA to a Claude project and everyone in that project gets the live tools, with no per-conversation setup. Setup steps and the current configuration live in the [mira-knowledge repository](https://github.com/moleculeprotocol/mira-knowledge).

**For project teams.** Uploading documentation to your data room is the fastest way to raise your TRL score and unlock the full evaluation.

### Tools reference <a href="#tools-reference" id="tools-reference"></a>

MIRA uses the tools of the Molecule MCP. MIRA chooses which to call from the user's question. All project and market data comes from what the Molecule API makes public.

| Tool                               | What it returns                                                                                                                                           | Use it for                                                                        |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `molecule-knowledge-search`        | Answers drawn from [Molecule docs](https://docs.molecule.xyz/), the [blog](https://molecule.xyz/blog) and [DeSci.Codes](https://www.desci.codes/)         | Conceptual questions: IP-NFTs, project tokens, Proof of Invention, ecosystem news |
| `molecule-get-labs`                | All projects with verification status, TRL, format and therapeutic fields, plus each project token's price, holders, market cap, liquidity and 24h volume | An overview of the ecosystem                                                      |
| `molecule-get-lab-categories`      | Projects grouped by therapeutic field (Longevity, Brain Health, Women's Health…) with TRL and market data                                                 | Browsing by research area                                                         |
| `molecule-get-lab-summary`         | One brief: research focus, team, TRL, assessment score, funding, links, project token market data with 14-day price history, recent data room activity    | A full picture of one project                                                     |
| `molecule-get-lab-activity`        | Data room announcements and files added, updated or removed, newest first                                                                                 | What changed recently in a project                                                |
| `molecule-get-token-price-history` | Daily price history (OHLCV, USD) of a project token, up to 90 days                                                                                        | Price trends over time                                                            |

Projects can be looked up by slug (e.g. `vita-fast`), name or token symbol.

### Related

* [Funders](../user-guides/investors.md) — how funders use MIRA's assessments
* [Scientists](../user-guides/scientists-researchers.md) — MIRA from the researcher's side

{% include "../.gitbook/includes/support.md" %}
