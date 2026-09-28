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
| **A term here is unfamiliar** | [Glossary](../../references/glossary.md) — Lab, `oclId`, data room, service token, indexer |
| **You want runnable code** | [Create a lab and upload a public file](../getting-started/create-lab-and-upload-file.md) · [Upload an encrypted file](../getting-started/upload-encrypted-file.md) · [Agent access](../getting-started/agent-as-a-lab-contributor.md) |
| **You're an AI agent** | [Agent one-pager](../getting-started/for-agents.md), or drive this API through the [Molecule Skill](../../ai-tooling/molecule-skill.md) plugin |
| **You want to pay per call** | [x402 Gateway](../x402-gateway.md) |

---

## Authentication

The Labs API uses consumer-credential authentication for reads and an additional Service Token for writes — which callers **issue for themselves** by signing a message with their wallet. Full details — public queries vs. protected mutations, obtaining and using credentials — are on the [Authentication](../authentication.md) page.

See also the functional sections: [Lab Management](lab-management.md), [Files](files.md), [Browse & Search](browse-and-search.md), and [Service Tokens](service-tokens.md). For runnable end-to-end walkthroughs, see the [tutorials](../getting-started/README.md) under Getting Started.

---

## Error Handling

The full error contract (both channels, every code, how to read `details`, troubleshooting and retry rules) is on the shared [Errors](../errors.md) page. In short:

- **Queries throw.** A failed query adds an entry to the top-level `errors[]` array; branch on `errorType`, and read `requestId`, `retryable` and `details` from `errorInfo`. See [Failed query](../errors.md#failed-query).
- **Mutations return errors in-band.** Every mutation result carries `error: ApiError`, and **success ⇔ `error == null`**. Select `error { code message requestId retryable details }` on every mutation. See [Failed mutation](../errors.md#failed-mutation).
- **Read `details` with the tolerant [`parseDetails`](../errors.md#reading-details)**, never a single `JSON.parse`. The in-band string is currently encoded twice.
- **Branch on the code, never on `message`.** Retry only when `retryable` is `true`, and quote `requestId` when reporting a problem. See [Labs error codes](../errors.md#labs-error-codes) and [Labs troubleshooting](../errors.md#labs-troubleshooting).

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

## Related

* [Getting Started](../getting-started/README.md) — prerequisites and the ten-minute quickstart
* [Authentication](../authentication.md) — credentials and headers
* [API Types](../types.md) — every result and input type
* [API Changelog & Migration](../changelog.md) — breaking changes and migrations
* [Release Notes](../../release-notes/README.md) — what changed in each release

## Getting Support

If you encounter any issues or have questions about the Labs API:

1. Check this documentation and the [troubleshooting section](../errors.md#labs-troubleshooting)
2. Run the [Tutorials](../getting-started/README.md) against staging — each step lists its expected response and failure modes
3. Join our [Discord community](https://t.co/L0VEiy4Bjk) for support, quoting the `requestId` from the failing response

---

_Last updated: July 2026_

{% include "../../.gitbook/includes/support.md" %}
