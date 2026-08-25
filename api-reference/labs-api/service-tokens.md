# Service Token Management

## Obtaining Tokens

Service tokens must be requested from the Molecule team (see [Authentication](../authentication.md) section above).

Alternatively, a service can obtain a token **self-service** by proving control of its wallet — useful for autonomous agents, bots, and CI/CD pipelines that don't have a browser-based Privy session. This is a two-step flow: fetch the deterministic sign-in message, sign it with the service wallet, then exchange the signature for a token.

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

Sign the returned `message` with the service wallet, then submit the signature:

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
| expiresIn        | String | No       | Token lifetime (e.g. `"30d"`, `"720h"`)                                      |

\* `walletAddress` and `messageSignature` must be provided together for signature-based issuance. The returned `token` is the JWT to pass as `X-Service-Token` on subsequent requests.

Success ⇔ `error == null`. On failure `error` carries the catalogue `code` (e.g. `UNAUTHENTICATED` when the signature does not verify), `message` mirrors `error.message`, and `token`, `tokenId`, `expiresAt` and `createdAt` are `null` (`serviceName` may echo the name you sent) — guard for `null`, not for empty strings, and branch on `error`, never on the token fields.

## Extending Token Expiration

You can extend your service token's expiration using the `extendServiceToken` mutation:

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

Revoke a service token immediately (e.g., if compromised):

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
  -H 'X-Service-Token: YOUR_CURRENT_TOKEN' \
  -d '{
    "query": "mutation RevokeServiceToken($tokenId: String!) { revokeServiceToken(tokenId: $tokenId) { tokenId message revokedAt error { code message requestId retryable details } } }",
    "variables": {
      "tokenId": "your-token-id"
    }
  }'
```

---

