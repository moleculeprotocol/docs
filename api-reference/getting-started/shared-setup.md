---
description: >-
  The config constants and helpers every tutorial opens with — copy this block
  once and each tutorial's snippets run against it.
icon: code
---

# Shared Setup

**Every tutorial in this section starts from the code on this page.** Creating a lab, uploading a file, encrypting and decrypting a file and adding an agent as a collaborator all assume the constants and helper functions below are already defined — their snippets call `graphql()`, `assertOk()` and `withIndexerLagRetry()` without redefining them, and read `GRAPHQL_URL`, `CHAIN`, `FACTORY_ADDRESS` and the rest from here. Copy this block into your script once, then follow whichever tutorial you need.

You do not have to copy it by hand if you only want to run one tutorial end to end: the **complete script** at the bottom of each tutorial page carries all of this inline and runs standalone. This page exists so the shared parts are documented and maintained in one place instead of three.

Everything here targets **staging** (Base Sepolia, testnet funds). To point the same code at mainnet, replace the constants using the table in [Running in Production](README.md#running-in-production) — nothing else changes, because every step reads from these constants.

***

## The shared block

Every environment-specific value lives in this one block, and every helper the tutorials call is defined in it. Paste it at the top of your script.

```javascript
import { baseSepolia } from "viem/chains"; // production: `base`

// ---- Staging (Base Sepolia) config — see "Running in Production" to swap ----
const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const CHAIN = baseSepolia;
const FACTORY_ADDRESS = "0xd629FE2310b4309a212495F10A47f8436dcEfD90"; // OnChainLabFactory
const LABNFT_ADDRESS = "0x13Ff210695fdb54A7F928ECcc28BC3486c05BB28"; // LabNFT (proxy)
const ACCESS_RESOLVER_ADDRESS = "0x5493F472602C87318EA5Eff753cDD593bf9bF559"; // AccessResolver
const ACCESS_CONDITION_CHAIN = "baseSepolia"; // the `chain` string inside accessControlConditions
const LAB_APP_URL = "https://testnet.labs.molecule.xyz"; // production: https://labs.molecule.xyz

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL; // mol_<id>_<secret> — no "Bearer" prefix
const WALLET_PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY;

// Set once Step 1 exchanges a wallet signature for a token. Every call after
// that automatically starts sending it; public queries (like Step 1's own
// sign-in-message lookup) work fine without it.
let serviceToken;

async function graphql(query, variables) {
  // Authorization is always required. X-Service-Token is added once we have
  // one — omit it entirely rather than sending an empty header.
  const headers = { "Content-Type": "application/json", Authorization: CONSUMER_CREDENTIAL };
  if (serviceToken) headers["X-Service-Token"] = serviceToken;

  const res = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });
  const { data, errors } = await res.json();
  // Queries report failure here: a top-level errors[] entry whose errorType is
  // the catalogue code. Mutations report expected failures in-band instead (see
  // assertOk); a top-level entry on a mutation means a transport/infrastructure
  // failure or an invalid request document.
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}

// Mutations report failure in-band: `error` is null on success. Throw on a
// non-null `error` so a failed step stops the workflow with the catalogue
// `code` and the `requestId` to quote in a bug report.
// `error.details` carries the specific cause under `reason`, but it arrives in
// more than one shape: a plain object on thrown query errors, a JSON string
// in-band, and currently a doubly-encoded JSON string in-band. Parse until it
// stops being a string, so one reader handles all three.
function parseDetails(details) {
  let value = details;
  for (let i = 0; i < 3 && typeof value === "string"; i++) {
    try {
      value = JSON.parse(value);
    } catch {
      break;
    }
  }
  return value && typeof value === "object" ? value : {};
}

function assertOk(result, op) {
  if (result.error) {
    const { code, message, requestId } = result.error;
    const { reason } = parseDetails(result.error.details);
    throw new Error(
      `${op} failed: ${code}${reason ? `/${reason}` : ""}: ${message} (requestId ${requestId})`,
    );
  }
  return result;
}

// Onchain state — a mint, a role grant — reaches the API through an event
// indexer, so a write issued immediately after one can fail on state the chain
// already has. Retry with backoff; re-issuing the token never helps.
async function withIndexerLagRetry(
  fn,
  { codes = ["NOT_FOUND"], attempts = 12, baseMs = 2000, capMs = 30000 } = {},
) {
  const laggy = new RegExp(codes.join("|"));
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      if (!laggy.test(String(err)) || i === attempts - 1) throw err;
      const delay = Math.min(baseMs * 2 ** i, capMs); // 2s, 4s, 8s, 16s, then 30s
      console.warn(`indexer not caught up (attempt ${i + 1}/${attempts}); retrying in ${delay / 1000}s`);
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}
```

***

## Where each piece is used

| Piece | What it does | Where it shows up |
| ----- | ------------ | ----------------- |
| The config constants | Every environment-specific value in one place — endpoint, chain, contract addresses, app URL | Every step of every tutorial |
| `graphql(query, variables)` | POSTs to the API with `Authorization` always set and `X-Service-Token` added once Step 1 has issued one | Every GraphQL call |
| `parseDetails(details)` | Reads `error.details` tolerantly — it arrives as an object, a JSON string, or a doubly-encoded JSON string | Inside `assertOk`; also useful when you branch on `details.reason` yourself |
| `assertOk(result, op)` | Turns an in-band mutation `error` into a thrown error carrying the catalogue `code` and the `requestId` | After every mutation |
| `withIndexerLagRetry(fn)` | Retries a call that failed only because onchain state has not been indexed yet | [Step 4 of Create a lab and upload a file](create-lab-and-upload-file.md#step-4-upload-the-file) (after a mint) and [Step 4 of Agent as a lab contributor](agent-as-a-lab-contributor.md#step-4-the-agent-uploads) (after a role grant) |

***

## Then what

| Next | Page |
| ---- | ---- |
| Prerequisites, costs and the ten-minute quickstart | [Getting Started](README.md) |
| Create a lab and upload a public file | [Create a lab and upload a file](create-lab-and-upload-file.md) |
| Upload an encrypted file | [Upload an encrypted file](upload-encrypted-file.md) |
| Give your agent access to a lab | [Agent as a lab contributor](agent-as-a-lab-contributor.md) |
| Run the same code against mainnet | [Running in Production](README.md#running-in-production) |
