---
description: Consumer-visible changes to the Labs API, newest first.
icon: flask
---

# Labs API release notes

Changes to the [Labs API](../api-reference/labs-api/README.md) that affect integrations. Versions not
listed shipped nothing consumer-visible.

## 1.0.14

_Released 2026-08-04_

### Breaking changes

#### `isSuccess` removed; errors now use a unified contract

Every Labs API operation previously reported failure through an `isSuccess: Boolean!` field on its
result type. That field is **gone**, and error reporting is split by operation class.

**Queries now throw.** A failed query arrives in the top-level GraphQL `errors[]` array with
`errorType` set to a catalogue code — `UNAUTHENTICATED`, `UNAUTHORIZED`, `NOT_FOUND`,
`VALIDATION_FAILED`, `CONFLICT`, `FAILED_PRECONDITION`, `COMPLEXITY_LIMIT_EXCEEDED`, `RATE_LIMITED`,
`TIMEOUT`, `UPSTREAM_UNAVAILABLE` or `INTERNAL_ERROR`. The entry also carries
`errorInfo { requestId, retryable, details }`. Branch on `errorType`, never on message text.

The `isSuccess` and `error` fields were removed from the query result types `ActivitiesResult`,
`FileCategoriesAndTagsResult`, `ListLabMembersResult`, `DidLinkStatusResult` and
`LegalAgreementTemplateResult`. **Selecting them now fails GraphQL validation**, so every operation
document that names them must be updated.

**Mutations return errors in band.** Each mutation's `*Result` carries
`error: ApiError { code, message, requestId, retryable, details }`, and **success means
`error == null`**.

```diff
  mutation {
    createAnnouncement(input: { ... }) {
-     isSuccess
+     error { code message requestId retryable details }
      announcement { id }
    }
  }
```

**Migration:**

- Replace every `isSuccess` selection. On mutations, test `error == null`. On queries, remove the
  selection and handle the top-level `errors[]` array instead.
- Classify failures by `errorType` (queries) or `error.code` (mutations) rather than by matching
  message strings. Unexpected failures are now masked behind generic catalogue text, so message
  sniffing that used to work will not.
- The specific pre-cutover cause is preserved under the `reason` key of `details`, an AWSJSON string,
  if you need to distinguish cases the catalogue code merges.

#### Silent degradation on backend failure is gone

`labs` and `searchLabs` no longer return an empty page when the backend fails, and `dataRoomFile` no
longer returns `null` on backend failure — `null` now strictly means the file does not exist.
`listLabMembers` on an unknown lab now throws `NOT_FOUND`.

**Migration:** if your UI treated an empty list or a `null` file as a loading or error state, it can
now trust those as real results, and must handle GraphQL errors explicitly instead.

### Added

- `ApiError` — the shared mutation error type described above, with `code`, `message`, `requestId`,
  `retryable` and `details`.
- `errorInfo` on thrown query errors, carrying the `requestId` correlation id. This replaces the
  interim `"… (requestId: <uuid>)"` message suffix, which has been removed.

---

_Earlier releases predate this section. For migrations off the pre-OCL naming and the `*V2`
operations, see [API Changelog & Migration](../api-reference/changelog.md)._
