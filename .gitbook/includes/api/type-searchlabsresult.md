Search results with pagination.

| Name | Type | Description |
| --- | --- | --- |
| `nodes` | `[SearchLabsHit!]!` | Hits on the requested page, most relevant first. An empty list means the search matched nothing or this page held only hits this API cannot yet render, and never stands in for a failure. |
| `totalCount` | `Int!` | Hits the search matched across all pages, counting matches rather than what this page could render, so it can exceed the length of `nodes`. Falls back to a lower bound when matches cannot be counted. |
| `pageInfo` | `PageInfo!` | Pagination information for the requested page. |
