Partial update of the LabNft display metadata. Restricted to the OCL admin (LabNft owner + multisig signers).

Returns `UpdateLabNftMetadataResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `input` | `UpdateLabNftMetadataInput!` | Fields to change; omitted fields are left as they are. |
