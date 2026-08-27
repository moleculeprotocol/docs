# ⚙️ Labs API

## Overview

The Labs API allows developers to interact with Molecule Labs datarooms without requiring browser-based user interaction. This enables integration with automated workflows, data pipelines, CI/CD systems, and external applications.

### Use Cases

- **Automated Data Pipelines**: Schedule regular data synchronization from research systems
- **CI/CD Integration**: Automatically publish build artifacts and test results
- **External System Integration**: Connect third-party tools and platforms to your Lab
- **Batch Operations**: Upload multiple files programmatically
- **Monitoring & Alerting**: Automated upload of logs and metrics

> **Ready for Production**: This API is production-ready and actively used by projects for automated data management. To get started, see [🚀 Getting Started](../getting-started/README.md) — it covers the one credential you need to request and gets you to a lab with a file in it in about ten minutes.

---

## Where to start

| | |
| --- | --- |
| **First time here** | [🚀 Getting Started](../getting-started/README.md) — prerequisites, costs, ten-minute quickstart |
| **You want runnable code** | [Tutorials](example-workflow.md) — public upload, encrypted upload, agent access, announce |
| **You're an AI agent** | [Agent one-pager](../getting-started/for-agents.md), or drive this API through the [Molecule Skill](../../ai-tooling/molecule-skill.md) plugin |
| **You want to pay per call** | [x402 Gateway](../x402-gateway.md) |

---

## Authentication

The Labs API uses consumer-credential authentication for reads and an additional Service Token for writes — which callers **issue for themselves** by signing a message with their wallet. Full details — public queries vs. protected mutations, obtaining and using credentials — are on the [Authentication](../authentication.md) page.

See also the functional sections: [Tutorials](example-workflow.md), [Lab Management](lab-management.md), [Files](files.md), [Browse & Search](browse-and-search.md), and [Service Tokens](service-tokens.md).

---

## Error Handling

The Labs API is a GraphQL API: once a request is accepted, the response is HTTP `200` whether or not the operation succeeded — success and failure are signalled inside the JSON body, not by the status code. Errors surface through one of two channels, depending on the operation class:

- **Queries throw.** A failed query adds an entry to the top-level GraphQL `errors[]` array and returns `null` for that field. Most Labs query result types are non-null, so the null propagates and `data` itself comes back `null` (as in the example below); only `labWithDataRoomAndFiles` and `dataRoomFile` are nullable and null just their own field. `errorType` carries the error code (the only value to branch on) and `errorInfo` carries `{ requestId, retryable, details }`.
- **Mutations return errors in-band.** Every mutation result type carries an `error: ApiError` field. **Success ⇔ `error == null`.** Where the result type also has a top-level `message`, it mirrors `error.message` on failure and is never empty. A top-level `errors[]` entry on a mutation means a transport or infrastructure failure, or that the request document itself failed validation.

Branch on the code — never on `message` text, which may change without notice. Include `requestId` whenever you report a problem.

### Failed Query

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

### Failed Mutation

Select `error { code message requestId retryable details }` on every mutation:

```graphql
type ApiError {
  code: String!       # error code from the catalogue below — the only field to branch on
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

In-band `details` is a JSON-encoded string — parse it with `JSON.parse(error.details ?? "{}")` (on thrown queries, `errorInfo.details` is already an object). Documented keys are `field` (the offending input field), `reason` (a more specific cause under the code, e.g. `PROJECT_NOT_FOUND` under `NOT_FOUND`), `hint` and `docs`; ignore unknown keys. `reason` values are diagnostic refinement and may be extended at any time — branch on `code` first.

```javascript
const result = (await response.json()).data.initiateCreateOrUpdateFile; // `response` from your fetch()

