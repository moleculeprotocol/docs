---
description: >-
  How delegated access to an Onchain Lab works — role hierarchy, capability
  matrix, expiry, agent flag, and the on-chain resolver.
icon: users-gear
---

# Roles & Permissions

## Why Roles Exist

An Onchain Lab's NFT holder is its sole ultimate controller — transferring the LabNFT transfers the entire project. In practice, most research projects need to delegate day-to-day data-room work (uploading files, posting announcements, decrypting confidential research) to collaborators and AI agents without surrendering ownership.

The role system lets a Lab owner grant scoped, expiring access to specific wallets — human or agent — while keeping ownership, treasury control, and the ability to revoke access at any time. Roles are enforced on-chain by the `AccessResolver` contract and honoured by every downstream system (GraphQL API, file encryption, UI) that checks them.

## Role Hierarchy

Every lab has three effective roles, ordered from most to least privileged.

<table><thead><tr><th width="170">Role</th><th width="200">How it's held</th><th>Scope</th></tr></thead><tbody><tr><td><strong>Owner</strong></td><td>Holder of the LabNFT, resolved through the Lab's ERC-6551 Token Bound Account (TBA). Safe multisigs holding the NFT are resolved recursively through their signers.</td><td>Full control; passes every permission check. Can transfer the LabNFT.</td></tr><tr><td><strong>Contributor</strong></td><td>Explicit on-chain grant: <code>ROLE_CONTRIBUTOR = 2</code>.</td><td>Full data-room access, can grant/revoke Viewers. Cannot add other Contributors or transfer the NFT.</td></tr><tr><td><strong>Viewer</strong></td><td>Explicit on-chain grant: <code>ROLE_VIEWER = 1</code>.</td><td>Read-only. Can decrypt confidential files and read data-room contents.</td></tr></tbody></table>

The `hasRole` check is hierarchical: a Contributor automatically passes Viewer checks, and the Owner passes every check.

## Capability Matrix

| Capability                                  | Owner | Contributor | Viewer |
| ------------------------------------------- | :---: | :---------: | :----: |
| View public data-room files                 |   ✓   |      ✓      |   ✓    |
| Decrypt confidential data-room files        |   ✓   |      ✓      |   ✓    |
| Upload / update / delete data-room files    |   ✓   |      ✓      |        |
| Create announcements                        |   ✓   |      ✓      |        |
| Grant / revoke Viewer role                  |   ✓   |      ✓      |        |
| Grant / revoke Contributor role             |   ✓   |             |        |
| Manage project owners (add/remove)          |   ✓   |             |        |
| Transfer the LabNFT                         |   ✓   |             |        |
| Authorize / install modules on the Lab      |   ✓   |             |        |

A Contributor cannot "downgrade" another Contributor to Viewer — downgrades are treated as an admin-level action and rejected for non-Owner callers.

## Grants: Expiry & Agent Flag

Each grant is an on-chain record with three fields:

```solidity
struct RoleGrant {
    uint8  role;     // 0 = none, 1 = Viewer, 2 = Contributor
    uint64 expiry;   // 0 = permanent, >0 = unix timestamp
    bool   isAgent;  // true if the grantee is an AI agent
}
```

- **Expiry** — A non-zero `expiry` makes the grant auto-expire. Expired grants still exist in storage (so `getRole` returns them for UI purposes) but are inactive: `hasRole` returns `false` once `block.timestamp >= expiry`. Expired grantees must be re-granted to regain access.
- **`isAgent`** — Purely informational metadata. It does **not** change on-chain authorization, but downstream systems (the members list, the data-room UI, the agent-auth flow) surface it to clearly distinguish AI-agent session keys from human team members.

A Lab owner granting access to an agent should set `isAgent = true` and a short `expiry` — typically the agent's session-key lifetime. When the session expires, the agent must request a new grant before it can continue to decrypt files or post announcements.

