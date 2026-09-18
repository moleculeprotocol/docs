One decoded onchain event emitted by a Molecule contract.

| Name | Type | Description |
| --- | --- | --- |
| `id` | `ID!` | Cursor-stable event id, formatted `<blockNumber>:<logIndex>`. Pass the last entry's value as `cursor` to fetch the next page. |
| `chainId` | `Int!` | EVM chain id the event was emitted on (e.g. 8453 for Base). |
| `contractAddress` | `String!` | Address of the emitting contract, lowercase 0x-hex. |
| `contractName` | `String!` | Logical subsystem the event came from. One of `accessresolver`, `ocl`, `ipnft`, `ipt`, `bio-agent`. |
| `eventName` | `String!` | Name of the decoded Solidity event exactly as emitted (e.g. `OclIdentityCreated`, `RoleGranted`, `Transfer`). Raw per-event name, not the classified `OnChainEvent.type`. |
| `blockNumber` | `String!` | Block height of the event as a decimal string, since block numbers can exceed the range of `Int`. |
| `blockTimestamp` | `AWSDateTime!` | Timestamp of the block that included the event. An event ingested before its block metadata was available carries `1970-01-01T00:00:00.000Z` until it is reconciled. |
| `txHash` | `String!` | Hash of the transaction that emitted the event, 0x-prefixed. With `logIndex` it identifies the event uniquely. |
| `logIndex` | `Int!` | Position of the event's log within its block. |
| `args` | `AWSJSON!` | Decoded event arguments. BigInts are decimal strings; addresses are lowercased. |
