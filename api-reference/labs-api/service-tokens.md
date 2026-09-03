# Service Token Management

A service token is the credential that proves *which wallet* a write request acts as. It is **self-issued** — you mint your own by proving control of the wallet, and nobody has to provision one for you.

> **Wallet-bound, not lab-bound.** A service token carries a wallet identity, not a list of labs. What it may do on a given lab is resolved per request from that wallet's live onchain role, so one token works across every lab the wallet has a role on, and a role granted after issuance takes effect without re-issuing. See [What a Service Token actually authorizes](../authentication.md#what-a-service-token-actually-authorizes).

## Obtaining a Token

Two calls, no human in the loop — the path for autonomous agents, bots and CI/CD pipelines that have no browser-based Privy session. Fetch a fresh sign-in message, sign it with the service wallet, then exchange the signature for a token. For the runnable version, see [Step 1 of Create a lab and upload a file](../getting-started/create-lab-and-upload-file.md#step-1-get-a-service-token).

Issuance is **not** gated on holding a role on any lab: any wallet can mint a token for itself. The role is what makes the token useful.

**Step 1 — Get the sign-in message (`getServiceSignInMessage`):**

```graphql
query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
  getServiceSignInMessage(
    walletAddress: $walletAddress
    serviceName: $serviceName
  ) {
    message
    expiresAt
  }
}
```

| Parameter     | Type   | Required | Description                                         |
| ------------- | ------ | -------- | --------------------------------------------------- |
| walletAddress | String | Yes      | Wallet address of the service (e.g. an agent's EOA) |
| serviceName   | String | Yes      | Name of the service requesting a token              |

Public query — no authentication required.

| Field | Description |
| ----- | ----------- |
| `message` | The exact string to sign. Contains a server-issued single-use nonce and its expiry, so **it changes on every call** |
| `expiresAt` | ISO-8601 expiry of the embedded nonce. After this, the signature is rejected and a new message must be requested |

{% hint style="warning" %}
**Single-use, and valid for 10 minutes.** The sign-in message is not deterministic — do not cache it, do not cache a signature over it, and never reconstruct the string client-side. Concretely:

* The nonce is **consumed** by the first successful `generateServiceToken`. Issuing a second token means fetching a new message and signing again.
* There is **one outstanding nonce per `(walletAddress, serviceName)`**, last-write-wins: calling this query again invalidates the message you have not yet redeemed.
* The window is **10 minutes** from issuance (`expiresAt`). Sign and redeem promptly rather than fetching a message ahead of time.
* Signatures over the older, nonce-free message format no longer verify.
{% endhint %}

**Step 2 — Exchange the signature for a token (`generateServiceToken`):**

Sign the returned `message` **verbatim** with the service wallet, as a plain personal message (EIP-191 `personal_sign` — **not** typed data). The backend recomposes the same string from the stored nonce record and verifies it server-side, so re-wording or re-formatting it fails with `UNAUTHENTICATED` / `reason: INVALID_SIGNATURE`. Then submit the signature:

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

**Failure modes on the signature path** — all `UNAUTHENTICATED`, distinguished by `details.reason`. Read it through the tolerant [`parseDetails`](README.md#error-handling), not a bare `JSON.parse` — the in-band string is currently doubly encoded:

| `reason` | What happened | Fix |
| -------- | ------------- | --- |
| `NONCE_NOT_FOUND` | No nonce record at all — you never called `getServiceSignInMessage` for this wallet + service, or the nonce was already consumed by an earlier token | Fetch a fresh message and sign it again. Do not retry the same signature |
| `NONCE_EXPIRED` | The message is older than its 10-minute window | Fetch a fresh message and sign it again |
| `INVALID_SIGNATURE` | The signed bytes are not the string the backend recomposes. Either the message was altered (re-formatted, rebuilt client-side, typed-data signing, the retired nonce-free format) **or it was superseded** — a later `getServiceSignInMessage` call replaced the stored nonce, so an earlier message no longer matches | Sign the `message` from the most recent call, byte-for-byte, with `personal_sign` |
| `WALLET_MISMATCH` | `walletAddress` is not the address that produced the signature | Pass the signing address |

None of these are retryable as-is: every one of them means "get a new message and sign that". A `VALIDATION_FAILED` here refers to `walletAddress` format or `expiresIn` bounds instead.

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

