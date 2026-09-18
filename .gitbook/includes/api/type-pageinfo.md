Page-based pagination information. Pages are 0-indexed.

| Name | Type | Description |
| --- | --- | --- |
| `hasNextPage` | `Boolean!` |  |
| `hasPreviousPage` | `Boolean!` | Whether a page before the current one exists (true for any page but the first). |
| `currentPage` | `Int!` | Page number, 0-indexed. |
| `totalPages` | `Int!` | Total pages; 0 when there are no results. |