if (result.error) {
  const { code, message, requestId, retryable, details } = result.error;
  const { reason } = JSON.parse(details ?? "{}");
  if (retryable) return retryWithBackoff(); // RATE_LIMITED, TIMEOUT, UPSTREAM_UNAVAILABLE, INTERNAL_ERROR
  throw new Error(`${code}${reason ? `/${reason}` : ""}: ${message} (requestId ${requestId})`);
}
```

### Error Codes

| Code                        | `retryable` | Meaning                                                                                              |
| --------------------------- | ----------- | ---------------------------------------------------------------------------------------------------- |
| `UNAUTHENTICATED`           | false       | Missing, invalid or expired credentials                                                              |
| `UNAUTHORIZED`              | false       | Authenticated, but not allowed (role/membership)                                                     |
| `NOT_FOUND`                 | false       | Referenced resource doesn't exist                                                                    |
| `VALIDATION_FAILED`         | false       | Input failed validation (unknown filter/sort fields, out-of-range pagination, malformed ids); `details.field` names the offender |
| `CONFLICT`                  | false       | Valid request conflicts with current state (e.g. `details.reason` `SHORTNAME_TAKEN`, `ALREADY_SIGNED`) |
| `FAILED_PRECONDITION`       | false       | Resource state makes the operation impossible until the state changes (e.g. `TEMPLATE_EXPIRED`)      |
| `COMPLEXITY_LIMIT_EXCEEDED` | false       | Query shape or result size over limits                                                               |
| `RATE_LIMITED`              | **true**    | Throttled — retry with backoff                                                                       |
| `TIMEOUT`                   | **true**    | Execution exceeded the request budget                                                                |
| `UPSTREAM_UNAVAILABLE`      | **true**    | A dependency failed (`details.reason` `KAMU`, `CMS`, `IPFS`)                                         |
| `INTERNAL_ERROR`            | **true**    | Unexpected failure — details are only in our logs, joined by `requestId`                             |

When `retryable` is `true`, retry with exponential backoff; when `false`, the request (or the resource state) must change before retrying. Any code not listed here: preserve it for diagnostics, treat it as non-retryable and surface it to a human — new codes are announced in the [API Changelog](../changelog.md). `PAYMENT_REQUIRED` is reserved for the [x402 Gateway](../x402-gateway.md) and is not emitted by the GraphQL API.

### Troubleshooting

**`UNAUTHENTICATED`** — missing, invalid or expired service token:

- Ensure the `X-Service-Token` header is included in mutation requests
- Verify the token is not empty or malformed
- If the token has expired, issue a new one yourself — the two-call [sign-in flow](service-tokens.md#obtaining-a-token) needs no human — or extend the existing one with `extendServiceToken`

A missing or malformed consumer credential is rejected before the GraphQL layer runs (an HTTP `401` from the API, not one of the error codes below) — check the `Authorization` header first, see [Authentication](../authentication.md).

**`UNAUTHORIZED`** — the wallet behind the service token lacks the required role on the lab:

- Check the wallet's role with the public `listLabMembers(oclId)` query. Content writes (uploads, metadata, announcements, moves, deletes) need **Contributor**; `createLab` and the LabNFT-metadata mutations need **Owner**
- Not the right role? The lab owner grants one onchain — see [Tutorial 3](example-workflow.md#tutorial-3-give-your-agent-access-to-a-lab-you-created-in-the-app)
- **Just granted the role?** Role state reaches the API through an event indexer, so a write can still return `UNAUTHORIZED` for a few seconds after the grant confirms onchain. Retry with backoff; re-issuing the token does not help

**Upload to presigned URL fails:**

- Ensure binary file upload (use `--data-binary` in curl)
- Verify headers match those returned in Step 1
- Check that presigned URL hasn't expired (expires after \~15 minutes)

**`NOT_FOUND`** — lab, dataroom or file not found:

- Verify the `oclId` refers to a registered lab
- For `updateFileMetadata` / `deleteDataRoomFile`: verify the file `ref` (DID) or `path` is correct and the file exists in the specified dataroom

**`VALIDATION_FAILED`** — invalid parameters; `details.field` names the offending input:

- Check that the `oclId` format is correct: a 32-byte hex string with `0x` prefix
- For `searchLabs`: verify filter values match expected types (arrays of strings)

**Retryable errors (`RATE_LIMITED`, `TIMEOUT`, `UPSTREAM_UNAVAILABLE`, `INTERNAL_ERROR`):**

- Retry with exponential backoff; if the failure persists, report it with the `requestId`

---

## Best Practices

### Token Security

- **Never commit tokens** to version control (add to `.gitignore`)
- **Use environment variables** to store tokens
- **Rotate tokens regularly** (quarterly recommended)
- **Use secrets management systems** in production (AWS Secrets Manager, HashiCorp Vault, etc.)
- **Revoke immediately** if a token is compromised

### Storage Management

- Monitor your 5GB storage limit per project
- Organize files with meaningful names and metadata
- Use categories and tags for easy file discovery
- Clean up old or unnecessary files regularly

### Metadata Best Practices

- **Use descriptive tags**: `["experiment-1", "2024-q4", "preliminary"]`
- **Organize with categories**: `["raw-data", "analysis", "results"]`
- **Add descriptions**: Help collaborators understand file contents
- **Include searchable text** (`contentText`): Enables full-text search via `searchLabs`
- **Update metadata as needed**: Use `updateFileMetadata` to refine tags and descriptions without re-uploading files

### Search and Discovery

- **Use contentText**: Populate `contentText` field when uploading files to enable full-text search
- **Tag consistently**: Use consistent tag names across files for better filtering
- **Filter strategically**: Combine filters (tags + access levels) to narrow search results
- **Test search queries**: Use `searchLabs` to verify your files are discoverable

---

## Deprecated & Renamed Operations

The legacy `*V2` operations and the pre-OCL naming have been **removed**. The current API is `oclId`-based. If you are migrating from an older integration, see the query/mutation/field rename tables in the [API Changelog & Migration](../changelog.md#labs-api) page.

---

## Getting Support

If you encounter any issues or have questions about the Labs API:

1. Check this documentation and the [troubleshooting section](#troubleshooting)
2. Run the [Tutorials](example-workflow.md) against staging — each step lists its expected response and failure modes
3. Join our [Discord community](https://t.co/L0VEiy4Bjk) for support, quoting the `requestId` from the failing response

---

_Last updated: July 2026_
