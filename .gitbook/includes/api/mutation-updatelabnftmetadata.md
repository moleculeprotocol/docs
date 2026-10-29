---
title: mutation-updatelabnftmetadata
---

Partial update of the LabNft display metadata. Restricted to lab owners (see `LabMemberRole.OWNER`), which for a nested lab includes the owners of the lab at the top of its chain. Changing `name` re-derives `shortname`, which changes the lab's public URL; the previous slug still resolves, and clients should route to `lab.shortname` from the result.

Returns [`UpdateLabNftMetadataResult!`](../../../api-reference/types.md#updatelabnftmetadataresult).

| Name    | Type                                                                                      | Description                                                    |
| ------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `oclId` | `String!`                                                                                 | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `input` | [`UpdateLabNftMetadataInput!`](../../../api-reference/types.md#updatelabnftmetadatainput) | Fields to change; omitted fields are left as they are.         |