## Scope: Per-Lab, Not Per-File

Roles are scoped to a **lab** — identified by the canonical `oclId` (a packed 32-byte identifier combining version, namespace, tokenId, and the TBA address). There is no per-data-room or per-file role; file-level access is enforced by the Onchain-Verified Envelope Encryption layer, which ultimately resolves back to the same `AccessResolver` predicates (`hasRole`, `isAuthorizedSignerForTba`, `isAuthorizedSignerForIpnft`) when evaluating a decryption request.

For the concrete `accessControlConditions` JSON that turns a role grant into file-level decryption rights, see [Worked Example: Encrypt for Owner OR Contributor OR Viewer](data/data-privacy-and-access.md#worked-example-encrypt-for-owner-or-contributor-or-viewer).

## Chain Scoping

`AccessResolver` is deployed on Base, Mainnet, and Sepolia, but canonical lab state lives on Base. Lab-owner self-administration works only on Base: the ERC-6551 reference implementation returns `address(0)` for `owner()` when `block.chainid` doesn't match the chain the OCL was CREATE2-salted for, so lab NFT holders on Mainnet or Sepolia must call `grantRole` / `revokeRole` through the Base deployment.

Every `grantRole` / `revokeRole` / `hasRole` / `getRole` call runs `_validateOclId`, which verifies the `oclId`'s version byte, namespace byte, TBA code, LabNFT binding, and canonical-chain metadata. Malformed identifiers revert with `InvalidOclId`.

## On-Chain Interface

```solidity
function grantRole(bytes32 oclId, address account, uint8 role, uint64 expiry, bool isAgent) external;
function revokeRole(bytes32 oclId, address account) external;

function hasRole(bytes32 oclId, address account, uint8 role) external view returns (bool);
function getRole(bytes32 oclId, address account)
    external view returns (uint8 role, uint64 expiry, bool isAgent);
```

### Who may call what

| Action                                     | Owner | Active Contributor |
| ------------------------------------------ | :---: | :----------------: |
| `grantRole(… Contributor)`                 |   ✓   |                    |
| `grantRole(… Viewer)` (fresh / same level) |   ✓   |         ✓          |
| `grantRole(…)` that downgrades a role      |   ✓   |                    |
| `revokeRole(…)` on a Contributor           |   ✓   |                    |
| `revokeRole(…)` on a Viewer                |   ✓   |         ✓          |

Revokes on accounts with no active grant return silently without emitting an event, to prevent unauthorised callers from spamming `RoleRevoked` logs.

### Events

```solidity
event RoleGranted(
    bytes32 indexed oclId,
    address indexed account,
    uint8   indexed role,
    uint64  expiry,
    bool    isAgent,
    address grantedBy
);

event RoleRevoked(
    bytes32 indexed oclId,
    address indexed account,
    uint8   indexed role,
    address revokedBy
);
```

Use these events to reconstruct the team-members list for a lab off-chain; the on-chain storage is a sparse `mapping(oclId => mapping(account => RoleGrant))` and cannot be enumerated without event indexing.

### Errors

- `InvalidOclId(bytes32 oclId)` — malformed identifier or LabNFT binding mismatch.
- `InvalidRole(uint8 role)` — role must be `ROLE_VIEWER (1)` or `ROLE_CONTRIBUTOR (2)`.
- `UnauthorizedRoleAdmin(bytes32 oclId, address caller, uint8 role)` — caller lacks permission for the requested grant/revoke.

## See Also

- [AccessResolver contract reference](../references/contracts/accessresolver.md) — full ABI, deployments, signer-authorization predicates (`isAuthorizedSignerForIpnft`, `isAuthorizedSignerForTba`).
- [Data Privacy & Access](data/data-privacy-and-access.md) — how role checks feed into file encryption / decryption.
- [On-Chain Lab](onchain-lab.md) — how `oclId` is derived and why ownership resolves through the TBA.
