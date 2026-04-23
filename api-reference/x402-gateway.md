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

Use the standard [Labs API](labs-api.md) with a service token when you have long-lived credentials for a known lab.

---

## Endpoints

The gateway exposes one HTTP endpoint per allow-listed mutation. All endpoints accept `POST` with a JSON body containing a GraphQL mutation.

```
POST /x402/labs/{mutation}
```

| Path                                         | Wraps mutation                 | Purpose                                                  |
| -------------------------------------------- | ------------------------------ | -------------------------------------------------------- |
| `/x402/labs/initiateCreateOrUpdateFileV2`    | `initiateCreateOrUpdateFileV2` | Start a file upload; returns a presigned URL (+ DEK if encryption requested) |
| `/x402/labs/finishCreateOrUpdateFileV2`      | `finishCreateOrUpdateFileV2`   | Finalise a file upload with metadata                     |
| `/x402/labs/createAnnouncementV2`            | `createAnnouncementV2`         | Publish a project announcement                           |
| `/x402/labs/createProject`                   | `createProject`                | Create a project / data room bound to an IP-NFT          |
| `/x402/labs/addProjectOwner`                 | `addProjectOwner`              | Add a wallet as an owner of a project                    |

The path mutation must match the top-level GraphQL mutation field in the request body, otherwise the gateway returns `400`.

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

1. **402 challenge** — The gateway returns an x402-standard payment-requirements response describing network, asset, price, and payTo address.
2. **Sign** — The client signs an EIP-3009 `transferWithAuthorization` (or Permit2) for the quoted amount to `X402_PAY_TO_ADDRESS` on the configured network.
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

## Request Format

```http
POST /x402/labs/createAnnouncementV2 HTTP/1.1
content-type: application/json
payment-signature: <base64 x402 payment payload>

{
  "query": "mutation CreateAnnouncement($ipnftUid: String!, $headline: String!, $body: String!) { createAnnouncementV2(ipnftUid: $ipnftUid, headline: $headline, body: $body) { isSuccess message error { message } } }",
  "variables": {
    "ipnftUid": "0xcaD8...Fc1_42",
    "headline": "Milestone 1 complete",
    "body": "..."
  },
  "operationName": "CreateAnnouncement"
}
```

Constraints enforced by the gateway (`validateMutationQuery`):

- Exactly one GraphQL operation.
- Operation kind `mutation`.
- Exactly one top-level selection.
- Top-level field name must equal the path `{mutation}`.

---

## Pricing & Configuration

Pricing is environment-driven and resolved per-mutation. The gateway evaluates the following env vars in order and uses the first non-empty value:

```
X402_PRICE_<SNAKE_CASE_MUTATION>     e.g. X402_PRICE_CREATE_ANNOUNCEMENT_V2
X402_PRICE_<UPPER_MUTATION>          e.g. X402_PRICE_CREATEANNOUNCEMENTV2
X402_PRICE_DEFAULT                   fallback, default "1.00"
```

Values are interpreted as USDC amounts (e.g. `"2.50"` = $2.50).

| Variable                         | Default                                                 | Purpose                                                  |
| -------------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| `X402_NETWORK`                   | `base` (prod) / `base-sepolia` (non-prod)               | CAIP-2 network (`base` → `eip155:8453`)                  |
| `X402_PAY_TO_ADDRESS`            | —                                                       | Wallet that receives settlement                          |
| `X402_FACILITATOR_URL`           | `https://api.cdp.coinbase.com/platform/v2/x402`         | Facilitator base URL                                     |
| `X402_PRICE_*` / `X402_PRICE_DEFAULT` | `"1.00"`                                           | Per-mutation price in USDC                               |
| `X402_TOKEN_TTL_SECONDS`         | `300`                                                   | Lifetime of the minted service token                     |

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

An autonomous agent typically wraps each gateway call in a helper:

```ts
// Pseudocode — actual wallet signing depends on your stack (viem, ethers, CDP SDK).
async function callX402(mutation: string, query: string, variables: any) {
  const url = `${GATEWAY_BASE}/x402/labs/${mutation}`;

  // 1. 402 challenge
  const challenge = await fetch(url, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ query, variables }),
  });

  const paymentRequirements = await challenge.json();
  const paymentHeader = await signX402Payment(paymentRequirements, agentWallet);

  // 2. Retry with payment
  const result = await fetch(url, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "payment-signature": paymentHeader,
    },
    body: JSON.stringify({ query, variables }),
  });

  return result.json();
}
```

See the [Developers / AI Agents guide](../user-guides/developers-ai-agents.md) for end-to-end agent integration patterns, and the [Labs API reference](labs-api.md) for the full GraphQL signatures of each gated mutation.

---

## Related

- [Labs API](labs-api.md) — full mutation signatures and variable types
- [Developers / AI Agents](../user-guides/developers-ai-agents.md) — agent integration guide
- [x402 specification](https://www.x402.org/)
