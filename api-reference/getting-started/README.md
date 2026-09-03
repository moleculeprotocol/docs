---
description: >-
  What a Lab is, what you need before your first call, and a ten-minute path to
  a lab with a file in it.
icon: rocket
---

# 🚀 Getting Started

The Molecule API lets you create a **Lab** — a research project with its own onchain identity and its own file store — and then read and write that Lab's files from code.

This page covers the two things you need before your first call, which way of calling the API fits you, and a ten-minute path that ends with a lab you can open in a browser.

{% hint style="info" %}
**New to the Molecule ecosystem?** The [Glossary](../../references/glossary.md) defines every term used in these guides — Lab, LabNFT, oclId, data room, service token, indexer — in a sentence or two each. Worth keeping open in a second tab.
{% endhint %}

Everything here runs against **staging** (Base Sepolia, testnet funds), so nothing spends real money. Moving to mainnet later is a matter of swapping a handful of constants: [Running in Production](#running-in-production).

***

## Choose how you'll call the API

Three ways in. None is better than the others — pick by who is making the calls.

| If this is you | What you'll use | Start here |
| -------------- | --------------- | ---------- |
| **You're writing a script** in Node/TypeScript and want to see the raw calls | GraphQL requests plus [viem](https://viem.sh) for the one onchain step | [The tutorials](#the-tutorials) below |
| **You run an AI coding agent** (Claude Code, Codex, Cursor) and want it to do the whole workflow for you | The Molecule Skill plugin, which wraps every network, onchain and crypto operation as a single tool call | [Molecule Skill](../../ai-tooling/molecule-skill.md) |
| **You'd rather pay per call** than hold a long-lived credential | The x402 gateway, which settles USDC on Base per request | [x402 Gateway](../x402-gateway.md) |

These combine rather than compete — choosing one now doesn't lock you out of the others. A common setup is the plugin for the workflow and x402 for the calls that cost money.

### If you are an agent reading this

Go to the [**Agent one-pager**](for-agents.md) instead. It is the whole default flow on a single page with no prose detours — written to be pasted into a system prompt.

***

## The tutorials

Each one is runnable end to end against staging, and shows the expected response and the failure modes at every step.

| Tutorial | What you have when you finish |
| -------- | ----------------------------- |
| [**Create a lab and upload a file**](create-lab-and-upload-file.md) — start here | A Lab of your own, with a public file in it |
| [**Upload an encrypted file**](upload-encrypted-file.md) | A confidential file that only certain people that you specify can decrypt |
| [**Agent as a lab contributor**](agent-as-a-lab-contributor.md) | An agent writing into a Lab that a human owns |

All three open with the same configuration constants and helper functions, which live on one page: [**Shared Setup**](shared-setup.md). Copy that block once and every snippet in the tutorials runs against it.

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

**You only need this if you are creating a new Lab from code.** Creating a Lab means minting a LabNFT, which is an onchain transaction, and the wallet that sends it pays the gas. If you create your Lab in the Molecule app instead, you need no funds at all — Molecule covers those transactions for you. The app is at [labs.molecule.xyz](https://labs.molecule.xyz/), or [testnet.labs.molecule.xyz](https://testnet.labs.molecule.xyz/) for the Base Sepolia environment these tutorials run against.

For the programmatic path you need an [EOA](../../references/glossary.md#calling-the-api) — an ordinary wallet with a private key — holding testnet ETH on Base Sepolia. Fund it from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets).

If — and only if — you are using the **x402 gateway**, you also need testnet **USDC** on Base Sepolia: get it from the [Circle faucet](https://faucet.circle.com/) (select Base Sepolia). Paying with a service token needs no USDC at all.

You do **not** need a pre-issued **service token** — it is proof that you control a particular wallet, and every tutorial issues its own by signing a message in its first step. What it is, what it authorizes and how it is sent: [Authentication](../authentication.md#what-a-service-token-actually-authorizes).

### What it costs

| Item | Cost |
| ---- | ---- |
| **LabNFT mint** | Gas only, on Base Sepolia **and** on Base mainnet. Read the fee live with `mintFeeWei()` and, if it is ever non-zero, send it as `value` — the tutorials already do this, so a future fee needs no code change on your side |
| **`createLab`, uploads and other content writes** | Free |
| **The same mutations through the x402 gateway** | Quoted per request in the `402` challenge, currently **$0.01 USDC** on both environments. [Read the price off the challenge](../x402-gateway.md#reading-the-402-challenge) rather than hardcoding it |
| **Storage** | 5 GB per lab included — see [Limits](../labs-api/files.md#storage-limits) |

### Tooling

```bash
npm install viem          # Node 18+ has fetch and node:crypto built in
```

Or, if you are using the Molecule Skill plugin, install [`mol-labs-plugin`](../../ai-tooling/molecule-skill.md#getting-the-plugin) and run `config_doctor` — it names exactly which configuration is still missing instead of letting a tool guess:

```bash
claude --plugin-dir /path/to/mol-labs-plugin
```

***

## Ten-minute quickstart

The shortest path from "I have a credential" to "there is a lab with my file in it". Four API calls and one transaction. Each step below is the condensed form of [Create a lab and upload a file](create-lab-and-upload-file.md), which shows the expected response and the failure modes for every call.

```bash
export CONSUMER_CREDENTIAL="mol_your-consumer-id_your-secret"
export WALLET_PRIVATE_KEY="0x…"          # funded on Base Sepolia
```

1. **Issue yourself a service token** — `getServiceSignInMessage` → sign the message with your wallet (EIP-191 `personal_sign`) → `generateServiceToken`. No human in the loop. Keep the three calls together: the message carries a single-use nonce valid for 10 minutes.
2. **Mint the LabNFT** — `OnChainLabFactory.mintAndCreateAccount(yourAddress)` with `value: mintFeeWei()`. Read `oclId` off the `OclIdentityCreated` event.
3. **Register the lab** — `createLab(input: { oclId })`. This attaches the data room your files will live in.
4. **Upload a file** — `initiateCreateOrUpdateFile` → `PUT` the bytes to the returned presigned URL → `finishCreateOrUpdateFile` with `accessLevel: "PUBLIC"`.
5. **Verify it worked** — see below.

The runnable version is the [complete script](create-lab-and-upload-file.md#complete-script):

```bash
node create-lab-and-upload-file.js ./research-data.csv
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

Your file appears in `dataRoom.files` with the `path` you sent and `accessLevel: "PUBLIC"`. If `labWithDataRoomAndFiles` comes back `null`, `createLab` did not complete — a missing lab nulls the field rather than throwing an error.

The second check is visual — once `shortname` is populated, the lab has a page of its own:

| Environment | Lab page |
| ----------- | -------- |
| Staging | `https://testnet.labs.molecule.xyz/projects/<shortname>` |
| Production | `https://labs.molecule.xyz/projects/<shortname>` |

`shortname` is derived server-side from the lab's name and is `null` until that has happened, so a freshly minted lab is reachable by `oclId` before it is reachable by slug.

***

## Running in Production

The tutorials all run against staging (Base Sepolia, testnet funds). To run the same scripts against production, replace the values in the [Shared Setup](shared-setup.md) config block — nothing else changes, because every step reads from these constants:

| Constant | Staging (these tutorials) | Production |
| -------- | ------------------------- | ---------- |
| `GRAPHQL_URL` | `https://staging.graphql.api.molecule.xyz/graphql` | `https://production.graphql.api.molecule.xyz/graphql` |
| `CHAIN` (viem import) | `baseSepolia` from `viem/chains` | `base` from `viem/chains` |
| `FACTORY_ADDRESS` | `0xd629FE2310b4309a212495F10A47f8436dcEfD90` | `0xECdF4f05384056507485C90aeAb0a83268760D6E` |
| `LABNFT_ADDRESS` | `0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28` | `0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92` |
| `ACCESS_RESOLVER_ADDRESS` | `0x5493F472602C87318EA5Eff753cDD593bf9bF559` | `0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B` |
| `ACCESS_CONDITION_CHAIN` | `"baseSepolia"` | `"base"` |
| `LAB_APP_URL` | `https://testnet.labs.molecule.xyz` | `https://labs.molecule.xyz` |

A few things follow automatically from that swap:

* **Headers and the `graphql()` helper** are identical — `Authorization` (consumer credential, no `Bearer`) and the self-issued `X-Service-Token` work the same against both endpoints. Note that credentials are **per environment**: a staging `mol_` credential does not authenticate against production.
* **The `mintFeeWei()` read** already queries the live contract, so it picks up whatever fee production has configured with no code change.
* **The access-condition ABIs** are unchanged; only `contractAddress` and `chain` differ, and both come from the config block.

What doesn't follow automatically, and is on you:

* **Real funds.** Minting on `base` spends real ETH. Test on staging first.
* **Introspection is off in production** and query depth is capped at 10. Generate types against staging — see [Getting the schema](#getting-the-schema).
* **`SERVICE_NAME`** should identify the real integration; it is echoed into the sign-in message and stored against the issued token.
* Full deployment list, including every other OCL contract on both chains: [Contracts reference](../../references/contracts/README.md).

***

## Getting the schema

**Staging has GraphQL introspection enabled** — point codegen, a playground or an SDK generator straight at it:

```bash
# graphql-codegen, apollo, gql.tada… all work against staging
DESCI_API_SCHEMA=https://staging.graphql.api.molecule.xyz/graphql npx graphql-codegen
```

Introspection requires only the `Authorization` consumer-credential header, the same as any query.

**Production has introspection disabled**, deliberately — `__schema` and `__type` return a validation error there (`__typename` still resolves). Generate against staging and point the generated client at production; the two environments serve the same schema.

Production also enforces a query-depth limit of 10, which fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data — a plain GraphQL error, not the catalogued shape. Handle both.

***

## Where to go next

| Next | Page |
| ---- | ---- |
| What every term in these guides means | [Glossary](../../references/glossary.md) |
| The config and helpers every tutorial uses | [Shared Setup](shared-setup.md) |
| Every operation, parameter and error code | [Labs API](../labs-api/README.md) |
| What each error code means and how to read it | [Error handling](../labs-api/README.md#error-handling) |
| Paying per call, and the gateway base URLs | [x402 Gateway](../x402-gateway.md) |
| What a Lab actually is, onchain | [Molecule Labs](../../technical-deep-dive/onchain-lab.md) |
