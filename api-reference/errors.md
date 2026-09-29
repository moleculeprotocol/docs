---
description: >-
  How the Labs API, the Tokenization API and the x402 Gateway report failure:
  where the error appears, what to branch on, and when to retry.
icon: triangle-exclamation
---

# 🚨 Errors

The Molecule APIs don't share one error shape. The Labs API uses `ApiError`, the Tokenization API keeps its older `isSuccess` + `EvmTokenizationError` envelope, and the x402 Gateway adds an HTTP layer in front of Labs mutations. This page covers all three, plus the rules that apply to every one of them.

Errors that only one operation produces stay on that operation's page, for example the [sign-in failure reasons](labs-api/service-tokens.md) or the [tokenization contract reverts](tokenization-api.md#error-handling). This page links to them.

---

## Which contract applies

| You called | Failure shows up in | Branch on | Retry signal | Correlation id |
| --- | --- | --- | --- | --- |
| A Labs API **query** | top-level `errors[]`; the field (often `data` itself) is `null` | `errors[i].errorType` | `errors[i].errorInfo.retryable` | `errors[i].errorInfo.requestId` |
| A Labs API **mutation** | the result's `error` field (`ApiError`), HTTP `200` | `error.code` | `error.retryable` | `error.requestId` |
| A Tokenization API query or mutation | the result's `isSuccess: false` and `error` field (`EvmTokenizationError`), HTTP `200` | `error.code` | `error.retryable` | none in-band; `errorInfo.requestId` on the [few thrown errors](#tokenization-api-errors) |
| A mutation through the x402 Gateway | the HTTP status first; on `200`, the Labs mutation contract above | HTTP status, then `error.code` | see [x402 Gateway errors](#x402-gateway-errors) | none; on settlement, the transaction hash |

Whichever API you call, the message is for humans and may change without notice. Branch on the code or status, never on the message text.

---

## Failures before your operation runs

Some requests fail before any resolver sees them. They look the same whichever API you called.

**Missing or malformed consumer credential: HTTP `401`.** The `Authorization` header is checked before the GraphQL layer runs, so there's no `errorType` and no `ApiError`. The `mol_…` credential goes in directly, with no `Bearer` prefix. See [Authentication](authentication.md).

**Malformed request body: HTTP `400`.** A body that isn't valid JSON is rejected before it's parsed as GraphQL, and before the credential is checked. The body has one `errors[]` entry with `errorType: "MalformedHttpRequestException"` and no `errorInfo`.

**Blocked at the edge: HTTP `403`.** The API sits behind a web application firewall with request-count rules. A request it blocks gets a raw `403` with a non-GraphQL body: no `errors[]`, no `errorType`, no `errorInfo`. These rules only count requests today and block nothing. If they start blocking, slow down before retrying. This is not the [`RATE_LIMITED`](#rate-limited-requests) error, which is a catalogued GraphQL error.

**Invalid GraphQL document: a plain GraphQL error with no code.** A query that selects a field that doesn't exist, or passes a variable of the wrong type, returns a top-level `errors[]` entry. It has a `message` but no catalogued `errorType` and no `errorInfo`. Fix the document; retrying it unchanged won't help. On a mutation, an `errors[]` entry like this means the document failed validation and the mutation didn't run.

**Query too deep: `QueryDepthLimitReached`.** Production enforces a query-depth limit of 10. A query deeper than that fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data. It's a plain GraphQL error, not the catalogued shape.

Your client has to handle both shapes: catalogued errors with `errorInfo`, and plain GraphQL errors without it.

---

## Rate-limited requests

A credential that has spent its rate-limit budget for the current window gets `RATE_LIMITED` until the window ends. Rate limits aren't enforced yet; budgets are measured, but nothing is refused.

A denied request is refused before any resolver runs, so nothing was read or changed. It's raised the same way on every API and operation class, **mutations and Tokenization API calls included**: HTTP `200`, a top-level `errors[]` entry and a `null` field, never an in-band `error`:

```json
{
  "data": null,
  "errors": [
    {
      "path": ["createLab"],
      "errorType": "RATE_LIMITED",
      "message": "Too many requests. Retry with exponential backoff.",
      "errorInfo": {
        "requestId": "8f1e4c9a-2b7d-4e10-9c3a-5d6f7a8b9c0d",
        "retryable": true,
        "details": {
          "reason": "COST_BUDGET_EXHAUSTED",
          "retryAfterSeconds": 37,
          "remainingBudget": 0,
          "budgetWindowSeconds": 300
        }
      }
    }
  ]
}
```

| `details` key | Type | Meaning |
| --- | --- | --- |
| `reason` | string | `COST_BUDGET_EXHAUSTED` |
| `retryAfterSeconds` | integer ≥ 0 | Seconds until the current window ends. `0` means retry once now, then back off |
| `remainingBudget` | integer ≥ 0 | Cost units left in the window. Currently always `0` on a denial |
| `budgetWindowSeconds` | integer > 0 | Length of the window |

Wait `retryAfterSeconds`, plus a little jitter, then retry the same request once. If it's denied again, back off exponentially, capped at `budgetWindowSeconds`. Never tight-loop: every retry is charged too.

---

## Labs API errors

The Labs API is a GraphQL API: once a request is accepted, the response is HTTP `200` whether or not the operation succeeded. Success and failure are signalled inside the JSON body, not by the status code. Errors surface through one of two channels, depending on the operation class:

- **Queries throw.** A failed query adds an entry to the top-level GraphQL `errors[]` array and returns `null` for that field. Most Labs query result types are non-null, so the null propagates and `data` itself comes back `null` (as in the example below); only `labWithDataRoomAndFiles` and `dataRoomFile` are nullable and null just their own field. `errorType` carries the error code (the only value to branch on) and `errorInfo` carries `{ requestId, retryable, details }`.
- **Mutations return errors in-band.** Every mutation result type carries an `error: ApiError` field. **Success ⇔ `error == null`.** Where the result type also has a top-level `message`, it mirrors `error.message` on failure and is never empty. A top-level `errors[]` entry on a mutation means a transport or infrastructure failure, that the request document itself failed validation, or a [`RATE_LIMITED`](#rate-limited-requests) denial. In all three cases the mutation didn't run.

Include `requestId` whenever you report a problem.

### Failed query

```json
{
  "data": null,
  "errors": [
    {
      "path": ["listLabMembers"],
      "message": "Project not found: 0x0101000000000000000000000000000000000000000000000000000000000042",
      "errorType": "NOT_FOUND",
      "errorInfo": {
        "requestId": "8f1e4c9a-2b7d-4e10-9c3a-5d6f7a8b9c0d",
        "retryable": false,
        "details": { "reason": "PROJECT_NOT_FOUND" }
      }
    }
  ]
}
```

### Failed mutation

Select `error { code message requestId retryable details }` on every mutation:

```graphql
type ApiError {
  code: String!       # error code from the table below — the only field to branch on
  message: String!    # human-readable, never empty; not part of the contract
  requestId: String!  # correlation id — include it in bug reports
  retryable: Boolean! # whether retrying the same request unchanged can plausibly succeed
  details: AWSJSON    # optional structured context (JSON-encoded string)
}
```

```graphql
mutation InitiateFileUpload($oclId: String!, $contentType: String!, $contentLength: Int!) {
  initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
    uploadUrl
    error { code message requestId retryable details }
  }
}
```

```json
{
  "data": {
    "initiateCreateOrUpdateFile": {
      "uploadUrl": null,
      "error": {
        "code": "UNAUTHORIZED",
        "message": "You are not allowed to perform this operation.",
        "requestId": "8f1e4c9a-2b7d-4e10-9c3a-5d6f7a8b9c0d",
        "retryable": false,
        "details": "{\"reason\":\"UNAUTHORIZED\"}"
      }
    }
  }
}
```

### Reading `details`

`details` reaches you in more than one shape, so **read it through a tolerant parse rather than a single `JSON.parse`**: it is a JSON-encoded string on in-band mutation errors, a plain object on thrown query errors (`errorInfo.details`), and the in-band string is currently encoded twice — a single parse there returns another string, and `.reason` on it is silently `undefined`. Parsing until the value stops being a string reads all three correctly and needs no change when the encoding is corrected.

Documented keys are `field` (the offending input field), `reason` (a more specific cause under the code, e.g. `PROJECT_NOT_FOUND` under `NOT_FOUND`), `hint` and `docs`; ignore unknown keys. `reason` values are diagnostic refinement and may be extended at any time — branch on `code` first.

A few codes carry extra keys that are part of the contract:

| Code and `reason` | Extra keys |
| --- | --- |
| `RATE_LIMITED` / `COST_BUDGET_EXHAUSTED` | `retryAfterSeconds`, `remainingBudget`, `budgetWindowSeconds`. See [Rate-limited requests](#rate-limited-requests) |
| `COMPLEXITY_LIMIT_EXCEEDED` / `RESULT_CARDINALITY_LIMIT` | `field` (the list, as `Type.field`) and `limit` (the largest list it serves). See [list limits](changelog.md#lists-without-paging-arguments-now-fail-above-a-size-limit) |
| `UNAUTHORIZED` / `INTERNAL_FIELD` | `fields`: the fields, as `Type.field`, that aren't part of the public API. See [Labs troubleshooting](#labs-troubleshooting) |

Keys that only one operation emits are documented on that operation's page.

```javascript
// Handles all three shapes: object, JSON string, doubly-encoded JSON string.
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

const result = (await response.json()).data.initiateCreateOrUpdateFile; // `response` from your fetch()

if (result.error) {
  const { code, message, requestId, retryable, details } = result.error;
  const { reason } = parseDetails(details);
  if (retryable) return retryWithBackoff(); // RATE_LIMITED, TIMEOUT, UPSTREAM_UNAVAILABLE, INTERNAL_ERROR
  throw new Error(`${code}${reason ? `/${reason}` : ""}: ${message} (requestId ${requestId})`);
}
```

### Labs error codes

| Code                        | `retryable` | Meaning                                                                                              |
| --------------------------- | ----------- | ---------------------------------------------------------------------------------------------------- |
| `UNAUTHENTICATED`           | **false**   | Missing, invalid or expired credentials                                                              |
| `UNAUTHORIZED`              | **false**   | Authenticated, but not allowed (role/membership)                                                     |
| `NOT_FOUND`                 | **false**   | Referenced resource doesn't exist                                                                    |
| `VALIDATION_FAILED`         | **false**   | Input failed validation (unknown filter/sort fields, out-of-range pagination, malformed ids); `details.field` names the offender |
| `CONFLICT`                  | **false**   | Valid request conflicts with current state (e.g. `details.reason` `SHORTNAME_TAKEN`, `ALREADY_SIGNED`) |
| `FAILED_PRECONDITION`       | **false**   | Resource state makes the operation impossible until the state changes (e.g. `TEMPLATE_EXPIRED`)      |
| `COMPLEXITY_LIMIT_EXCEEDED` | **false**   | Query shape or result size over limits                                                               |
| `RATE_LIMITED`              | **true**    | Throttled — retry with backoff                                                                       |
| `TIMEOUT`                   | **true**    | Execution exceeded the request budget                                                                |
| `UPSTREAM_UNAVAILABLE`      | **true**    | A dependency failed (`details.reason` `KAMU`, `CMS`, `IPFS`)                                         |
| `INTERNAL_ERROR`            | **true**    | Unexpected failure — details are only in our logs, joined by `requestId`                             |

The typical `details.reason` values under each code are listed in the [API Changelog](changelog.md#error-codes). Codes may be added over time; see [Unknown codes](#unknown-codes).

### Labs troubleshooting

**`UNAUTHENTICATED`** — missing, invalid or expired service token:

- Ensure the `X-Service-Token` header is included in mutation requests
- Verify the token is not empty or malformed
- If the token has expired, issue a new one yourself — the two-call [sign-in flow](labs-api/service-tokens.md#obtaining-a-token) needs no human — or extend the existing one with `extendServiceToken`
- The sign-in path's own failure reasons are listed on [Service Tokens](labs-api/service-tokens.md)

An HTTP `401` instead of `UNAUTHENTICATED` means the consumer credential was rejected, not the service token — see [Failures before your operation runs](#failures-before-your-operation-runs).

**`UNAUTHORIZED`** — the wallet behind the service token lacks the required role on the lab:

- Check the wallet's role with the public `listLabMembers(oclId)` query. Content writes (uploads, metadata, moves, deletes) need **Contributor**; `createLab` and the LabNFT-metadata mutations need **Owner**
- Not the right role? The lab owner grants one onchain — see [Agent access](getting-started/agent-as-a-lab-contributor.md)
- **Just granted the role?** Role state reaches the API through an event indexer, so a write can still return `UNAUTHORIZED` for a few seconds after the grant confirms onchain. Retry with backoff; re-issuing the token does not help
- **`details.reason` is `INTERNAL_FIELD`?** The request selects, filters or sorts by a field that isn't part of the public API, and `details.fields` lists them. Nothing was read. Remove those fields from the query; if your integration needs one, ask Molecule for access. This check only logs today and refuses nothing; enforcement will be announced in the [API Changelog](changelog.md)

**`NOT_FOUND`** — lab, dataroom or file not found:

- Verify the `oclId` refers to a registered lab
- For `updateFileMetadata` / `deleteDataRoomFile`: verify the file `ref` (DID) or `path` is correct and the file exists in the specified dataroom

**`VALIDATION_FAILED`** — invalid parameters; `details.field` names the offending input:

- Check that the `oclId` format is correct: a 32-byte hex string with `0x` prefix
- For `searchLabs`: verify filter values match expected types (arrays of strings)

**Retryable errors (`RATE_LIMITED`, `TIMEOUT`, `UPSTREAM_UNAVAILABLE`, `INTERNAL_ERROR`):**

- Retry with exponential backoff; if the failure persists, report it with the `requestId`
- One known exception: a bad `expiresIn` on `generateServiceToken` comes back as `INTERNAL_ERROR` / `TOKEN_GENERATION_FAILED` and never succeeds on retry — see [Service Tokens](labs-api/service-tokens.md)

---

## Tokenization API errors

The Tokenization API keeps its own envelope. It did not move to `ApiError`, so handle it separately from the Labs API. Every result, **queries included**, carries a success flag and an `error` object, and returns HTTP `200` either way:

```graphql
type EvmTokenizationError {
  message: String     # human-readable; not part of the contract
  code: String        # failure code from the table below
  retryable: Boolean! # whether retrying the same request unchanged can plausibly succeed
  details: AWSJSON    # currently always null for this service
}
```

```json
{
  "data": {
    "getOclTermsMessage": {
      "message": null,
      "isSuccess": false,
      "error": {
        "message": "contentHash does not match the agreement document at agreementKey",
        "code": "INVALID_INPUT",
        "retryable": false,
        "details": null
      }
    }
  }
}
```

How it differs from the Labs API:

| | Labs API | Tokenization API |
| --- | --- | --- |
| Success check | `error == null` | `isSuccess == true` (and `error == null`) |
| Queries | throw into `errors[]` | return `isSuccess: false` + `error` in-band, like mutations |
| `code` | always set (`String!`) | nullable (`String`); treat a missing code as non-retryable |
| `requestId` | on every error | not present |
| `details` | structured, read with [`parseDetails`](#reading-details) | always `null` |

Select `error { code message retryable }` on every tokenization operation, not just `error { message }`.

Four failures bypass the envelope and arrive as catalogued, thrown GraphQL errors (top-level `errors[]`, `errorType`, with `errorInfo.requestId`). Handle them the same way as a [Labs query error](#failed-query):

- `COMPLEXITY_LIMIT_EXCEEDED`: the selection set on a tokenization field is too deep or selects too many fields. Trim it. Not retryable.
- `RATE_LIMITED`: the credential's cost budget is spent. The operation didn't run. See [Rate-limited requests](#rate-limited-requests). Retryable.
- `TIMEOUT`: the resolver ran out of time and was stopped. Retryable. On a mutation, the work may have partly happened, so check the state before resending.
- `INTERNAL_ERROR`: an unhandled failure inside the resolver, masked before it reaches you. Retryable.

### Tokenization error codes

| Code | `retryable` | Meaning |
| --- | --- | --- |
| `INVALID_INPUT` | false | Required fields missing or malformed, or an invalid `oclId`. From `getOclTermsMessage`, also: no agreement at `agreementKey`, or a `contentHash` that doesn't match the stored agreement |
| `INVALID_METADATA` | false | `agreementData` or the metadata JSON could not be parsed |
| `MISSING_METADATA_FIELD` | false | The metadata is missing a required field |
| `UNSUPPORTED_IMAGE_TYPE` | false | The image content type isn't accepted |
| `IMAGE_UPLOAD_FAILED` | false | The referenced image wasn't found in storage. Upload it before referencing its key |
| `INVALID_TERMS_SIGNATURE` | false | The signature doesn't verify against the terms message |
| `METADATA_UPLOAD_FAILED` | **true** | Storing the metadata failed |
| `TERMS_MESSAGE_FAILED` | **true** | Building the terms message failed |
| `SIGNOFF_FAILED` | **true** | The backend could not produce its authorization signature |
| `INTERNAL_ERROR` | **true** | Unexpected failure, e.g. agreement generation or reading the stored agreement |

The OCL flow documented on the [Tokenization API](tokenization-api.md) page (`generateOclMembershipAgreement`, `getOclTermsMessage`) emits `INVALID_INPUT`, `INVALID_METADATA` and `INTERNAL_ERROR`. The rest come from the IP-NFT and IPT operations on the same service.

The in-band `error` has no `requestId` to quote. When reporting a tokenization failure, include the operation name, the time of the request and the `error.message`.

Contract reverts (`AlreadyTokenized()`, `MustControlLab()`) are a separate channel: they come from the onchain transaction, not from the API. See [Tokenization API › Error Handling](tokenization-api.md#error-handling).

---

## x402 Gateway errors

A call through the [x402 Gateway](x402-gateway.md) can fail in two places: in the gateway itself, which returns a non-`200` status with its own small body, or in the forwarded Labs mutation, which returns `200` with the usual in-band `error`. Check the HTTP status first.

### Gateway responses

Every error body the gateway builds itself has the same shape: `{"isSuccess": false, "message": "…"}`. There's no `code` field. One `500` also carries an `error` string with internal detail; ignore it and read `message`.

| Status | `message` | What happened | What to do |
| --- | --- | --- | --- |
| `402` | `Payment required` | No payment header (`Payment-Signature`, `X-Payment` or `Payment`) was sent. The payment requirements are in the base64-encoded `payment-required` **header**, not the body | Normal first step of the handshake. [Read the challenge](x402-gateway.md#reading-the-402-challenge), sign, resend |
| `402` | `Payment verification failed` | A payment was sent, but the facilitator rejected it, for example because the amount, asset, network or `payTo` doesn't match the challenge, the authorization expired, or the payload was already used | Re-read the challenge, build and sign a fresh payment, resend. Nothing was charged and the mutation didn't run |
| `400` | `Missing path parameter: mutation`, `Mutation '<name>' is not enabled for x402 gateway`, `Request body is required`, `Body must include GraphQL mutation in 'query'` | The request is malformed or targets a mutation that isn't on the [allow-list](x402-gateway.md#endpoints) | Fix the request. Nothing was charged |
| `400` | `Route configuration error: payment should be required` | The gateway is misconfigured for this mutation. Not something your request caused | Report it. Nothing was charged and the mutation didn't run |
| `400` | `Unable to determine payer address from verified payment payload` | The payment verified, but no valid payer address could be resolved from it | Check that the signed authorization carries a valid `from` address (see [Payer address resolution](x402-gateway.md#payer-address-resolution)). Nothing was charged and the mutation didn't run |
| `500` | The parse or validation reason, e.g. `Path mutation '<path>' does not match GraphQL field '<field>'`, `Only GraphQL mutation operations are accepted`, `Mutation must contain exactly one top-level field` | The body isn't valid JSON, or the `query` isn't exactly one mutation whose single top-level field matches the path | Fix the document. Rejected before payment is verified, so nothing was charged |
| `500` | Any other message | An unexpected gateway failure. It can happen **after** the mutation ran, for example while contacting the facilitator to settle | Don't resend blindly. Read back the state the mutation would have changed first (see [Retrying](#retrying)) |
| Upstream `4xx`/`5xx` | The upstream body, as-is | The Labs API rejected the forwarded request at the HTTP level | Settlement is skipped, so nothing was charged. Handle it as the upstream error |

`PAYMENT_REQUIRED` is not a code any Molecule response carries. A missing or rejected payment is HTTP `402`. If your client maps errors into a single code space, map `402` to your own `PAYMENT_REQUIRED` value, and keep the two `402` cases apart: if you sent no payment header, it's the normal handshake; if you sent one, the payment was rejected.

### Forwarded mutation

A `200` body is the Labs API response verbatim. Read it with the [Labs mutation contract](#failed-mutation): success is `error == null`; otherwise branch on `error.code`.

**A `200` is settled even when the mutation failed.** Settlement is triggered by the upstream `2xx`, not by mutation success, so a `200` with `error.code: "UNAUTHORIZED"` or `"VALIDATION_FAILED"` has been paid for. Payment buys a short-lived service token for the payer wallet; it doesn't grant a role. Validate the target lab, your role on it (`listLabMembers`) and your inputs **before** signing.

**One exception: a `200` with a `RATE_LIMITED` error didn't run.** The gateway calls the Labs API with its own credential, which has a [rate limit](#rate-limited-requests) like any other. A denial comes back as HTTP `200` with a top-level `errors[]` entry whose `errorType` is `RATE_LIMITED`. The mutation didn't run, but the request was still settled, because settlement follows the HTTP status. Wait `errorInfo.details.retryAfterSeconds`, then resend with a freshly signed payment. Rate limits aren't enforced yet, so you won't see this today.

### Settlement result

The settlement outcome comes back in the `payment-response` response header on `200` responses: base64-encoded JSON. It has `success`, plus `transaction` and `network` on success, or `errorReason` on failure.

**If settlement fails, the gateway still returns `200` with the mutation's result.** The mutation already ran, and you weren't charged for it. Don't re-sign and resend: that runs the mutation a second time (a second lab, a second file version). Treat the mutation result as final and log the `errorReason`.

```javascript
const raw = res.headers.get("payment-response");
const settlement = raw ? JSON.parse(atob(raw)) : null;
if (settlement && !settlement.success) {
  // Mutation ran, payment didn't settle. Don't resend.
  console.warn("x402 settlement failed:", settlement.errorReason);
}
```

---

## Retrying

| Situation | Retry? |
| --- | --- |
| Labs or Tokenization error with `retryable: true` | Yes, with exponential backoff and jitter, and a cap on attempts |
| `RATE_LIMITED` | Yes, after `details.retryAfterSeconds`, then with backoff capped at `budgetWindowSeconds`. See [Rate-limited requests](#rate-limited-requests) |
| `retryable: false` | No. Change the request, or wait for the resource state to change |
| Labs `UNAUTHORIZED` just after an onchain role grant | Yes, with backoff for a few seconds; [indexer lag](#labs-troubleshooting), not a real denial |
| `INTERNAL_ERROR` / `TOKEN_GENERATION_FAILED` on `generateServiceToken` | No. Flagged retryable but permanent; fix `expiresIn` |
| x402 `402 Payment verification failed` | Yes, with a **freshly signed** payment. Never replay the same payload |
| x402 `200` with a top-level `RATE_LIMITED` error | Yes, after `retryAfterSeconds`, with a freshly signed payment. The mutation didn't run, but the first payment was settled |
| x402 `200`, any other body | No automatic resend. The mutation ran and the request was settled (or failed to settle, which still means it ran) |
| x402 `500` with an unexpected message | Only after reading back that the mutation didn't take effect |

Retrying a mutation is only safe when you know it didn't take effect. On the Labs API, a retryable in-band error means the mutation failed. Through x402, a `200` or a late `500` can mean it succeeded.

---

## Unknown codes

Codes may be added over time, and each addition is published in the [API Changelog](changelog.md). When you get a code you don't recognise, from any of the three APIs:

- Treat it as **non-retryable**.
- Keep the raw value, and the `requestId` if there is one, for diagnostics.
- Surface it to a human rather than guessing at a recovery.

---

## Reporting a problem

| API | Quote |
| --- | --- |
| Labs API | `requestId` (in `error.requestId` or `errorInfo.requestId`) |
| Tokenization API | operation name, request time, `error.code` and `error.message` |
| x402 Gateway | the HTTP status and `message`; on a `200`, the upstream `requestId` and the settlement `transaction` hash from `payment-response` |

---

## Related

- [Authentication](authentication.md) — credentials, headers, and which role each operation needs
- [API Types](types.md) — the full `ApiError` and `EvmTokenizationError` definitions
- [API Changelog & Migration](changelog.md) — new codes, and the `details.reason` values under each
- [Agent one-pager](getting-started/for-agents.md) — the error contract condensed for agents

{% include "../.gitbook/includes/support.md" %}
