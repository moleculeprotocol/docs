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

The skill format (`SKILL.md`) and MCP are open standards, so the same plugin works under Claude Code, OpenAI Codex, and any other MCP-capable agent harness. To obtain and install it, jump to [Getting the Plugin](molecule-skill.md#getting-the-plugin).

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

Both paths are billed per mutation through the x402 Gateway; the private path additionally uses a [service token](molecule-skill.md#the-service-token) for the key-management calls.

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

All configuration and secrets are plain **process environment variables** read by the MCP server subprocess — set them wherever your harness injects env into MCP servers (the `env` block of the MCP registration, or Claude Code's settings files as shown in [Installation](molecule-skill.md#claude-code)). Tools read credentials from the environment — the agent passes file paths, queries, and addresses, not keys.

Every non-secret value is published: the GraphQL endpoints on [API Overview](../api-reference/README.md), the [x402 Gateway base URLs](../api-reference/x402-gateway.md#gateway-base-urls), and the contract addresses in the [Contracts reference](../references/contracts/). The only thing you have to request is a `mol_` consumer credential — see [Getting Started](../api-reference/getting-started/README.md#1-a-mol-consumer-credential-the-one-manual-step) for the template.

**Ready-to-paste values per environment:**

| Variable | Staging | Production |
| -------- | ------- | ---------- |
| `ENVIRONMENT` | `staging` | `production` |
| `CHAIN_ID` | `84532` | `8453` |
| `MOLECULE_LABS_URL` | `https://staging.graphql.api.molecule.xyz/graphql` | `https://production.graphql.api.molecule.xyz/graphql` |
| `MOLECULE_CLIENT_URL` | `https://testnet.labs.molecule.xyz` | `https://labs.molecule.xyz` |
| `X402_GATEWAY_URL` | `https://0go1j7o645.execute-api.eu-central-2.amazonaws.com/prod` | `https://0qb5gyw72f.execute-api.eu-central-2.amazonaws.com/prod` |
| `ONCHAIN_LAB_FACTORY_ADDRESS` | `0xd629FE2310b4309a212495F10A47f8436dcEfD90` | `0xECdF4f05384056507485C90aeAb0a83268760D6E` |
| `LABNFT_ADDRESS` | `0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28` | `0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92` |
| `ACCESS_RESOLVER_ADDRESS` | `0x5493F472602C87318EA5Eff753cDD593bf9bF559` | `0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B` |

Run **`config_doctor`** after setting these: it reports which environment profile and wallet backend are active and names exactly which configuration is still missing, instead of letting a tool guess.

| Variable                                                     | Purpose                                                                                |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| `ENVIRONMENT`                                                | Deployment profile: `staging` (Base Sepolia) or `production` (Base)                     |
| `MOLECULE_LABS_URL`                                          | Labs API GraphQL endpoint for the chosen environment — see [API Overview](../api-reference/README.md) for the URLs |
| `MOLECULE_CLIENT_URL`                                        | Labs app base URL, used to build project links in announcements                         |
| `X402_GATEWAY_URL`                                           | x402 Gateway base URL (endpoint paths are documented on the [x402 Gateway](../api-reference/x402-gateway.md) page) |
| `CHAIN_ID`                                                   | `84532` (Base Sepolia) or `8453` (Base), matching `ENVIRONMENT`                         |
| `EVM_RPC_URL`                                                | RPC endpoint for onchain reads and broadcasts (optional; falls back to a public node)   |
| `ONCHAIN_LAB_FACTORY_ADDRESS`, `LABNFT_ADDRESS`, `ACCESS_RESOLVER_ADDRESS` | Onchain Lab contract addresses for the selected environment — see the [Contracts reference](../references/contracts/) |
| `WALLET_BACKEND`                                             | Wallet backend selector: `privy` or `eoa` (auto-selected when only one is configured)   |
| `EVM_WALLET_ADDRESS`                                         | Watch-only address for reads and the optional hand-off target                           |
| `PRIVY_APP_ID`, `PRIVY_APP_SECRET`, `PRIVY_WALLET_ID`        | Privy backend credentials (secret)                                                      |
| `WALLET_PRIVATE_KEY`                                         | EOA backend private key (secret)                                                        |
| `MOLECULE_CONSUMER_CREDENTIAL`                               | Your `mol_<consumerId>_<secret>` consumer credential for Labs API calls, sent as the `Authorization` header — see [Authentication](../api-reference/authentication.md) (secret) |
| `MOLECULE_API_KEY`                                           | Legacy shared API key — fallback only, while the `mol_` credential migration completes (secret) |
| `MOLECULE_SERVICE_TOKEN`                                     | Service token for private-upload key management (secret)                                |

The wallet variables are all optional until you pick a backend — configure the Privy trio or the EOA key, not both (unless you pin `WALLET_BACKEND`).

{% hint style="info" %}
**Use a `mol_` consumer credential.** The Labs API is moving from one shared API key to per-consumer credentials — a single `mol_<consumerId>_<secret>` string sent as the `Authorization` header with **no `Bearer` prefix** (see [Authentication](../api-reference/authentication.md)). Set it as `MOLECULE_CONSUMER_CREDENTIAL`; keep `MOLECULE_API_KEY` only if you still hold the legacy shared key. If both are set, the plugin sends both headers, so the same configuration works throughout the migration.
{% endhint %}

#### The Service Token

The service token is an **off-chain JWT bound to a wallet** — issued by signing a sign-in message with that wallet, not minted on chain. The skill needs it **only for private (encrypted) uploads**: the key-management calls that generate and decrypt the file's data-encryption key authenticate with it, while public uploads and all x402-paid mutations work without one.

Two things matter in practice:

* **Which wallet the token is bound to decides what it can decrypt.** The backend authorizes `decryptDataKey` against the token's bound wallet, so that wallet must satisfy the file's access conditions (a role on the Lab, or being an authorized signer of its token-bound account).
* **How to get one.** Preferably issue it once during setup and store it as `MOLECULE_SERVICE_TOKEN`. The plugin can do the issuance itself, matching your wallet backend: `issue_service_token` signs the sign-in message with the Privy agent wallet, `issue_owner_service_token` signs with the owner EOA. Both return the JWT for you to place in your harness's secret configuration. The underlying two-step GraphQL flow (plus extending and revoking tokens) is documented in [Service Token Management](../api-reference/labs-api/service-tokens.md).

If the token is missing or expired, the DEK tools fail with an error naming it — nothing falls back to an unauthenticated call.

### Security Model

The plugin is designed to keep secrets and confidential data out of the agent conversation:

* **Secrets stay in the environment.** Tools read credentials from environment variables; the agent passes file paths, queries, and addresses — no tool requires a key or token as an argument. The one deliberate exception is service-token bootstrapping: the `issue_service_token` tools return the issued JWT so you can store it in your harness's secret configuration. Prefer issuing it once during setup (and setting `MOLECULE_SERVICE_TOKEN`) over issuing per run, so the token stays out of agent transcripts.
* **The encryption key never leaves the server.** For private uploads, the data-encryption key is held in MCP server memory and referenced by an opaque, short-lived handle; the plaintext key is never returned to the agent, written to a file, or logged.
* **Fail-closed confidentiality.** Once a file enters the private upload path, the server refuses — for the lifetime of the server process, with no override flag — to upload that file's plaintext or to finalize it as public, even if the agent were instructed to. A failed private upload aborts; it never falls back to a public one.
* **Local encryption.** Files are encrypted with AES-256-GCM before upload, byte-for-byte compatible with the Labs client encryption, and verified by content hash after decryption.

### Getting the Plugin

The plugin is open source — install it from [moleculeprotocol/mol-labs-plugin](https://github.com/moleculeprotocol/mol-labs-plugin):

```bash
git clone https://github.com/moleculeprotocol/mol-labs-plugin.git
```

The one value you have to request is a `mol_` **consumer credential** — ask on our [Discord community](https://t.co/L0VEiy4Bjk) using the [template in Getting Started](../api-reference/getting-started/README.md#1-a-mol-consumer-credential-the-one-manual-step). Everything else — endpoints, gateway base URLs, contract addresses — is in the [Configuration](molecule-skill.md#configuration) table above. The repository layout:

```
mol-labs-plugin/
├── .claude-plugin/                     # Claude Code plugin manifest ("molecule-desci") + marketplace
├── .codex-plugin/                      # OpenAI Codex plugin manifest
├── .mcp.json                           # registers the "molecule" MCP server (uv run mcp/server.py)
├── skills/aura-orchestrator/SKILL.md   # the skill: the runbook the agent follows
└── mcp/server.py                       # the MCP server (Python, stdio transport)
```

The skill itself is a standard `SKILL.md` file — frontmatter that tells the harness when to use it, followed by the phase-by-phase runbook:

```yaml
---
name: aura-orchestrator
description: End-to-end DeSci molecule on the OCL (On-Chain Labs) surface —
  resolve-or-create an on-chain lab (LabNFT + token-bound account), register it,
  upload files (public or private/encrypted), and announce. Driven entirely
  through the `molecule` MCP server.
---
```

#### Prerequisite: `uv`

The MCP server is launched with [`uv`](https://docs.astral.sh/uv/), which reads the inline dependency header in `server.py` and provisions Python dependencies automatically on first run (a plain virtualenv works too — see the plugin's `mcp/README.md`):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # or: brew install uv
```

#### Claude Code

Load the plugin directory directly:

```bash
claude --plugin-dir /path/to/mol-labs-plugin
```

or install it straight from GitHub via the plugin marketplace:

```
/plugin marketplace add moleculeprotocol/mol-labs-plugin
/plugin install molecule-desci@molecule-desci-marketplace
```

The `molecule` MCP server registers automatically from the plugin's `.mcp.json`. Put the environment variables in your project's Claude Code settings — non-secrets in `.claude/settings.json`, secrets in `.claude/settings.local.json` (which stays out of version control), both under the `"env"` key:

```json
{
  "env": {
    "ENVIRONMENT": "staging",
    "MOLECULE_LABS_URL": "https://staging.graphql.api.molecule.xyz/graphql",
    "CHAIN_ID": "84532",
    "WALLET_BACKEND": "privy"
  }
}
```

Then run the skill: `/molecule-desci:aura-orchestrator` (attach or point it at the research file you want published).

#### OpenAI Codex

Register the MCP server in `~/.codex/config.toml` and give it the same environment:

```toml
[mcp_servers.molecule]
command = "uv"
args = ["run", "/path/to/mol-labs-plugin/mcp/server.py"]

[mcp_servers.molecule.env]
ENVIRONMENT = "staging"
MOLECULE_LABS_URL = "https://staging.graphql.api.molecule.xyz/graphql"
CHAIN_ID = "84532"
WALLET_BACKEND = "privy"
# ...plus the gateway URL and contract addresses from the Configuration table, and your secrets
```

Then copy `skills/aura-orchestrator/SKILL.md` into the skills directory your Codex version scans (check `/skills`), or surface it through `AGENTS.md`.

#### Other MCP hosts

Any harness that can spawn a stdio MCP server works — register it with the equivalent of:

```json
{
  "mcpServers": {
    "molecule": {
      "command": "uv",
      "args": ["run", "/path/to/mol-labs-plugin/mcp/server.py"],
      "env": { "ENVIRONMENT": "staging" }
    }
  }
}
```

#### Verify the install (offline, no secrets)

```bash
cd /path/to/mol-labs-plugin/mcp && uv run smoke.py
```

This lists every tool and exercises the pure-compute ones (encryption round-trip, ABI encoding, access-condition building) without any network access or credentials.

### Related Pages

* [Getting Started](../api-reference/getting-started/README.md) — the four lanes, prerequisites and costs; this plugin is the "agent runner" lane
* [Tutorials](../api-reference/getting-started/README.md) — the same workflow as raw GraphQL, if you want to see the calls underneath
* [Molecule Labs](../technical-deep-dive/onchain-lab.md) — what an Onchain Lab is
* [Roles & Permissions](../technical-deep-dive/roles-and-permissions.md) — the role model used by access conditions
* [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md) — encryption and access evaluation in depth
* [Labs API](../api-reference/labs-api/README.md) — the GraphQL surface the skill drives
* [x402 Gateway](../api-reference/x402-gateway.md) — pay-per-call settlement for protected mutations
* [MCP Tools](../references/mcp-tools.md) — the read-only ecosystem-data MCP server
