---
description: Consumer-visible changes to the Labs API, newest first.
icon: flask
---

# Labs API release notes

Changes to the [Labs API](../api-reference/labs-api/README.md) that affect integrations. Versions not
listed shipped nothing consumer-visible.

## 2.0.1

_Released 2026-08-18_

### Breaking changes

#### GraphQL introspection disabled and query depth capped in production

The production endpoint no longer serves `__schema` / `__type` introspection queries — they return a validation error. `__typename` still resolves. Selection-set depth is also capped at 10 in production; a query beyond that fails at execution time with `errorType: "QueryDepthLimitReached"` rather than the usual error shape. See [API Changelog & Migration](../api-reference/changelog.md#graphql-introspection-disabled-and-query-depth-capped-in-production) for details and migration steps.

For migrations off the pre-OCL naming and the `*V2` operations, see
[API Changelog & Migration](../api-reference/changelog.md).
