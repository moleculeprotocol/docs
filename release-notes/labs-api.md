---
description: Consumer-visible changes to the Labs API, newest first.
icon: flask
---

# Labs API release notes

Changes to the [Labs API](../api-reference/labs-api/README.md) that affect integrations. Versions not
listed shipped nothing consumer-visible.

## 3.0.1

_Released 2026-08-25_

### Changed

#### Assignment Agreement no longer required before data-room writes

Data-room write mutations — `initiateCreateOrUpdateFile`, `finishCreateOrUpdateFile`,
`deleteDataRoomFile`, `updateFileMetadata`, `moveEntry`, and `createAnnouncement` (see
[Files](../api-reference/labs-api/files.md)) — used to fail with `ASSIGNMENT_AGREEMENT_NOT_SIGNED`
(`FAILED_PRECONDITION`) until a lab's Assignment Agreement was signed via `signLegalAgreement`. That
gate is now disabled: these mutations succeed regardless of the agreement's sign state.
[Sign Legal Agreement and Check Legal Agreement Status](../api-reference/labs-api/legal-agreements.md)
are unchanged and remain fully functional — signing is now optional rather than a write precondition.

**Migration:** No action required. Integrations that retried after handling
`ASSIGNMENT_AGREEMENT_NOT_SIGNED` can drop that handling; it is safe to leave in place, since the
error is simply no longer emitted for this type.

## 2.0.1

_Released 2026-08-18_

### Breaking changes

#### GraphQL introspection disabled and query depth capped in production

The production endpoint no longer serves `__schema` / `__type` introspection queries — they return a validation error. `__typename` still resolves. Selection-set depth is also capped at 10 in production; a query beyond that fails at execution time with `errorType: "QueryDepthLimitReached"` rather than the usual error shape. See [API Changelog & Migration](../api-reference/changelog.md#graphql-introspection-disabled-and-query-depth-capped-in-production) for details and migration steps.

For migrations off the pre-OCL naming and the `*V2` operations, see
[API Changelog & Migration](../api-reference/changelog.md).
