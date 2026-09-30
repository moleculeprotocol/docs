---
description: >-
  The limits the Molecule APIs apply to a request: the per-credential cost
  budget, edge request counts, query depth and shape, list sizes, page sizes
  and storage.
icon: gauge
---

# ⏱️ Rate & Query Limits

Every request to the Molecule GraphQL APIs (the Labs API, the Tokenization API and the x402 Gateway's upstream calls) passes through the same set of limits. This page covers what each limit counts, and what you get back when you go over one.

For how a refused request looks on the wire and how to retry it, see [Errors](errors.md). This page doesn't repeat that.

{% hint style="info" %}
**Rate limits aren't enforced yet.** In every environment, cost budgets are charged and measured but nothing is refused, and the edge rules only count requests. Enforcement will be announced in the [API Changelog](changelog.md) before it starts. The query-shape, list-size and page-size limits further down **are** enforced today.
{% endhint %}

---

## At a glance

| Limit | Applies to | Value | Over it | Enforced |
| --- | --- | --- | --- | --- |
| [Cost budget](#cost-budget) | each machine credential, per 5-minute window | partner and personal credentials have different budgets, see [Budgets](#budgets) | `RATE_LIMITED` | No, measured only |
| [Edge request counts](#edge-request-counts) | each source IP and each credential, per 5 minutes | see [Edge request counts](#edge-request-counts) | raw HTTP `403` | No, counted only |
| [Query depth](#query-depth) | the whole query | 10 levels, production only | `QueryDepthLimitReached` | Yes |
| [Per-field selection](#per-field-selection-limits) | one field's own sub-selection | depth 6, 100 selections | `COMPLEXITY_LIMIT_EXCEEDED` | Yes |
| [List size](#lists-without-paging-arguments) | 11 list fields with no paging arguments | 1000 items; 50 for `Token.relations` and `Token.markets` | `COMPLEXITY_LIMIT_EXCEEDED` | Yes |
| [Page size and filters](#page-size-sorting-and-filters) | connection arguments | 100 per page, 3 sort keys, 100 ids, 200-character strings | `VALIDATION_FAILED` | Yes |
| [Storage](#storage) | each lab's data room | 5 GB | contact Molecule | Yes |
| [Concurrency](#concurrency) | none | no limit | `TIMEOUT` or `INTERNAL_ERROR` under load | — |

---

## Cost budget

Each machine credential (a `mol_<consumerId>_…` consumer credential) has a **cost budget per time window**. Every request's cost is charged against the credential's current window. Once the window's budget is spent, further requests in that window get [`RATE_LIMITED`](errors.md#rate-limited-requests) until the window ends.

The budget is a guardrail against accidents, such as a runaway loop, a retry storm or a leaked credential. It's not a usage quota. Budgets are set well above what integrations are seen to use, so a legitimate integration shouldn't ever hit one. If yours does, tell us: the budget is wrong, not your integration.

### How a request is priced

The cost of a request is worked out once, from the query document, before any resolver runs:

- **About one unit per selected field.** A small lookup costs a handful of units.
- **Multiplied through lists.** A field under a list is charged once per item, using the page size you asked for (for example `first` or `perPage`), or an assumed size where the list takes no paging argument. Nested lists multiply, so a deeply nested list query can cost millions of units.
- **Mutations cost ×10.** The same selection costs ten times more on a mutation than on a query.
- **At most 1,000,000,000 units per request.** No single request is charged more than that, so one pathological query can empty a window but never more than one.
- **Documents that can't be priced are charged the maximum.** That's a document over 100 KB of query text, or a selection set too large to walk. Documents the API rejects anyway, such as syntax errors or a missing operation, are free.
- **Introspection queries are free.** A query on `__schema` or `__type` isn't priced. Introspection is only available on staging.

Aliases don't make a query cheaper: fields are priced by their real name, not the alias.

To keep costs down, select only the fields you use, and ask for the page size you need rather than the largest one allowed.

### Budgets

| Credential | Budget per 5-minute window | Can be exempted |
| --- | --- | --- |
| Partner credential, issued by Molecule | 250,000,000 units by default; Molecule can set a different budget per credential | Yes |
| Personal credential (`mol_usr-…`), issued by you for yourself; staging only for now | 10,000,000 units | No |

These are the current defaults. They may be recalibrated before enforcement starts; any change will be announced in the [API Changelog](changelog.md).

A personal credential's budget is roughly one or two of the heaviest queries seen in production per window. If your integration needs more headroom than that, it's time to talk to us about a partner credential.

### What shares a budget

- **A budget is per credential, not per process.** Every process, container or agent using the same credential draws from the same window. If several of them share one credential, coordinate their backoff, or ask for one credential each.
- **Personal credentials share a budget per owner.** All the `mol_usr-…` credentials you issue draw from one window. Revoking a credential and issuing a new one doesn't reset it.
- **x402 payers share one budget.** Every call through the [x402 Gateway](x402-gateway.md) is made with the gateway's own partner credential, so all payers draw from the same window. That credential isn't exempt and runs on the default 250,000,000-unit budget. Every x402 call is a mutation, so each one is charged ×10. A `RATE_LIMITED` you get through the gateway can be caused by other payers' traffic, and it's still settled: see [Errors › Forwarded mutation](errors.md#forwarded-mutation).
- **Windows are fixed, not rolling.** Each window starts on a 5-minute boundary, and `retryAfterSeconds` in a denial is the time until the current one ends.

### What is charged

- **Personal credentials are charged on every request.**
- **Partner credentials may be charged on a sample.** A partner credential is charged either on every request or once per authorization-cache period (up to 5 minutes). Molecule sets which applies per credential. Sampled budgets are calibrated on the same sampled numbers, so plan as if every request counts.
- **Only authenticated machine requests are charged.** A request with a missing or invalid credential is refused before it's priced.
- **Retries are charged too.** A denied request is itself charged, and a retry before `retryAfterSeconds` can't succeed: it only spends requests, which also count against the [edge request counts](#edge-request-counts). See [Rate-limited requests](errors.md#rate-limited-requests) for the backoff to use.

### Signed-in sessions

Requests made with a signed-in Labs app session (a Privy `Bearer` token rather than a `mol_…` credential) have no cost budget and are never refused with `RATE_LIMITED`.

---

## Edge request counts

The API sits behind a web application firewall with two request-count rules, each over a 5-minute window:

| Rule | Counts requests per | Staging | Production |
| --- | --- | --- | --- |
| Per source IP | IP address | 100 | 2,000 |
| Per credential | `Authorization` header value | 50 | 6,000 |

These rules only **count** today; they block nothing. The production numbers are starting values, and they may change before any rule starts blocking. The staging numbers are much tighter, so a load test against staging is not a guide to production headroom.

A blocked request gets a raw HTTP `403` with no GraphQL body. It's a different thing from `RATE_LIMITED`; see [Failures before your operation runs](errors.md#failures-before-your-operation-runs).

---

## Query depth

**Production caps query depth at 10.** Scalar leaves count as a level, so `{ root { child { name } } }` is depth 3. A deeper query fails at execution time with `errorType: "QueryDepthLimitReached"` and partial data. This is a plain GraphQL error, without `errorInfo`.

Staging has no depth limit. It also has introspection enabled, which production doesn't (see [Getting the schema](getting-started/README.md#getting-the-schema)). Test your deepest queries against production-like limits before you ship: a query that works on staging can fail on production.

---

## Per-field selection limits

Separately from the whole-query depth, each field measures its **own** sub-selection, the part of the query below it. The token fields are the exception: `token`, `tokenById`, `tokens`, `Lab.tokens`, `linkToken` and `unlinkToken` aren't measured yet.

| Limit | Value |
| --- | --- |
| Depth, counting the field itself as level 1 | 6 |
| Selections, including `__typename` | 100 |

Over either one, the field fails with `COMPLEXITY_LIMIT_EXCEEDED` and `details.reason: "QUERY_SHAPE_LIMIT"`. `details.field` names the field that refused as a `Type.field` coordinate, such as `"Query.labs"`. `details.limit` says which limit (`depth` or `selections`), and `details.observed` and `details.allowed` give the numbers:

```json
{
  "errorType": "COMPLEXITY_LIMIT_EXCEEDED",
  "message": "Query has too many field selections for this field: 103 exceeds the limit of 100.",
  "errorInfo": {
    "requestId": "8f1e4c9a-2b7d-4e10-9c3a-5d6f7a8b9c0d",
    "retryable": false,
    "details": {
      "reason": "QUERY_SHAPE_LIMIT",
      "field": "Query.labActivity",
      "limit": "selections",
      "observed": 103,
      "allowed": 100
    }
  }
}
```

On a Labs API mutation, the refusal comes back in the mutation's `error` field instead, with `code: "COMPLEXITY_LIMIT_EXCEEDED"` and the same `details`. See [Failed mutation](errors.md#failed-mutation).

On union results, such as an activity feed, the branches add up: the selections under every `... on` branch count towards the same 100. Select only the fields you render from each branch.

A named fragment spread anywhere inside a `... on` branch of a union or interface result can't be measured, so it's refused with `details.reason: "QUERY_SHAPE_UNVERIFIABLE"`, `details.limit: "shape"` and no numbers. Inline the fragment's fields into the branch and retry. A selection too large to measure gets the same answer.

Retrying an unchanged query gives the same answer; neither reason is retryable.

---

## Lists without paging arguments

Eleven list fields return every item, take no `first` or `limit` argument, and have a size limit instead, stated in each field's description:

| Field | Limit |
| --- | --- |
| `Lab.members`, `LabRef.members`, `ListLabMembersResult.members` | 1000 |
| `DataRoom.files` | 1000 |
| `Announcement.attachments`, `SearchLabsAnnouncement.attachments` | 1000 |
| `OnChainEvent.events` | 1000 |
| `FileCategoriesAndTagsResult.data`, `FileCategory.tags` | 1000 |
| `Token.relations`, `Token.markets` | 50 |

Above its limit, a query that selects the list fails with `COMPLEXITY_LIMIT_EXCEEDED` and `details.reason: "RESULT_CARDINALITY_LIMIT"`. `details.field` names the list as a `Type.field` coordinate, and `details.limit` is its limit as a number, for example `1000`. Unlike the per-field selection limits, there's no `details.observed`. A shortened list is never returned as if it were complete.

`listLabMembers` is the one exception to "selects the list": it fails over the limit even when the query doesn't select `members`.

Which part of the response fails, the list field alone or the root field that returned it, is covered in [the changelog entry](changelog.md#lists-without-paging-arguments-now-fail-above-a-size-limit).

---

## Page size, sorting and filters

Connection fields (`labsConnection`, `tokens` and `Lab.tokens`, paged with `first` / `after`) validate their arguments before running:

| Argument | Limit |
| --- | --- |
| Page size (`first` or `last`) | 1 to 100 |
| `orderBy` | 1 to 3 distinct keys |
| An id list in a filter | 1 to 100 ids |
| A string in a filter (`eq`, `contains`) | 200 characters |

Over a limit, the query fails with `VALIDATION_FAILED`, and `details.field` names the argument. An id list that's empty or has more than 100 ids, or a filter string over 200 characters, also carries `details.reason: "FILTER_COMPLEXITY_LIMIT"`. Split the request into smaller ones.

---

## Storage

Each lab's data room can hold **5 GB** by default. Molecule can raise the limit for a lab on request. See [Files › Storage Limits](labs-api/files.md#storage-limits).

---

## Concurrency

There is no limit on how many requests you have in flight, and no number to stay under. The cost budget is the only throttle, and it counts per window, not per moment.

That doesn't make concurrency free. Requests in flight at the same time share database capacity, so running many at once makes each one slower. Past some point, requests start failing, and not all in the same way:

- **`TIMEOUT` with `details.reason: "DB_POOL_TIMEOUT"` or `"DB_CONNECT_TIMEOUT"`**: the request waited too long for a database connection. Usually that's backpressure: lower your concurrency, then retry with backoff. Retrying at the same width won't help. One heavy request can also cause it by itself, because each request gets a single connection and the queries behind its fields wait their turn on it. If it keeps happening at low concurrency, trim the selection.
- **`TIMEOUT` with `details.reason: "LAMBDA_TIMEOUT"`**: a single request was too large. Reduce its page size or selection.
- **`INTERNAL_ERROR`**: when the service is already running as many requests as it can, new ones are rejected before they run. That comes back as a masked `INTERNAL_ERROR`, not `TIMEOUT`, so a burst of them under load is backpressure too.

If you're walking every page of a list, for example to build an index or a sitemap:

1. **Use `labsConnection`, not `labs`.** It's answered in one query per page, where `labs` is much slower per page. Add `filter: { hasDataRoom: true }` if you only want labs that have a page.
2. **Select only what you render.** `totalCount`, the scoring fields (`weightedScore`, `scoreInterpretation`, `criterionScores`, `scoredAt`, `todos`), `members`, `ipnft` and `legalAgreementStatus` are fetched only when you select them; `totalCount` is the most expensive.
3. **Walk sequentially.** The next cursor arrives with the current page, and with (1) and (2) a full walk is usually fast without any parallelism.
4. **If you still need parallelism, keep it small.** Treat the first `TIMEOUT` or `INTERNAL_ERROR` as the ceiling you just found, and drop back to sequential.

---

## Staging and production

| Limit | Staging | Production |
| --- | --- | --- |
| Cost budget | same budgets, measured only | same budgets, measured only |
| Edge request counts | 100 per IP, 50 per credential, per 5 minutes; counted only | 2,000 per IP, 6,000 per credential, per 5 minutes; counted only |
| Query depth | no limit | 10 |
| Introspection | enabled | disabled |
| Per-field selection, list size, page size and filters | same | same |

Credentials are per environment: each one has its own budget in the environment that issued it.

---

## Related

- [Errors](errors.md) — what a `RATE_LIMITED` response looks like, and how to retry it
- [Authentication](authentication.md) — consumer credentials and the `Authorization` header
- [API Changelog & Migration](changelog.md) — the depth limit, list limits, and future enforcement announcements
- [Browse & Search](labs-api/browse-and-search.md) — the connection queries and their paging arguments

{% include "../.gitbook/includes/support.md" %}
