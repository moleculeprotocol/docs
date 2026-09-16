Semantic search across labs' files and announcements. Public. A blank `prompt` fails with `VALIDATION_FAILED`, and an upstream outage fails with `UPSTREAM_UNAVAILABLE` or `TIMEOUT` rather than returning an empty page.

Returns [`SearchLabsResult!`](/api-reference/types.md#searchlabsresult).

| Name | Type | Description |
| --- | --- | --- |
| `prompt` | `String!` | Natural-language search text. Truncated to 2000 characters. |
| `filters` | [`SearchLabsFilters`](/api-reference/types.md#searchlabsfilters) | Filters narrowing which labs, tags, categories, access levels and entry kinds are searched. |
| `page` | `Int` | Page number, 0-indexed; negative or non-integer values fall back to 0. |
| `perPage` | `Int` | Hits per page, to a maximum of 100; values outside 1-100 and non-integers fall back to 10. |
