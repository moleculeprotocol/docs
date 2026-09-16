Onchain labs, with optional membership filtering and page-numbered pagination. Superseded by `labsConnection` for new consumers, which adds cursor pagination, sorting and richer filters.

Returns [`LabsResult!`](/api-reference/types.md#labsresult).

| Name | Type | Description |
| --- | --- | --- |
| `walletAddress` | `String` | Wallet address to filter labs by membership. When provided, returns labs where this wallet holds any active role (owner/contributor/viewer) unless narrowed by `role`. |
| `role` | [`LabMemberRole`](/api-reference/types.md#labmemberrole) | Role filter, applied only when `walletAddress` is set, restricting the result to labs where that wallet holds this role. Omit to include every lab the wallet belongs to under any role. |
| `page` | `Int` | Page number (0-indexed). |
| `perPage` | `Int` | Number of items per page (max 100). |
