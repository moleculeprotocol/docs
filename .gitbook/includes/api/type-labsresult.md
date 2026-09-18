Result type for paginated labs queries.

| Name | Type | Description |
| --- | --- | --- |
| `nodes` | `[LabRef!]!` | Labs on the current page. An empty list means the page is genuinely empty and never stands in for a failure, which fails the request with `UPSTREAM_UNAVAILABLE` or `TIMEOUT` instead. |
| `totalCount` | `Int!` | Labs matching the query, ignoring pagination; `0` means nothing matched, never a failure. On the `walletAddress` form, labs whose identifier cannot be read are excluded from this count and `nodes`. |
| `pageInfo` | `PageInfo!` |  |
