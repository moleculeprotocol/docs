Snapshot of DID-linking state for an OCL. `status` is null until the first linking attempt reaches PENDING, and `linkedDidCount` counts the DIDs recorded as active onchain, so a count with a non-terminal `status` means the terminal transition has not landed yet.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte 0x-hex OCL id of the lab. |
| `status` | `DidLinkingStatus` | Current state of the linking state machine; null before the first attempt. |
| `userOpHash` | `String` | Hash of the user operation submitted for the most recent attempt; null until one has been submitted. |
| `txHash` | `String` | Hash of the transaction that included the user operation; null until it was mined. |
| `accountDid` | `String` | DID of the lab's smart account (the ERC-6551 account); null until known. |
| `dataRoomDid` | `String` | DID of the lab's data room; null until known. |
| `linkedDidCount` | `Int!` | Number of DIDs recorded as linked onchain for this lab (2 when both the account and data room DIDs are linked). |
| `attempts` | `Int!` | Number of linking attempts made so far (0 before the first). |
| `updatedAt` | `AWSDateTime` | When the linking state last changed; null before the first attempt. |
