Onchain role of a lab member. Roles are granted and revoked onchain through the lab's AccessResolver contract (`grantRole(oclId, account, role, expiry, isAgent)`, role 1 = VIEWER, 2 = CONTRIBUTOR; OWNER is whoever holds the LabNft). This API reflects those grants and cannot change them.

| Value | Description |
| --- | --- |
| `OWNER` | Holds the lab's LabNft; full control of the lab and its data room. |
| `CONTRIBUTOR` | May write to the data room (upload files, post announcements). |
| `VIEWER` | Read-only membership: may read the data room, including confidential files, but not write to it. |
