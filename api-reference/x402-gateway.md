---
description: >-
  Pay-per-call HTTP 402 gateway that fronts write mutations on the Labs API
  with per-request stablecoin settlement on Base.
icon: coins
---

# 💳 x402 Gateway

## Overview

The x402 gateway wraps a small set of Labs API write mutations with the [HTTP 402 Payment Required](https://www.x402.org/) protocol. Callers — typically autonomous AI agents or external services that don't hold a long-lived service token — settle a USDC payment on Base per request, and the gateway forwards the underlying GraphQL mutation to AppSync.

Each successful request costs the configured mutation price in USDC, which is debited directly from the payer wallet via EIP-3009 `transferWithAuthorization` (or Permit2) and settled through the Coinbase facilitator.

### When to use x402

Use the gateway when:

- **An agent needs write access without provisioning a per-agent service token.** The gateway mints a short-lived, scoped service token on the fly after payment is verified.
- **You want pay-per-call economics.** Each mutation has its own price; no subscription or prepaid balance.
- **You're building third-party tooling that pays for users.** The payer wallet is recorded as the mutation author.

Use the standard [Labs API](labs-api/README.md) with a service token when you have long-lived credentials for a known lab.

---

## Gateway base URLs

| Environment | Base URL | Network | Asset |
| ----------- | -------- | ------- | ----- |
| **Staging** | `https://0go1j7o645.execute-api.eu-central-2.amazonaws.com/prod` | Base Sepolia (`eip155:84532`) | [USDC `0x036CbD…dCF7e`](https://sepolia.basescan.org/address/0x036CbD53842c5426634e7929541eC2318f3dCF7e) |
| **Production** | `https://0qb5gyw72f.execute-api.eu-central-2.amazonaws.com/prod` | Base (`eip155:8453`) | [USDC `0x833589…02913`](https://basescan.org/address/0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913) |

Endpoints are `POST {base}/x402/labs/{mutation}`. Note the `/prod` stage segment on the base URL: it is part of the path you call. (The `resource.url` echoed back inside the `402` challenge omits it — call the URL you built, not the one in the challenge.)

Testnet USDC for the staging gateway comes from the [Circle faucet](https://faucet.circle.com/) — select **Base Sepolia**. You also need a little Base Sepolia ETH for gas if you are doing anything onchain alongside; the payment itself is signed, not sent, so it costs the payer no gas.

Both base URLs are also what the [Molecule Skill](../ai-tooling/molecule-skill.md) plugin expects in `X402_GATEWAY_URL`.

---

## Endpoints

The gateway exposes one HTTP endpoint per allow-listed mutation. All endpoints accept `POST` with a JSON body containing a GraphQL mutation.

```
POST {base}/x402/labs/{mutation}
```

| Path                                         | Wraps mutation                 | Purpose                                                  |
| -------------------------------------------- | ------------------------------ | -------------------------------------------------------- |
| `/x402/labs/initiateCreateOrUpdateFile`      | `initiateCreateOrUpdateFile`   | Start a file upload; returns a presigned URL             |
| `/x402/labs/finishCreateOrUpdateFile`        | `finishCreateOrUpdateFile`     | Finalise a file upload with metadata                     |
| `/x402/labs/createLab`                       | `createLab`                    | Create a lab (data room) for an onchain lab (OCL)       |
| `/x402/labs/generateDataEncryptionKey`       | `generateDataEncryptionKey`    | Generate a data encryption key (DEK) for encrypted uploads |
| `/x402/labs/decryptDataKey`                  | `decryptDataKey`               | Decrypt a file's data key for an authorized caller       |

The path mutation must match the top-level GraphQL mutation field in the request body, otherwise the gateway returns `400`. The allow-list above is the single source of truth in `lambda/x402-gateway-lambda/mutations.ts` (`X402_WRITE_MUTATIONS`).

---

## Payment Flow

The gateway implements the standard x402 three-phase flow: **verify → serve → settle**.

```
┌──────────┐                          ┌────────────┐                    ┌──────────────┐
│  Client  │                          │  Gateway   │                    │ Facilitator  │
│ (agent)  │                          │ (Lambda)   │                    │ (Coinbase)   │
└────┬─────┘                          └─────┬──────┘                    └──────┬───────┘
     │ 1. POST /x402/labs/{mutation}         │                                 │
     │    (no payment header)                │                                 │
     ├──────────────────────────────────────▶│                                 │
     │                                       │                                 │
     │ 2. 402 Payment Required               │                                 │
     │    + payment requirements             │                                 │
     │◀──────────────────────────────────────┤                                 │
     │                                       │                                 │
     │ 3. Sign EIP-3009/Permit2 auth         │                                 │
     │                                       │                                 │
     │ 4. POST again w/ Payment-Signature    │                                 │
     ├──────────────────────────────────────▶│ 5. verify                       │
     │                                       ├────────────────────────────────▶│
     │                                       │◀────────────── verified ────────┤
     │                                       │                                 │
     │                                       │ 6. mint scoped service token    │
     │                                       │ 7. forward GraphQL to AppSync   │
     │                                       │                                 │
     │                                       │ 8. settle                       │
     │                                       ├────────────────────────────────▶│
     │                                       │◀─────────── settled ────────────┤
     │ 9. 200 OK + mutation result           │                                 │
     │◀──────────────────────────────────────┤                                 │
```

1. **402 challenge** — The gateway responds `402` with the x402 payment requirements — network, asset, amount, and `payTo` address — as **base64-encoded JSON in the `payment-required` response header**. The response *body* is only `{"isSuccess":false,"message":"Payment required"}`; the requirements are not in it. See [Reading the 402 challenge](#reading-the-402-challenge).
2. **Sign** — The client signs an EIP-3009 `transferWithAuthorization` (or Permit2) for the quoted amount to the challenge's `payTo` on the quoted network.
3. **Retry with payment** — The signed authorization is submitted as a base64 JSON header under any of `Payment-Signature`, `X-Payment`, or `Payment`.
4. **Verify** — The gateway calls the Coinbase facilitator's `/verify` endpoint. On failure it returns `402` with the original requirements.
5. **Serve** — The gateway mints a scoped, short-lived JWT service token (`allowedMutations: [mutation]`, `authMethod: "x402"`, `ttl = X402_TOKEN_TTL_SECONDS`, default `300s`) with the payer wallet as `adminAddress`, then forwards the GraphQL mutation to AppSync using that token.
6. **Settle** — After a `2xx` upstream response, the gateway calls the facilitator's `/settle` endpoint to broadcast the transfer. Settlement headers are merged into the final response.

### Payer address resolution

The mutation is executed as if the **payer wallet** called it. The payer address is resolved from the verified payment payload, in this order:

1. `payload.authorization.from` (EIP-3009)
2. `payload.permit2Authorization.from` (Permit2)
3. `payerAddress` field on the JSON request body (fallback only — used when upstream wrappers strip the verified payload)

If none resolves to a valid address the gateway returns `400`. Source: `lambda/x402-gateway-lambda/index.ts` (`extractPayerAddress`).

---

## Reading the 402 challenge

The price is **quoted per request** — always read it from the challenge rather than hardcoding an amount.

**Step 1 — call without a payment header:**

```bash
curl -i -X POST \
  https://0go1j7o645.execute-api.eu-central-2.amazonaws.com/prod/x402/labs/createLab \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "mutation CreateLab($oclId: String!) { createLab(input: { oclId: $oclId }) { message error { code message requestId retryable details } } }",
    "variables": { "oclId": "0x0101…" }
  }'
```

**Step 2 — read the `payment-required` header, not the body:**

```http
HTTP/2 402
content-type: application/json
payment-required: eyJ4NDAyVmVyc2lvbiI6MiwiZXJyb3IiOiJQYXltZW50IHJlcXVpcmVkIiw…

{"isSuccess":false,"message":"Payment required"}
```

Base64-decode the header:

```bash
curl -sD - -o /dev/null -X POST "$URL" -H 'Content-Type: application/json' -d "$BODY" \
  | grep -i '^payment-required:' | cut -d' ' -f2- | tr -d '\r' | base64 -d | jq
```

```json
{
  "x402Version": 2,
  "error": "Payment required",
  "resource": {
    "url": "https://0go1j7o645.execute-api.eu-central-2.amazonaws.com/x402/labs/createLab",
    "description": "x402 payment for createLab",
    "mimeType": ""
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0xb016Eb733479874b67c6BeC2470e86a64b33AD76",
      "maxTimeoutSeconds": 300,
      "extra": { "name": "USDC", "version": "2" }
    }
  ]
}
```

`amount` is in the asset's smallest unit — USDC has 6 decimals, so `"10000"` is **$0.01**. That is the price on both staging and production today, for every allow-listed mutation. `extra.name` / `extra.version` are the EIP-712 domain values for the token's `transferWithAuthorization`.

**Step 3 — sign the authorization and retry.** Build an EIP-3009 `transferWithAuthorization` for `amount` to `payTo` on `network`, wrap it in the x402 payment payload, base64 it, and re-send the *same* request with the header:

```javascript
async function callX402(base, mutation, query, variables, wallet) {
  const url = `${base}/x402/labs/${mutation}`;
  const body = JSON.stringify({ query, variables });
  const init = { method: "POST", headers: { "content-type": "application/json" }, body };

  // 1. Challenge
  const challenge = await fetch(url, init);
  if (challenge.status !== 402) return challenge.json(); // already paid / different error

  const raw = challenge.headers.get("payment-required");
  if (!raw) throw new Error("402 without a payment-required header");
  const requirements = JSON.parse(Buffer.from(raw, "base64").toString("utf8"));
  const accepts = requirements.accepts[0];

  // 2. Sign for EXACTLY the quoted amount, asset, network and payTo.
  //    signX402Payment depends on your stack (viem, ethers, CDP SDK).
  const paymentHeader = await signX402Payment(accepts, wallet);

  // 3. Retry the same request with the payment attached
  const result = await fetch(url, {
    ...init,
    headers: { ...init.headers, "payment-signature": paymentHeader },
  });
  return result.json();
}
```

The signed payload is accepted under any of `Payment-Signature`, `X-Payment`, or `Payment`.

**Step 4 — read the result as an ordinary GraphQL response.** A `200` body is the AppSync response verbatim; success is `error == null`, exactly as on the Labs API.

{% hint style="warning" %}
**Authorization is still checked after payment, and settlement does not wait for the mutation to succeed.** Payment buys a short-lived service token for the payer wallet; it does not grant the payer a role. A `createLab` for an OCL the payer does not own, or a write into a lab the payer has no Contributor role on, comes back `200` with `error.code: "UNAUTHORIZED"` — and because settlement is triggered by the upstream `2xx`, you have paid for it. Validate the target lab and your role on it (`listLabMembers`) **before** signing.
{% endhint %}

---

## Request Format

```http
POST /x402/labs/createLab HTTP/1.1
content-type: application/json
payment-signature: <base64 x402 payment payload>

{
  "query": "mutation CreateLab($oclId: String!) { createLab(input: { oclId: $oclId }) { message error { code message requestId retryable details } } }",
  "variables": {
    "oclId": "0x0101...abcd"
  },
  "operationName": "CreateLab"
}
```

The `200` body is the mutation's GraphQL response verbatim, so read it exactly as on the Labs API: the mutation succeeded when `error` is `null`; otherwise branch on `error.code` — see [Error Handling](labs-api/README.md#error-handling). Note that settlement is triggered by the upstream `2xx`, not by mutation success — a `200` whose body carries a non-null `error` (e.g. `VALIDATION_FAILED`, `UNAUTHORIZED`) is still settled, so you pay for a mutation that failed in-band; only an upstream `4xx`/`5xx` skips settlement. Validate inputs (ids, categories/tags, role) before paying.

Constraints enforced by the gateway (`validateMutationQuery`):

- Exactly one GraphQL operation.
- Operation kind `mutation`.
- Exactly one top-level selection.
- Top-level field name must equal the path `{mutation}`.

---

## Pricing & Configuration

Pricing is environment-driven and resolved per-mutation. The gateway evaluates the following env vars in order and uses the first non-empty value:

```
X402_PRICE_<SNAKE_CASE_MUTATION>     e.g. X402_PRICE_CREATE_LAB
X402_PRICE_<UPPER_MUTATION>          e.g. X402_PRICE_CREATELAB
X402_PRICE_DEFAULT                   fallback when no per-mutation price is set
```

Values are interpreted as USDC amounts (e.g. `"2.50"` = $2.50). Prices are environment-configured — the authoritative price for a given mutation is the one quoted in the `402` challenge, so clients should always [read it from the challenge](#reading-the-402-challenge) rather than hardcoding amounts. As of 2026-08-27 every allow-listed mutation quotes **$0.01** on both staging and production.

| Variable                         | Default                                                 | Purpose                                                  |
| -------------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| `X402_NETWORK`                   | `base` (prod) / `base-sepolia` (non-prod)               | CAIP-2 network (`base` → `eip155:8453`)                  |
| `X402_ASSET`                     | Base USDC (prod) / Base Sepolia USDC (non-prod)         | Settlement asset — see [base URLs](#gateway-base-urls)   |
| `X402_PAY_TO_ADDRESS`            | `0xb016Eb733479874b67c6BeC2470e86a64b33AD76`            | Wallet that receives settlement (echoed as `payTo`)      |
| `X402_FACILITATOR_URL`           | `https://api.cdp.coinbase.com/platform/v2/x402`         | Facilitator base URL                                     |
| `X402_PRICE_*` / `X402_PRICE_DEFAULT` | `0.01`                                             | Per-mutation price in USDC (quoted in the 402 challenge) |
| `X402_TOKEN_TTL_SECONDS`         | `300`                                                   | Lifetime of the minted service token                     |
| `X402_MAX_TIMEOUT_SECONDS`       | `60`                                                    | Upstream request budget                                  |

Facilitator authentication uses Coinbase CDP API keys (`CDP_API_KEY_ID_SECRET_ARN` / `CDP_API_KEY_SECRET_SECRET_ARN` in Secrets Manager, or `CDP_API_KEY_ID` / `CDP_API_KEY_SECRET` in local mode) to sign the `/verify`, `/settle`, and `/supported` requests.

---

## Response Semantics

| Status | Meaning                                                                                  |
| ------ | ---------------------------------------------------------------------------------------- |
| `200`  | Payment verified, upstream AppSync returned `2xx`. Body is the AppSync response verbatim; settlement headers are merged in. |
| `402`  | Payment required or payment verification failed. Body includes facilitator hints in headers. |
| `400`  | Path mutation mismatch, missing `query`, invalid GraphQL, or unresolvable payer address. |
| `4xx/5xx` | Upstream AppSync error — settlement is skipped and the upstream response is returned as-is. |

### Idempotency

Each minted service token has a unique `jti` claim, so requests are not idempotent by default — replaying the same signed payment may be rejected by the facilitator's replay protection, and the downstream AppSync call may succeed twice if you retry after a settlement failure. Agents should treat settlement failures as "payment not charged yet" and re-sign.

---

## Agent Usage Pattern

The runnable helper is in [Reading the 402 challenge](#reading-the-402-challenge) above — the one detail agents get wrong is reading the requirements from the response *body* instead of the `payment-required` *header*.

Beyond that, three habits:

* **Read the price every time.** It is quoted per request and per mutation; nothing guarantees it stays at $0.01.
* **Check authorization before paying.** Confirm the lab exists and the payer wallet holds the role the mutation needs (`labWithDataRoomAndFiles`, `listLabMembers` — both public and free) before signing anything. Payment does not grant a role.
* **Treat a settlement failure as "not charged yet."** Re-sign rather than assuming the transfer went through — see [Idempotency](#idempotency).

If you would rather not implement the handshake at all, the [Molecule Skill](../ai-tooling/molecule-skill.md) plugin's `x402_pay` tool does the whole flow in one call. See the [Developers / AI Agents guide](../user-guides/developers-ai-agents.md) for broader integration patterns, and the [Labs API reference](labs-api/README.md) for the full GraphQL signatures of each gated mutation.

---

## Related

- [Getting Started](getting-started/README.md) — which lane to pick, prerequisites, costs
- [Molecule Skill](../ai-tooling/molecule-skill.md) — `x402_pay` does this handshake in one tool call
- [Labs API](labs-api/README.md) — full mutation signatures and variable types
- [Developers / AI Agents](../user-guides/developers-ai-agents.md) — agent integration guide
- [x402 specification](https://www.x402.org/)
