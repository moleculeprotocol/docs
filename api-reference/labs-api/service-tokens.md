# Service Token Management

A service token is the credential that proves *which wallet* a write request acts as. It is **self-issued** — you mint your own by proving control of the wallet, and nobody has to provision one for you.

> **Wallet-bound, not lab-bound.** A service token carries a wallet identity, not a list of labs. What it may do on a given lab is resolved per request from that wallet's live onchain role, so one token works across every lab the wallet has a role on, and a role granted after issuance takes effect without re-issuing. See [What a Service Token actually authorizes](../authentication.md#what-a-service-token-actually-authorizes).

## Obtaining a Token

Two calls, no human in the loop — the path for autonomous agents, bots and CI/CD pipelines that have no browser-based Privy session. Fetch the deterministic sign-in message, sign it with the service wallet, then exchange the signature for a token. For the runnable version, see [Tutorial 1 Step 1](../getting-started/tutorial-1-public-upload.md#step-1-get-a-service-token).

Issuance is **not** gated on holding a role on any lab: any wallet can mint a token for itself. The role is what makes the token useful.

**Step 1 — Get the sign-in message (`getServiceSignInMessage`):**

```graphql
query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
  getServiceSignInMessage(
    walletAddress: $walletAddress
    serviceName: $serviceName
  ) {
    message
  }
}
```

| Parameter     | Type   | Required | Description                                         |
| ------------- | ------ | -------- | --------------------------------------------------- |
| walletAddress | String | Yes      | Wallet address of the service (e.g. an agent's EOA) |
| serviceName   | String | Yes      | Name of the service requesting a token              |

Public query — no authentication required.

**Step 2 — Exchange the signature for a token (`generateServiceToken`):**

Sign the returned `message` **verbatim** with the service wallet, as a plain personal message (EIP-191 `personal_sign` — **not** typed data). The backend recomposes and verifies the same string server-side, so re-wording or re-formatting it fails with `UNAUTHENTICATED` / `reason: INVALID_SIGNATURE`. Then submit the signature:

```graphql
mutation GenerateServiceToken(
  $serviceName: String!
  $walletAddress: String!
  $messageSignature: String!
  $expiresIn: String
) {
  generateServiceToken(
    serviceName: $serviceName
    walletAddress: $walletAddress
    messageSignature: $messageSignature
    expiresIn: $expiresIn
  ) {
    token
    tokenId
    serviceName
    expiresAt
    createdAt
    message
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

| Parameter        | Type   | Required | Description                                                                  |
| ---------------- | ------ | -------- | ---------------------------------------------------------------------------- |
| serviceName      | String | Yes      | Name of the service the token is issued for                                  |
| walletAddress    | String | No\*     | Service wallet address (required together with `messageSignature`)           |
| messageSignature | String | No\*     | Hex-encoded signature of the sign-in message (required with `walletAddress`) |
| expiresIn        | String | No       | Token lifetime — defaults to `180d`. See the bounds below                     |

\* `walletAddress` and `messageSignature` must be provided together for signature-based issuance. The returned `token` is the JWT to pass as `X-Service-Token` on subsequent requests.

**`expiresIn`:**

| | |
| --- | --- |
| Default when omitted | `180d` |
| Format | `<integer><unit>`, unit one of `s` `m` `h` `d` `w` `M` `y` — e.g. `"30d"`, `"720h"`, `"6M"`, `"1y"` |
| Minimum | 1 hour |
| Maximum | 2 years |

`M` is a 30-day month and `y` is a 365-day year. Anything outside the bounds, or in another format, is rejected with `VALIDATION_FAILED`. Prefer a short lifetime matched to the caller's purpose — for an agent, match the expiry of its role grant — over the 180-day default.

Success ⇔ `error == null`. On failure `error` carries the catalogue `code` (e.g. `UNAUTHENTICATED` when the signature does not verify), `message` mirrors `error.message`, and `token`, `tokenId`, `expiresAt` and `createdAt` are `null` (`serviceName` may echo the name you sent) — guard for `null`, not for empty strings, and branch on `error`, never on the token fields.

## Extending Token Expiration

You can extend your service token's expiration using the `extendServiceToken` mutation. **Scoped to your own tokens:** the token you present must own the `tokenId` you name. A `tokenId` belonging to another wallet returns the same `NOT_FOUND` as one that does not exist, so token existence cannot be probed.

```graphql
mutation ExtendServiceToken($tokenId: String!, $expiresIn: String!) {
  extendServiceToken(tokenId: $tokenId, expiresIn: $expiresIn) {
    token
    tokenId
    expiresAt
    message
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

**Parameters:**

| Parameter | Type   | Description                                     |
| --------- | ------ | ----------------------------------------------- |
| tokenId   | String | Token ID provided when token was generated      |
| expiresIn | String | New duration (e.g., `"30d"`, `"720h"`, `"90d"`) |

**Example:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation ExtendServiceToken($tokenId: String!, $expiresIn: String!) { extendServiceToken(tokenId: $tokenId, expiresIn: $expiresIn) { token tokenId expiresAt message error { code message requestId retryable details } } }",
    "variables": {
      "tokenId": "your-token-id",
      "expiresIn": "90d"
    }
  }'
```

**Important:** Extension returns a **new JWT token** - update your stored token accordingly. On failure `error` is set and `token` and `expiresAt` are `null`; `tokenId` may echo the id you sent, so branch on `error`, not on the token fields.

## Revoking Tokens

Revoke a service token immediately (e.g., if compromised). Like `extendServiceToken`, this is **scoped to your own tokens** — the presented token must own the `tokenId`, and a foreign one returns `NOT_FOUND`.

```graphql
mutation RevokeServiceToken($tokenId: String!) {
  revokeServiceToken(tokenId: $tokenId) {
    tokenId
    message
    revokedAt
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

Success ⇔ `error == null`. On failure `message` mirrors `error.message`; do not infer the outcome from `tokenId` or `revokedAt` — `tokenId` may echo the id you sent, and when the token was already revoked (`CONFLICT`, reason `ALREADY_REVOKED`) `revokedAt` carries the original revocation time.

**Example:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: YOUR_CONSUMER_CREDENTIAL' \
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation RevokeServiceToken($tokenId: String!) { revokeServiceToken(tokenId: $tokenId) { tokenId message revokedAt error { code message requestId retryable details } } }",
    "variables": {
      "tokenId": "your-token-id"
    }
  }'
```

---

