---
description: >-
  Pick the lane that fits your caller, gather the two prerequisites, and get a
  lab with a file in it inside ten minutes.
icon: rocket
---

# 🚀 Getting Started

This is the entry point to the Molecule API. It does three things: helps you **pick a lane**, lists the **two prerequisites** you actually need, and walks a **ten-minute quickstart** that ends with a lab you can see.

Everything below runs against **staging** (Base Sepolia, testnet funds). Nothing here spends real money. The [staging → production swap table](../labs-api/example-workflow.md#running-in-production) is at the end of the tutorials.

***

## Choose your path

Four ways to write to a Lab. They are not ranked — pick by who is calling.

| Your situation | Lane | Start here |
| -------------- | ---- | ---------- |
| **I run an AI coding agent** (Claude Code, Codex, Cursor) and want it to do the whole workflow | **Molecule Skill plugin** — a skill + MCP server that wraps every network, onchain and crypto operation as one tool call | [Molecule Skill](../../ai-tooling/molecule-skill.md) |
| **I'm scripting against the API** in Node/TypeScript and want to see the raw calls | **Raw GraphQL + viem** — self-issue a service token, mint, upload | [Tutorial 1](../labs-api/example-workflow.md#tutorial-1-create-a-lab-and-upload-a-public-file) |
| **I have no credential, or I want to pay per call** instead of holding a long-lived token | **x402 gateway** — settle USDC on Base per request, no service token to provision | [x402 Gateway](../x402-gateway.md) |
| **I already made my lab in the app** (email sign-in, no wallet) **and now I want my agent writing into it** | **Agent-as-Contributor** — the human grants a role, the agent issues its own token | [Tutorial 3](../labs-api/example-workflow.md#tutorial-3-give-your-agent-access-to-a-lab-you-created-in-the-app) |

The lanes compose. A common shape is the plugin lane for the workflow plus x402 for the paid mutations, which is exactly what the plugin does by default.

### If you are an agent reading this

Go to the [**Agent one-pager**](for-agents.md) instead. It is the whole default flow on a single page with no prose detours — written to be pasted into a system prompt.

***

## Prerequisites

Two things, and only one of them involves a human.

### 1. A `mol_` consumer credential — the one manual step

Every request to the API carries a consumer credential in the `Authorization` header. There is no self-service issuance yet, so this is the single "ask the team" left in the API docs.

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

You need an EOA with testnet ETH for the LabNFT mint. Fund it from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets).

If — and only if — you are taking the **x402 lane**, you also need testnet **USDC** on Base Sepolia: get it from the [Circle faucet](https://faucet.circle.com/) (select Base Sepolia). The service-token lane needs no USDC at all.

You do **not** need a pre-issued service token. Every tutorial below mints its own from a wallet signature in the first step.

### What it costs

| Item | Cost | How we know |
| ---- | ---- | ----------- |
| **LabNFT mint** | Gas only. `mintFeeWei()` reads **0** on Base Sepolia **and** on Base mainnet (verified 2026-08-27 by `eth_call`) | Read it live yourself — the tutorials do, and send it as `value` |
| **`createLab`, uploads, announcements** (service-token lane) | Free | Consumer credential + self-issued service token |
| **The same mutations via x402** | Quoted per request in the `402` challenge — **$0.01 USDC** on both environments today | [Read the price off the challenge](../x402-gateway.md#reading-the-402-challenge); never hardcode it |
| **Storage** | 5 GB per lab included | [Limits](../labs-api/files.md#storage-limits) |

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

## Ten-minute quickstart

The shortest path from "I have a credential" to "there is a lab with my file in it". Four calls and one transaction. Each step is the condensed form of [Tutorial 1](../labs-api/example-workflow.md#tutorial-1-create-a-lab-and-upload-a-public-file), which shows the expected response and the failure modes for every call.

```bash
export CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret"
export WALLET_PRIVATE_KEY="0x…"          # funded on Base Sepolia
```

1. **Self-issue a service token** — `getServiceSignInMessage` → sign the message with your wallet (EIP-191 `personal_sign`) → `generateServiceToken`. No human in the loop.
2. **Mint the LabNFT** — `OnChainLabFactory.mintAndCreateAccount(yourAddress)` with `value: mintFeeWei()`. Read `oclId` off the `OclIdentityCreated` event.
3. **Register the lab** — `createLab(input: { oclId })`.
4. **Upload a file** — `initiateCreateOrUpdateFile` → `PUT` the bytes to the returned presigned URL → `finishCreateOrUpdateFile` with `accessLevel: "PUBLIC"`.
5. **Verify it worked** — see below.

The runnable version is the [Tutorial 1 complete script](../labs-api/example-workflow.md#tutorial-1-complete-script):

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
| Every step with expected responses and failure handling | [Tutorial 1 — public upload](../labs-api/example-workflow.md#tutorial-1-create-a-lab-and-upload-a-public-file) |
| Encrypt a file so only wallets with a role can read it | [Tutorial 2 — encrypted upload](../labs-api/example-workflow.md#tutorial-2-upload-an-encrypted-file) |
| Let an agent write into a lab a human created in the app | [Tutorial 3 — agent access](../labs-api/example-workflow.md#tutorial-3-give-your-agent-access-to-a-lab-you-created-in-the-app) |
| Publish an update that attaches the dataset | [Tutorial 4 — announce](../labs-api/example-workflow.md#tutorial-4-announce-the-dataset) |
| Pay per call instead of holding a token | [x402 Gateway](../x402-gateway.md) |
| Full operation reference | [Labs API](../labs-api/README.md) |
| What every error code means | [Error handling](../labs-api/README.md#error-handling) |

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
