---
description: >-
  An agent plugin that runs the full Lab workflow — create an Onchain Lab,
  upload research data, and announce it — through AI coding agents
icon: wand-magic-sparkles
---

# Molecule Skill

### Overview

The Molecule skill lets AI agents execute the complete Lab lifecycle end-to-end — create an Onchain Lab, upload research files (public or encrypted), publish announcements, and manage roles — without a browser and without hand-written API calls.

It ships as a cross-harness agent plugin with two parts:

* **The `aura-orchestrator` skill** (`SKILL.md`) — a step-by-step runbook the agent follows: resolve or create an Onchain Lab (LabNFT plus its token-bound account), register it, upload files to the data room, announce, and optionally grant roles or hand the Lab off to another owner.
* **The `molecule` MCP server** — a typed [Model Context Protocol](https://modelcontextprotocol.io) server that performs every network, onchain, and cryptographic operation as a single tool call. Paid mutations are settled automatically through the [x402 Gateway](../api-reference/x402-gateway.md).

The skill format (`SKILL.md`) and MCP are open standards, so the same plugin works under Claude Code, OpenAI Codex, and any other MCP-capable agent harness.

{% hint style="info" %}
This is a different component from the read-only [MCP Tools](../references/mcp-tools.md) server, which answers ecosystem data questions (IPT prices, project activity). The Molecule skill's MCP server runs locally, holds your credentials, signs transactions, and writes to Labs.
{% endhint %}

### What the Skill Does

The workflow is sequential — each phase consumes the previous phase's output:

| Phase | Step                           | What happens                                                                                             |
| ----- | ------------------------------ | -------------------------------------------------------------------------------------------------------- |
| 0     | Wallet setup                   | The agent operates a wallet of your choice (see [Wallet Backends](molecule-skill.md#wallet-backends))     |
| 1     | Resolve or create the Lab      | Reuse a Lab the wallet already owns, or mint a new LabNFT with its token-bound account                    |
| 2     | Register the Lab               | `createLab` mutation, paid via x402                                                                       |
| 3     | Upload a file to the data room | Public (plaintext) or private (client-side encrypted, access-controlled)                                  |
| 4     | Announce                       | `createAnnouncement` mutation attaching the uploaded dataset, paid via x402                               |
| 5     | Grant roles / hand off         | Optionally grant a co-owner role or transfer the LabNFT to another wallet                                 |

#### Public vs. Private Uploads

Phase 3 is the only branch in the workflow:

* **Public** — the file is uploaded as-is with `accessLevel: PUBLIC`.
* **Private** — the file is encrypted client-side with AES-256-GCM before upload and finalized with encryption metadata plus onchain access conditions (a role on the Lab, or being an authorized signer of its token-bound account). Only wallets satisfying those conditions can later decrypt it — see [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md) for how access is evaluated.

Both paths are billed per mutation through the x402 Gateway; the private path additionally uses a service token for the key-management calls.

### MCP Server Tools

The `molecule` MCP server exposes typed tools grouped by concern:

| Group         | Tools (examples)                                                        | Purpose                                                                              |
| ------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Wallet        | `wallet_address`, `privy_*`, `eoa_send_transaction`                      | Dual-backend wallet operations: address lookup, transaction signing and sending      |
| Onchain reads | `ocl_read`, `ocl_tx_identity`                                           | Read Lab state (ownership, roles, mint fee) and parse mint receipts                  |
| Molecule API  | `labs_graphql`, `x402_pay`, `s3_upload`                                  | Labs API queries, paid mutations (the full x402 handshake in one call), file upload  |
| Encryption    | `labs_generate_dek`, `labs_decrypt_dek`, `encrypt_file`, `decrypt_file`  | Envelope encryption for private files                                                |
| Utilities     | `sha256_file`, `abi_encode`, `build_access_conditions`, `config_doctor`  | Hashing, calldata encoding, access-condition JSON, configuration diagnostics         |
| Bootstrap     | `issue_service_token`, `issue_owner_service_token`                       | Issue the service token used for key-management calls                                |

`config_doctor` reports which environment profile and wallet backend are active and names exactly which configuration is still missing, instead of letting a tool guess.

### Wallet Backends

Every signing and spending step works with either of two backends — you choose, and you can switch later with a configuration change:

* **Privy agentic wallet** — transactions are signed server-side via the [Privy](https://privy.io) API. No private key ever exists on your machine. The skill can create the wallet for you on first run, with a single-chain, value-capped policy.
* **Raw EOA** — you provide a private key via an environment variable; the MCP server signs locally and the key never leaves that process.

If only one backend is configured it is selected automatically; if both are configured you must pin the choice explicitly — the server refuses to guess which wallet to spend from.

{% hint style="warning" %}
The operating wallet pays real costs: USDC on Base for x402-billed mutations plus native gas for onchain transactions (LabNFT mint, role grants, transfers). Fund it before running the workflow.
{% endhint %}

### Configuration

All configuration and secrets are supplied as environment variables injected into the MCP server process by your agent harness. Tools read credentials from the environment — the agent passes file paths, queries, and addresses, not keys. The concrete values for staging and production (endpoints, chain, contract addresses) are provided with plugin access.

| Variable                                                     | Purpose                                                                                |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| `ENVIRONMENT`                                                | Deployment profile: `staging` (Base Sepolia) or `production` (Base)                     |
| `MOLECULE_LABS_URL`                                          | Labs API GraphQL endpoint                                                               |
| `MOLECULE_CLIENT_URL`                                        | Labs app base URL, used to build project links in announcements                         |
| `X402_GATEWAY_URL`                                           | x402 Gateway base URL                                                                   |
| `CHAIN_ID`                                                   | Chain of the selected environment                                                       |
| `EVM_RPC_URL`                                                | RPC endpoint for onchain reads and broadcasts (optional; falls back to a public node)   |
| `ONCHAIN_LAB_FACTORY_ADDRESS`, `LABNFT_ADDRESS`, `ACCESS_RESOLVER_ADDRESS` | Onchain Lab contract addresses for the selected environment — see the [Contracts reference](../references/contracts/) |
| `WALLET_BACKEND`                                             | Wallet backend selector: `privy` or `eoa` (auto-selected when only one is configured)   |
| `EVM_WALLET_ADDRESS`                                         | Watch-only address for reads and the optional hand-off target                           |
| `PRIVY_APP_ID`, `PRIVY_APP_SECRET`, `PRIVY_WALLET_ID`        | Privy backend credentials (secret)                                                      |
| `WALLET_PRIVATE_KEY`                                         | EOA backend private key (secret)                                                        |
| `MOLECULE_API_KEY`                                           | API key for Labs API queries (secret)                                                   |
| `MOLECULE_SERVICE_TOKEN`                                     | Service token for private-upload key management (secret)                                |

The wallet variables are all optional until you pick a backend — configure the Privy trio or the EOA key, not both (unless you pin `WALLET_BACKEND`).

### Security Model

The plugin is designed to keep secrets and confidential data out of the agent conversation:

* **Secrets stay in the environment.** Tools read credentials from environment variables; the agent passes file paths, queries, and addresses — no tool requires a key or token as an argument. The one deliberate exception is service-token bootstrapping: the `issue_service_token` tools return the issued JWT so you can store it in your harness's secret configuration. Prefer issuing it once during setup (and setting `MOLECULE_SERVICE_TOKEN`) over issuing per run, so the token stays out of agent transcripts.
* **The encryption key never leaves the server.** For private uploads, the data-encryption key is held in MCP server memory and referenced by an opaque, short-lived handle; the plaintext key is never returned to the agent, written to a file, or logged.
* **Fail-closed confidentiality.** Once a file enters the private upload path, the server refuses — for the lifetime of the server process, with no override flag — to upload that file's plaintext or to finalize it as public, even if the agent were instructed to. A failed private upload aborts; it never falls back to a public one.
* **Local encryption.** Files are encrypted with AES-256-GCM before upload, byte-for-byte compatible with the Labs client encryption, and verified by content hash after decryption.

### Installation

The plugin runs on any MCP-capable harness. The MCP server is a Python process launched via [`uv`](https://docs.astral.sh/uv/) or a plain virtualenv, with dependencies resolved automatically on first run.

* **Claude Code** — install the plugin directory (or add it from a plugin marketplace); the skill and the MCP server register automatically.
* **OpenAI Codex and other MCP hosts** — register the MCP server in the harness configuration and point the harness at the skill file.

Detailed, harness-specific installation steps and an offline smoke test ship with the plugin itself.

{% hint style="info" %}
**Getting access**: the plugin is distributed by the Molecule team. Join our [Discord community](https://t.co/L0VEiy4Bjk) and reach out to receive the plugin together with the configuration values for your environment.
{% endhint %}

### Related Pages

* [Molecule Labs](../technical-deep-dive/onchain-lab.md) — what an Onchain Lab is
* [Roles & Permissions](../technical-deep-dive/roles-and-permissions.md) — the role model used by access conditions
* [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md) — encryption and access evaluation in depth
* [Labs API](../api-reference/labs-api/README.md) — the GraphQL surface the skill drives
* [x402 Gateway](../api-reference/x402-gateway.md) — pay-per-call settlement for protected mutations
* [MCP Tools](../references/mcp-tools.md) — the read-only ecosystem-data MCP server
