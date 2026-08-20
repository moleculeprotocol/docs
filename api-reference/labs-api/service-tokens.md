# Service Token Management

## Obtaining Tokens

Service tokens must be requested from the Molecule team (see [Authentication](../authentication.md) section above).

Alternatively, a service can obtain a token **self-service** by proving control of its wallet — useful for autonomous agents, bots, and CI/CD pipelines that don't have a browser-based Privy session. This is a three-step flow: fetch a sign-in message carrying a single-use nonce, sign it with the service wallet, then exchange the signature (and nonce) for a token.

> **Breaking change.** The sign-in message is now **EIP-712 typed data**, not a plain string, and must be signed with `eth_signTypedData_v4` — signatures over the old `personal_sign` message are no longer accepted. Each message also embeds a server-issued, single-use `nonce` that expires after about 10 minutes; a captured signature can mint exactly one token, and only within that window.

**Step 1 — Get the sign-in message (`getServiceSignInMessage`):**

```graphql
query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
  getServiceSignInMessage(
    walletAddress: $walletAddress
    serviceName: $serviceName
  ) {
    message
    nonce
    issuedAt
    expiresAt
  }
}
```

| Parameter     | Type   | Required | Description                                         |
| ------------- | ------ | -------- | --------------------------------------------------- |
| walletAddress | String | Yes      | Wallet address of the service (e.g. an agent's EOA) |
| serviceName   | String | Yes      | Name of the service requesting a token              |

Public query — no authentication required.

| Field      | Type         | Description                                                                                                       |
| ---------- | ------------ | ------------------------------------------------------------------------------------------------------------------ |
| message    | String       | JSON-serialized EIP-712 typed data (`{domain, types, primaryType, message}`) — sign this whole string, don't hand-edit it |
| nonce      | String       | Single-use nonce embedded in `message`; echo it into `generateServiceToken`                                        |
| issuedAt   | AWSTimestamp | Unix seconds the message was issued at                                                                             |
| expiresAt  | AWSTimestamp | Unix seconds the nonce (and any signature over it) expires at — sign and redeem before this                       |

**Step 2 — Sign the typed data:**

Sign `message` with `eth_signTypedData_v4`, then submit the signature and nonce:

```ts
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const account = privateKeyToAccount(SERVICE_WALLET_PRIVATE_KEY);
const client = createWalletClient({ account, transport: http(RPC_URL) });

const typedData = JSON.parse(message); // `message` from Step 1
const messageSignature = await client.signTypedData({ account, ...typedData });
```

**Step 3 — Exchange the signature for a token (`generateServiceToken`):**

```graphql
mutation GenerateServiceToken(
  $serviceName: String!
  $walletAddress: String!
  $messageSignature: String!
  $nonce: String!
  $expiresIn: String
) {
  generateServiceToken(
    serviceName: $serviceName
    walletAddress: $walletAddress
    messageSignature: $messageSignature
    nonce: $nonce
    expiresIn: $expiresIn
  ) {
    token
    tokenId
    serviceName
    expiresAt
    createdAt
    isSuccess
    message
  }
}
```

| Parameter        | Type   | Required | Description                                                                                                                                    |
| ---------------- | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| serviceName      | String | Yes      | Name of the service the token is issued for                                                                                                      |
| walletAddress    | String | No\*     | Service wallet address (required together with `messageSignature` and `nonce`)                                                                  |
| messageSignature | String | No\*     | Hex-encoded `eth_signTypedData_v4` signature over the typed data from Step 1                                                                     |
| nonce            | String | No\*     | The nonce from Step 1; consumed atomically on a successful mint — reusing it fails with `UNAUTHENTICATED` / `details.reason: "INVALID_NONCE"`     |
| expiresIn        | String | No       | Token lifetime (e.g. `"30d"`, `"720h"`), clamped to **\[1 hour, 2 years]** — out of range or malformed fails `VALIDATION_FAILED`. Defaults to 180 days |

\* `walletAddress`, `messageSignature`, and `nonce` must all be provided together for signature-based issuance. The returned `token` is the JWT to pass as `X-Service-Token` on subsequent requests. The same `expiresIn` clamp applies to `extendServiceToken` below.

## Extending Token Expiration

You can extend your service token's expiration using the `extendServiceToken` mutation:

```graphql
mutation ExtendServiceToken($tokenId: String!, $expiresIn: String!) {
  extendServiceToken(tokenId: $tokenId, expiresIn: $expiresIn) {
    token
    tokenId
    expiresAt
    isSuccess
    message
  }
}
```

**Parameters:**

| Parameter | Type   | Description                                     |
| --------- | ------ | ----------------------------------------------- |
| tokenId   | String | Token ID provided when token was generated      |
| expiresIn | String | New duration (e.g., `"30d"`, `"720h"`, `"90d"`), clamped to **\[1 hour, 2 years]** |

**Example:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation ExtendServiceToken($tokenId: String!, $expiresIn: String!) { extendServiceToken(tokenId: $tokenId, expiresIn: $expiresIn) { token tokenId expiresAt isSuccess message } }",
    "variables": {
      "tokenId": "your-token-id",
      "expiresIn": "90d"
    }
  }'
```

**Important:** Extension returns a **new JWT token** - update your stored token accordingly.

## Revoking Tokens

Revoke a service token immediately (e.g., if compromised):

```graphql
mutation RevokeServiceToken($tokenId: String!) {
  revokeServiceToken(tokenId: $tokenId) {
    tokenId
    isSuccess
    message
    revokedAt
  }
}
```

**Example:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation RevokeServiceToken($tokenId: String!) { revokeServiceToken(tokenId: $tokenId) { tokenId isSuccess message revokedAt } }",
    "variables": {
      "tokenId": "your-token-id"
    }
  }'
```

---

