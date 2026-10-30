---
description: >-
  The production release on Base Sepolia, with its own data and testnet funds —
  where every tutorial in these docs runs.
icon: flask
---

# Sandbox

The sandbox is where you build and test against the Molecule API without spending real money. It runs **the same release as production**, with production's configuration and limits, on **Base Sepolia** with its own contracts and its own data. Every [Getting Started](getting-started/README.md) tutorial runs against it.

Because it mirrors production, code that works on the sandbox works on production once you swap a handful of constants — see [Running in Production](getting-started/README.md#running-in-production).

***

## Endpoints

| What | Sandbox | Production |
| ---- | ------- | ---------- |
| GraphQL API | `https://sandbox.graphql.api.molecule.xyz/graphql` | `https://production.graphql.api.molecule.xyz/graphql` |
| Labs app | [sandbox.app.molecule.xyz](https://sandbox.app.molecule.xyz) | [app.molecule.xyz](https://app.molecule.xyz) |
| [x402 gateway](x402-gateway.md#gateway-base-urls) | `https://cjwbtwwqz2.execute-api.eu-central-2.amazonaws.com/prod` | `https://0qb5gyw72f.execute-api.eu-central-2.amazonaws.com/prod` |
| Agent Connect MCP server | `https://sandbox.mcp.molecule.xyz` | `https://mcp.molecule.xyz` |
| Agent Connect authorization server | `https://sandbox.agent-connect.molecule.xyz` | `https://agent-connect.molecule.xyz` |
| Chain | Base Sepolia (`84532`) | Base (`8453`) |

The sandbox contract set — factory, LabNFT, AccessResolver, tokenizer and the rest — is on [Supported Networks & Contracts](../references/contracts/README.md).

**Agent Connect** is the MCP server an AI host (Claude, ChatGPT, Codex, Cursor) connects to in order to act in a lab as an agent its owner approved. To try it against a sandbox lab, add `https://sandbox.mcp.molecule.xyz` to your host as a remote MCP server; the host is sent to the sandbox authorization server to sign in.

***

## Getting a credential

1. Sign in to the sandbox Labs app at [sandbox.app.molecule.xyz](https://sandbox.app.molecule.xyz).
2. Open the **API credentials** page and click **Create**.
3. Copy the `mol_usr-…` credential straight away — it's shown once.

Credentials are per environment: a sandbox credential doesn't authenticate against production, and a production credential doesn't work here. Everything else about credentials — lifetime, the limit of 5 active ones, revoking, the API terms — is the same as in production and covered in [Authentication](authentication.md#obtaining-api-access).

## Testnet funds

You need funds only for onchain steps, such as minting a LabNFT from code, and for paying through the x402 gateway:

* **Base Sepolia ETH** for gas, from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets).
* **Base Sepolia USDC**, only for the x402 gateway, from the [Circle faucet](https://faucet.circle.com/) (select Base Sepolia).

Creating a lab in the sandbox Labs app needs no funds: Molecule covers those transactions, as it does in production.

***

## Sandbox data is permanent

Treat everything you put in the sandbox as public and permanent:

* **Onchain state stays.** LabNFTs, role grants and every other transaction live on Base Sepolia for good.
* **Data rooms keep every version.** A file you upload is committed to the lab's data room with its history. Deleting it hides it from the API; it does not erase it.
* **Use test data only.** Don't upload real research data, personal data or anything confidential, even encrypted.

The sandbox may be **reset by a contract redeploy**. If that happens, the contract addresses change and labs created on the old contracts don't carry over. A reset is announced in advance in the [API Changelog](changelog.md).

***

## Demo lab

The sandbox has a public demo lab you can query straight away, before you create a lab of your own:

| Field | Value |
| ----- | ----- |
| Name | Schrödinger's Catnip Lab |
| `shortname` | `schrodingers-catnip-lab` |
| Lab page | [sandbox.app.molecule.xyz/labs/schrodingers-catnip-lab](https://sandbox.app.molecule.xyz/labs/schrodingers-catnip-lab) |
| `oclId` | `0x0101000000000000000000000064fc5fc7134a36be03b7bc56a852abf996dc33` |
| `labNftTokenId` | `0` |
| Public file | `/preprints/catnip-superposition-preprint-v1.pdf` |

Read it and its files with your sandbox credential:

```bash
curl -s -X POST https://sandbox.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H "Authorization: $CONSUMER_CREDENTIAL" \
  -d '{
    "query": "query($oclId: String!) { labWithDataRoomAndFiles(oclId: $oclId) { oclId shortname name dataRoom { id files { path contentType accessLevel version } } } }",
    "variables": { "oclId": "0x0101000000000000000000000064fc5fc7134a36be03b7bc56a852abf996dc33" }
  }'
```

***

## Differences from production

Only these:

* **Chain and contracts.** Base Sepolia and the sandbox contract set, instead of Base and the production set.
* **Funds.** Testnet ETH and USDC, so nothing costs real money.
* **Data.** The sandbox has its own labs, files and credentials. Nothing is shared with production in either direction.

The API release, the schema and the limits are the same — introspection is off and query depth is capped at 10 in both. See [Rate & Query Limits](rate-limits.md#sandbox-and-production) and [Getting the schema](getting-started/README.md#getting-the-schema).

{% include "../.gitbook/includes/support.md" %}
