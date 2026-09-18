Registers a data room for an onchain lab whose LabNft has already been minted. Requires the LabNft owner or an authorized signer for it. Fails with `INVALID_OCL_ID` when `oclId` is not a canonical 32-byte value.

Returns `CreateLabResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `input` | [`CreateLabInput!`](input-createlabinput.md) | Onchain lab to register. |
