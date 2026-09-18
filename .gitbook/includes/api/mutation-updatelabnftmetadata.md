Partial update of the LabNft display metadata. Restricted to the OCL admin (LabNft owner + multisig signers).

Returns [`UpdateLabNftMetadataResult!`](/api-reference/types.md#updatelabnftmetadataresult).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `input` | [`UpdateLabNftMetadataInput!`](/api-reference/types.md#updatelabnftmetadatainput) | Fields to change; omitted fields are left as they are. |
