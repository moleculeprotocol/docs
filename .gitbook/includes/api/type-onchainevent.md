One transaction's onchain activity classified into a single timeline entry. An OCL creation renders as one `New Onchain Lab created` entry rather than a burst of raw events, and its constituent events stay available in `events`.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` | Cursor-stable group id, formatted `<blockNumber>:<logIndex>` from the newest matching event of the transaction. Pass the last entry's value as `cursor` to fetch the next page. |
| `chainId` | `Int!` | EVM chain id the transaction was mined on (e.g. 8453 for Base). |
| `txHash` | `String!` | Hash of the transaction, 0x-prefixed. Every entry in `events` shares it. |
| `blockNumber` | `String!` | Block height of the newest matching event of the transaction, as a decimal string. |
| `blockTimestamp` | `AWSDateTime!` | Block timestamp of the transaction, from its latest event carrying block metadata. Falls back to `1970-01-01T00:00:00.000Z` only while every event is awaiting reconciliation. |
| `type` | `String!` | One of: OCL_CREATED, OCL_TOKENIZED, OCL_TRANSFERRED, OCL_DID_LINKED, ROLE_GRANTED, ROLE_REVOKED, ROLE_CHANGED, IPT_TOKENIZED, IPNFT_MINTED, IPNFT_TRANSFERRED, IPNFT_METADATA_UPDATED, OTHER. |
| `title` | `String!` | Human-readable title, e.g. `New Onchain Lab created`. For `OTHER` it is the raw event name. |
| `args` | `AWSJSON!` | Structured facts of the classified action, such as oclId, from/to, role and account. Addresses appear lowercased in full; titles use shortened forms. |
| `events` | `[RawOnChainEvent!]!` | Raw events of the transaction in ascending log order. Includes events that did not match the oclId or wallet filter, giving full transaction context. |
