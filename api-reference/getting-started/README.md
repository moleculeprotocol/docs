---
description: >-
  Pick the lane that fits your caller, gather the two prerequisites, and get a
  lab with a file in it inside ten minutes.
icon: rocket
---

# 🚀 Getting Started

This is the entry point to the Molecule API. It helps you **pick a lane**, lists the **two prerequisites** you actually need, and hands you a **ten-minute quickstart** that ends with a lab you can see. The three tutorials underneath it take the same ground step by step.

Everything here runs against **staging** (Base Sepolia, testnet funds). Nothing spends real money. The code every tutorial shares — config constants and helpers — is on [Shared Setup](shared-setup.md); the staging → production swap table is [further down this page](#running-in-production).

| |
| --- |
| [**Create a lab and upload a public file (start here)**](tutorial-1-public-upload.md) |
| [**Upload an encrypted file, verified with a decrypt round trip**](tutorial-2-encrypted-upload.md) |
| [**Add your agent as a contributor to your lab**](tutorial-3-agent-access.md) |

***

## Choose your path

Four ways to write to a Lab. They are not ranked — pick by who is calling.

| Your situation | Lane | Start here |
| -------------- | ---- | ---------- |
| **I run an AI coding agent** (Claude Code, Codex, Cursor) and want it to do the whole workflow | **Molecule Skill plugin** — a skill + MCP server that wraps every network, onchain and crypto operation as one tool call | [Molecule Skill](../../ai-tooling/molecule-skill.md) |
| **I'm scripting against the API** in Node/TypeScript and want to see the raw calls | **Raw GraphQL + viem** — self-issue a service token, mint, upload | [Tutorial 1](tutorial-1-public-upload.md) |
| **I have no credential, or I want to pay per call** instead of holding a long-lived token | **x402 gateway** — settle USDC on Base per request, no service token to provision | [x402 Gateway](../x402-gateway.md) |
| **I already made my lab in the app** (email sign-in, no wallet) **and now I want my agent writing into it** | **Agent-as-Contributor** — the human grants a role, the agent issues its own token | [Tutorial 3](tutorial-3-agent-access.md) |

The lanes are composable, so choosing one now doesn't lock you in to that lane only.

### If you are an agent reading this

Go to the [**Agent one-pager**](for-agents.md) instead. It is the whole default flow on a single page with no prose detours — written to be pasted into a system prompt.

***

## Prerequisites

Two things, and only one of them involves a human.

### 1. A `mol_` consumer credential — the one manual step

Every request to the API carries a consumer credential in the `Authorization` header. There is no self-service issuance yet (coming soon), so you will need to request this from the Molecule team.

Request it on the [Molecule Discord](https://t.co/L0VEiy4Bjk) with this template:

```
Consumer credential request
- Who: <your name / org>
- What you're building: <one line>
- Environment: staging   (add production if you need both)
- Contact: <Discord handle or email>
```

What comes back is a single opaque string per environment:

```
mol_<consumerId>_<secret>
```

Send it as the `Authorization` header value **directly — no `Bearer` prefix**:

```bash
Authorization: mol_your-consumer-id_your-secret
```

Treat the whole string as one secret: it is not split into a public and a private half. Full header reference: [Authentication](../authentication.md).

{% hint style="warning" %}
`Authorization: Bearer mol_…` fails authentication. `Bearer` is reserved for Privy user tokens.
{% endhint %}

### 2. A funded wallet on Base Sepolia

You need an EOA with testnet ETH for the LabNFT mint, only if you want to interact with the API programmatically. If you interact with out API through our frontend, you do not require any funds, we subsidize all the transactions.
Fund it from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets).

If — and only if — you are taking the **x402 lane**, you also need testnet **USDC** on Base Sepolia: get it from the [Circle faucet](https://faucet.circle.com/) (select Base Sepolia). The service-token lane needs no USDC at all.

You do **not** need a pre-issued service token. Every tutorial below mints its own from a wallet signature in the first step.

### What it costs

| Item | Cost  |
| ---- | ----  |
| **LabNFT mint** | Gas only. `mintFeeWei()` reads **0** on Base Sepolia **and** on Base mainnet, Read it live yourself using mintFeeWei(), and if non-zero, send it as value |
| **`createLab`, uploads and other content writes** (service-token lane) | Free |
| **The same mutations via x402** | Quoted per request in the `402` challenge — **$0.01 USDC** on both environments [Read the price off the challenge](../x402-gateway.md#reading-the-402-challenge); |
| **Storage** | 5 GB per lab included [Limits](../labs-api/files.md#storage-limits) |Ô

`mintFeeWei()` is a live contract read, not a constant. The tutorials call it and forward the result, so a future non-zero fee needs no code change on your side — but it will need funds.

### Tooling

```bash
npm install viem          # Node 18+ has fetch and node:crypto built in
```

Or, for the plugin lane, install [`mol-labs-plugin`](../../ai-tooling/molecule-skill.md#getting-the-plugin) and run `config_doctor` — it names exactly which configuration is still missing instead of letting a tool guess:

```bash
claude --plugin-dir /path/to/mol-labs-plugin
```

***

## Shared setup

Every tutorial opens with the same config constants and the same `graphql()` / `assertOk()` / `withIndexerLagRetry()` helpers. They live on their own page — copy them once and every tutorial snippet runs against them: [**Shared Setup**](shared-setup.md).

***

## Ten-minute quickstart

The shortest path from "I have a credential" to "there is a lab with my file in it". Four calls and one transaction. Each step is the condensed form of [Tutorial 1](tutorial-1-public-upload.md), which shows the expected response and the failure modes for every call.

```bash
export CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret"
export WALLET_PRIVATE_KEY="0x…"          # funded on Base Sepolia
```

1. **Self-issue a service token** — `getServiceSignInMessage` → sign the message with your wallet (EIP-191 `personal_sign`) → `generateServiceToken`. No human in the loop. Keep the three calls together: the message carries a single-use nonce valid for 10 minutes.
2. **Mint the LabNFT** — `OnChainLabFactory.mintAndCreateAccount(yourAddress)` with `value: mintFeeWei()`. Read `oclId` off the `OclIdentityCreated` event.
3. **Register the lab** — `createLab(input: { oclId })`.
4. **Upload a file** — `initiateCreateOrUpdateFile` → `PUT` the bytes to the returned presigned URL → `finishCreateOrUpdateFile` with `accessLevel: "PUBLIC"`.
5. **Verify it worked** — see below.

The runnable version is the [Tutorial 1 complete script](tutorial-1-public-upload.md#complete-script):

```bash
node tutorial-1.js ./research-data.csv
```

### Verify it worked

Two checks. The first works on every environment and needs nothing but your credential:

```bash
curl -s -X POST https://staging.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H "Authorization: $CONSUMER_CREDENTIAL" \
  -d '{
    "query": "query($oclId: String!) { labWithDataRoomAndFiles(oclId: $oclId) { oclId shortname name dataRoom { id files { path contentType accessLevel version } } } }",
    "variables": { "oclId": "0xYOUR_OCL_ID" }
  }'
```

Your file appears in `dataRoom.files` with the `path` you sent and `accessLevel: "PUBLIC"`. If `labWithDataRoomAndFiles` returns `null`, `createLab` did not complete — it is one of the two nullable queries on this API, so a missing lab nulls the field rather than throwing.

The second is visual — once `shortname` is populated, the lab has a page:

| Environment | Lab page |
| ----------- | -------- |
| Staging | `https://testnet.labs.molecule.xyz/projects/<shortname>` |
| Production | `https://labs.molecule.xyz/projects/<shortname>` |

`shortname` is derived server-side from the lab's name and is `null` until it has been derived — a freshly minted lab is reachable by `oclId` before it is reachable by slug.

***

## Then what

| Next | Page |
| ---- | ---- |
| Every step with expected responses and failure handling | [Tutorial 1 — public upload](tutorial-1-public-upload.md) |
| Encrypt a file so only wallets with a role can read it | [Tutorial 2 — encrypted upload](tutorial-2-encrypted-upload.md) |
| Let an agent write into a lab a human created in the app | [Tutorial 3 — agent access](tutorial-3-agent-access.md) |
| Pay per call instead of holding a token | [x402 Gateway](../x402-gateway.md) |
| Full operation reference | [Labs API](../labs-api/README.md) |
| What every error code means | [Error handling](../labs-api/README.md#error-handling) |

***

## Running in Production

All three tutorials run against staging (Base Sepolia, testnet funds). To run the same scripts against production, replace the values in the [shared setup](shared-setup.md) config block — nothing else changes, since every step reads from these constants:

| Constant | Staging (these tutorials) | Production |
| -------- | ------------------------- | ---------- |
| `GRAPHQL_URL` | `https://staging.graphql.api.molecule.xyz/graphql` | `https://production.graphql.api.molecule.xyz/graphql` |
| `CHAIN` (viem import) | `baseSepolia` from `viem/chains` | `base` from `viem/chains` |
| `FACTORY_ADDRESS` | `0xd629FE2310b4309a212495F10A47f8436dcEfD90` | `0xECdF4f05384056507485C90aeAb0a83268760D6E` |
| `LABNFT_ADDRESS` | `0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28` | `0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92` |
| `ACCESS_RESOLVER_ADDRESS` | `0x5493F472602C87318EA5Eff753cDD593bf9bF559` | `0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B` |
| `ACCESS_CONDITION_CHAIN` | `"baseSepolia"` | `"base"` |
| `LAB_APP_URL` | `https://testnet.labs.molecule.xyz` | `https://labs.molecule.xyz` |

```javascript
import { base } from "viem/chains"; // instead of baseSepolia

const GRAPHQL_URL = "https://production.graphql.api.molecule.xyz/graphql";
const CHAIN = base;
const FACTORY_ADDRESS = "0xECdF4f05384056507485C90aeAb0a83268760D6E";
const LABNFT_ADDRESS = "0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92";
const ACCESS_RESOLVER_ADDRESS = "0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B";
const ACCESS_CONDITION_CHAIN = "base";
const LAB_APP_URL = "https://labs.molecule.xyz";
```

A few things follow automatically from that swap:

* **Headers and the `graphql()` helper** are identical — `Authorization` (consumer credential, no `Bearer`) and the self-issued `X-Service-Token` work the same against both endpoints. Note that credentials are **per environment**: a staging `mol_` credential does not authenticate against production.
* **The `mintFeeWei()` read** already queries the live contract, so it picks up whatever fee production has configured with no code change. It reads `0` today on both chains.
* **The condition ABIs** are unchanged; only `contractAddress` and `chain` differ, and both come from the config block.

What doesn't follow automatically, and is on you:

* **Real funds.** Minting on `base` spends real ETH. Test on staging first.
* **Introspection is off in production** and query depth is capped at 10. Generate types against staging — see [Getting the schema](README.md#getting-the-schema).
* **`SERVICE_NAME`** should identify the real integration; it is echoed into the sign-in message and stored against the issued token.
* Full deployment list, including every other OCL contract on both chains: [Contracts reference](../../references/contracts/README.md).

***

## Getting the schema

**Staging has GraphQL introspection enabled** — point codegen, a playground or an SDK generator straight at it:

```
https://staging.graphql.api.molecule.xyz/graphql
```

```bash
# graphql-codegen, apollo, gql.tada… all work against staging
DESCI_API_SCHEMA=https://staging.graphql.api.molecule.xyz/graphql npx graphql-codegen
```

Introspection requires only the `Authorization` consumer-credential header, the same as any query.

**Production has introspection disabled**, deliberately — `__schema` and `__type` return a validation error there (`__typename` still resolves), and selection-set depth is capped at 10. Generate against staging and point the generated client at production; the two environments serve the same schema.

Production also enforces a query-depth limit of 10, which fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data — a plain GraphQL error, not the catalogued shape. Handle both.

***

## Endpoints

```
Staging:    https://staging.graphql.api.molecule.xyz/graphql
Production: https://production.graphql.api.molecule.xyz/graphql
```

x402 gateway base URLs are published on the [x402 Gateway](../x402-gateway.md#gateway-base-urls) page.
