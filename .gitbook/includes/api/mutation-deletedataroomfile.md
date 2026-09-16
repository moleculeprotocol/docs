Deletes a file from the lab data room.

Returns [`DeleteDataRoomFileResult`](/api-reference/types.md#deletedataroomfileresult).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `path` | `String!` | Path within the data room. Paths under `agreements/` are reserved for signed legal agreements and cannot be deleted here. |
| `changeBy` | `String` | Deprecated and ignored: the change is attributed to the authenticated caller. Removed after 2027-01-01. |
