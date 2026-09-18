Where a membership record came from. `ONCHAIN_EVENT` and `MULTISIG_RESOLUTION` derive from canonical owner state, `ACCESS_CONTRACT` is the legacy V2 IPNFT-auth contract, and `ACCESS_RESOLVER_EVENT` records stream from AccessResolver V3 role events and may have an expiry.

| Value | Description |
| --- | --- |
| `ONCHAIN_EVENT` | Derived from the LabNft ownership events onchain. |
| `MULTISIG_RESOLUTION` | Derived by resolving a multisig owner to its signers. |
| `ACCESS_CONTRACT` | Granted through the legacy V2 IPNFT access contract. |
| `ACCESS_RESOLVER_EVENT` | Granted by an AccessResolver V3 RoleGranted event; the only source whose grants can carry an `expiry`. |
