# ⚙️ Labs API

## Overview

The Labs API allows developers to interact with Molecule Labs datarooms without requiring browser-based user interaction. This enables integration with automated workflows, data pipelines, CI/CD systems, and external applications.

### Use Cases

- **Automated Data Pipelines**: Schedule regular data synchronization from research systems
- **CI/CD Integration**: Automatically publish build artifacts and test results
- **External System Integration**: Connect third-party tools and platforms to your Lab
- **Batch Operations**: Upload multiple files programmatically
- **Monitoring & Alerting**: Automated upload of logs and metrics

> **Ready for Production**: This API is production-ready and actively used by projects for automated data management. To request API access, please join our [Discord community](https://t.co/L0VEiy4Bjk) and reach out to our team.

---

## Authentication

The Labs API uses API-key authentication for reads and an additional Service Token for writes. Full details — public queries vs. protected mutations, obtaining and using credentials — are on the [Authentication](../authentication.md) page.

See also the functional sections: [Lab Management](lab-management.md), [Access Policies](access-policies.md), [Files](files.md), [Browse & Search](browse-and-search.md), [Legal Agreements](legal-agreements.md), and [Service Tokens](service-tokens.md).

> **Permissionless Labs.** By default a Lab is role-gated: only its owner and contributors can write. A Lab owner can additionally open specific capabilities — file contributions, edits, deletions, announcements — to any authenticated caller, to a deadline, or to wallets satisfying an onchain condition. Note that permissionless does not mean unauthenticated; see [Access Policies](access-policies.md).

---

## Error Handling

All API responses follow a consistent error format:

### Error Response Structure

```json
{
  "data": {
    "initiateCreateOrUpdateFile": {
      "isSuccess": false,
      "error": {
        "message": "Error description",
        "code": "ERROR_CODE",
        "retryable": true
      }
    }
  }
}
```

### Common Error Codes

| Status Code | Error                 | Description                                                   |
| ----------- | --------------------- | ------------------------------------------------------------- |
| 401         | Unauthorized          | Missing or invalid service token                              |
| 403         | Forbidden             | Service token does not have access to the specified lab       |
| 400         | Bad Request           | Invalid parameters (e.g., missing oclId, invalid contentType) |
| 404         | Not Found             | Lab or dataroom not found                                     |
| 413         | Payload Too Large     | File exceeds size limits                                      |
| 500         | Internal Server Error | Server error - check if retryable and try again               |

### Troubleshooting

**"Missing service token" error:**

- Ensure `X-Service-Token` header is included in requests
- Verify token is not empty or malformed

**"Service does not have access to lab" error:**

- Verify your wallet address (linked to service token) has admin access to the lab/dataroom
- Check that the oclId format is correct: a 32-byte hex string with `0x` prefix

**"Token expired" error:**

- Request a new token from the Molecule team, or
- Use `extendServiceToken` mutation to extend expiration

**Upload to presigned URL fails:**

- Ensure binary file upload (use `--data-binary` in curl)
- Verify headers match those returned in Step 1
- Check that presigned URL hasn't expired (expires after \~15 minutes)

**"File not found" error (updateFileMetadata, deleteDataRoomFile):**

- Verify the file `ref` (DID) or `path` is correct
- Check that the file exists in the specified dataroom
- Ensure you have access to the dataroom

**"Invalid search filters" error (searchLabs):**

- Verify filter values match expected types (arrays of strings)
- Ensure oclId format is correct if using byOclIds filter

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

If you encounter any issues or have questions about the Programmatic File Upload API:

1. Check this documentation and [troubleshooting section](#troubleshooting)
2. Review the [complete example](files.md#complete-example) for implementation guidance
3. Join our [Discord community](https://t.co/L0VEiy4Bjk) for support
4. Contact the Molecule Labs development team directly

---

_Last updated: July 2026_
