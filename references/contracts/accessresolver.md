---
icon: gavel
---

# AccessResolver

## AccessResolver Overview

The `AccessResolver` contract is the on-chain authorization primitive for Onchain Labs. It answers two questions:

1. **"Is this wallet an authorized signer for a given IP-NFT or ERC-6551 Token Bound Account?"** — used by the file-encryption layer to gate decryption of confidential data-room files and by back-office flows that need to resolve Safe multisigs and Ownable contracts to their leaf EOAs.
2. **"What role does this wallet hold on a given lab, and is the grant still active?"** — the V3 role system (`ROLE_VIEWER`, `ROLE_CONTRIBUTOR`) with per-grant expiry and `isAgent` metadata, hierarchical (Owner > Contributor > Viewer), and administered per `oclId`.

See [Roles & Permissions](../../core-concepts/roles-and-permissions.md) for the product-level role model and [Data Privacy & Access](../../core-concepts/data/data-privacy-and-access.md) for how these predicates feed into the encryption / decryption pipeline.

### Contract Details

| Property | Value                                       |
| -------- | ------------------------------------------- |
| Contract | AccessResolver (V3)                         |
| Type     | UUPS Upgradeable Proxy                      |
| Solidity | 0.8.30                                      |
| Storage  | Preserves V1/V2 layout; V3 adds `_roles` map and `labNftContractAddress` |

#### Deployments

The **V3 role system runs only on Base and Base Sepolia**. The Ethereum Mainnet and Sepolia deployments are **v2** — they expose the signer predicates (`isAuthorizedSignerForIpnft`, `isAuthorizedSignerForTba`, …) but have **no role functions** (`grantRole` / `hasRole` / `revokeRole` / `getRole`).

* **Base (canonical chain — v3, roles live here)**
  * Address: `0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B`
  * [Verified on BaseScan](https://basescan.org/address/0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B)
* **Base Sepolia (v3)**
  * Address: `0x5493F472602C87318EA5Eff753cDD593bf9bF559`
  * [Verified on BaseScan](https://sepolia.basescan.org/address/0x5493F472602C87318EA5Eff753cDD593bf9bF559)
* **Ethereum Mainnet (v2 — signer predicates only)**
  * Address: `0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0`
  * [Verified on Etherscan](https://etherscan.io/address/0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0)
* **Sepolia Testnet (v2 — signer predicates only)**
  * Address: `0xd9b492fd34b1579C052b2EA25970178B3011Ce6B`
  * [Verified on Etherscan](https://sepolia.etherscan.io/address/0xd9b492fd34b1579C052b2EA25970178B3011Ce6B)

### How It Works

1. A caller (file-encryption layer, GraphQL resolver, UI) asks the `AccessResolver` whether a wallet is authorized for a given IP-NFT / TBA / lab role.
2. The resolver checks direct ownership and the ERC-6551 `isValidSigner` fast-path first, then walks the ownership graph — Safe `isOwner` → `Ownable.owner()` — recursively, up to depth 10.
3. For role checks it reads the per-lab `RoleGrant` struct, enforces the Owner > Contributor > Viewer hierarchy, and respects each grant's expiry timestamp.
4. Returns `true`/`false`. The resolver never mints, grants, or revokes encryption keys itself — it is read-only from the caller's perspective (except for the explicit `grantRole` / `revokeRole` entry points).

#### Signer Authorization (V1/V2)

*   **isAuthorizedSignerForIpnft**: Checks if an address is authorized for a specific IP-NFT, resolving Safe multisigs and Ownable wrappers recursively.

    ```solidity
    function isAuthorizedSignerForIpnft(address signer, uint256 ipnftId)
        external view returns (bool);
    ```
*   **isAuthorizedSignerForTba**: Determines if an address can act on behalf of an ERC-6551 Token Bound Account. Fast path uses `isValidSigner`; slow path resolves the TBA's bound NFT owner (handles Safe-held NFTs).

    ```solidity
    function isAuthorizedSignerForTba(address signer, address account)
        external view returns (bool);
    ```
*   **ownersOfIpnft**: Returns the deduplicated leaf (EOA) owners of an IP-NFT after recursively unwrapping Safe multisigs and Ownable smart accounts.

    ```solidity
    function ownersOfIpnft(uint256 ipnftId) external view returns (address[] memory);
    ```
*   **isApprovedLock**: Checks if a signer holds a locked token and is approved (used by locked-token-gated access conditions).

    ```solidity
    function isApprovedLock(address tokenAddress, address signer)
        external view returns (bool);
    ```

#### Role Management (V3)

The V3 role system adds hierarchical, per-lab roles (`ROLE_VIEWER = 1`, `ROLE_CONTRIBUTOR = 2`) administered per canonical `oclId`. `hasRole` is hierarchical: Contributor passes Viewer checks, and the Lab Owner (resolved via the OCL TBA) passes every check. See [Roles & Permissions](../../core-concepts/roles-and-permissions.md) for the full model and capability matrix.

```solidity
uint8 public constant ROLE_VIEWER = 1;
uint8 public constant ROLE_CONTRIBUTOR = 2;

struct RoleGrant {
    uint8  role;     // 0 = none, 1 = Viewer, 2 = Contributor
    uint64 expiry;   // 0 = permanent, >0 = unix timestamp
    bool   isAgent;  // true if grantee is an AI agent (metadata only)
}

function grantRole(bytes32 oclId, address account, uint8 role, uint64 expiry, bool isAgent) external;
function revokeRole(bytes32 oclId, address account) external;
function hasRole(bytes32 oclId, address account, uint8 role) external view returns (bool);
function getRole(bytes32 oclId, address account)
    external view returns (uint8 role, uint64 expiry, bool isAgent);
```

**`oclId` layout** (bytes32, MSB → LSB): version byte (`0x01`), namespace byte (`0x01` = EVM), 10 reserved / tokenId-high bytes, 20-byte TBA address. Every role entry point runs `_validateOclId`, which verifies the version / namespace bytes, that the TBA has code, that `LabNFT.accountOf(tokenId) == tba`, and that `IERC6551.token()` returns `(CANONICAL_CHAIN_ID = 8453, labNft, tokenId)`. Malformed identifiers revert with `InvalidOclId`.

**Chain scoping.** The role system exists only on the **Base and Base Sepolia (v3)** deployments — Ethereum Mainnet and Sepolia run v2, which has no role functions at all. Canonical lab state lives on Base: lab-owner self-administration (`grantRole` / `revokeRole` called by the NFT holder) works only there, because the reference ERC-6551 `owner()` returns `address(0)` off-canonical-chain.

**Global admin.** In addition to per-lab owners, the **contract owner (Molecule's protocol multisig)** is a global role admin: it can grant and revoke roles on any lab and passes every `hasRole` check. This is the operational escape hatch for support and recovery flows.

**Setup (owner-only).**

```solidity
function setLabNftContract(address labNftAddress) external; // onlyOwner
function setLockedTokenFactory(address factoryAddress) external; // onlyOwner
function initializeV3(address _labNftContractAddress) public; // onlyOwner, reinitializer(2)
```

#### Events

*   **RoleGranted(oclId, account, role, expiry, isAgent, grantedBy)** — emitted when a role is granted.
*   **RoleRevoked(oclId, account, role, revokedBy)** — emitted when a role is revoked. Revoking an account with no stored grant (`role == 0`) returns silently without emitting; revoking an expired-but-present grant still requires authorization and emits.
*   **Initialized(uint64 version)** — emitted when the contract is initialized or reinitialized.
*   **OwnershipTransferred(previousOwner, newOwner)** — emitted on contract-owner change.
*   **Upgraded(implementation)** — emitted when the UUPS implementation is upgraded.

#### Errors

| Error                                                  | Description                                                                   |
| ------------------------------------------------------ | ----------------------------------------------------------------------------- |
| `InvalidOclId(bytes32 oclId)`                          | Malformed `oclId` (bad version / namespace, no TBA code, or LabNFT mismatch). |
| `InvalidRole(uint8 role)`                              | Role must be `ROLE_VIEWER (1)` or `ROLE_CONTRIBUTOR (2)`.                     |
| `UnauthorizedRoleAdmin(bytes32 oclId, address, uint8)` | Caller lacks permission for the requested grant/revoke.                       |
| `OwnersOverflow(uint256)`                              | Owner-resolution exceeded `MAX_OWNERS (50)` or recursion depth 10.            |
| `OwnableUnauthorizedAccount(address)` *(inherited)*    | Caller is not the contract owner for owner-only functions.                    |
| `OwnableInvalidOwner(address)` *(inherited)*           | Invalid owner address provided.                                               |
| `UUPSUnauthorizedCallContext()` *(inherited)*          | Upgrade called in an incorrect context.                                       |

The first four are the contract's custom errors; the rest are inherited from OpenZeppelin.

### Integration Guide

The same `accessControlConditions` shape is reused across both Molecule's Onchain-Verified Envelope Encryption and the legacy Lit Protocol path — `AccessResolver` is the on-chain oracle either way. New integrations should drive encryption through the Labs API (`initiateCreateOrUpdateFile` / `decryptDataKey`); the Lit examples below remain valid for legacy files.

#### Use as an Access Control Condition (current)

For Onchain-Verified Envelope Encryption, attach an `accessControlConditions` array referencing this contract's predicates to the file's `encryptionMetadata`. The example below gates decryption on _LabNFT owner OR active Contributor OR active Viewer_ by OR'ing `isAuthorizedSignerForTba` with `hasRole(oclId, :userAddress, ROLE_VIEWER)` (hierarchy makes one role check cover Contributor + Viewer). Substitute `<accessresolver-address>` with the deployment matching the chain the backend evaluator targets — see [Deployments](#deployments) above:

```json
[
  {
    "conditionType": "evmContract",
    "contractAddress": "<accessresolver-address>",
    "chain": "base",
    "functionName": "isAuthorizedSignerForTba",
    "functionParams": [":userAddress", "0x<40hex-tba>"],
    "functionAbi": {
      "name": "isAuthorizedSignerForTba",
      "inputs": [
        { "name": "signer",  "type": "address" },
        { "name": "account", "type": "address" }
      ],
      "outputs": [{ "name": "", "type": "bool" }],
      "stateMutability": "view",
      "type": "function"
    },
    "returnValueTest": { "key": "", "comparator": "=", "value": "true" }
  },
  { "operator": "or" },
  {
    "conditionType": "evmContract",
    "contractAddress": "<accessresolver-address>",
    "chain": "base",
    "functionName": "hasRole",
    "functionParams": [
      "0x0101<20hex-tokenId><40hex-tba>",
      ":userAddress",
      "1"
    ],
    "functionAbi": {
      "name": "hasRole",
      "inputs": [
        { "name": "oclId",   "type": "bytes32" },
        { "name": "account", "type": "address" },
        { "name": "role",    "type": "uint8"   }
      ],
      "outputs": [{ "name": "", "type": "bool" }],
      "stateMutability": "view",
      "type": "function"
    },
    "returnValueTest": { "key": "", "comparator": "=", "value": "true" }
  }
]
```

`:userAddress` is substituted with the authenticated caller at evaluate time. See [Data Privacy & Access](../../core-concepts/data/data-privacy-and-access.md) for the full upload / decrypt flow, condition shape definitions, and evaluator behaviour.

#### With Lit Protocol _(legacy)_

Utilize `AccessResolver` as an `EvmContractCondition` in file encryption. Pass the condition directly to the Lit client.

```js
// AccessResolver as a Lit EvmContractCondition
const accessCondition = {
  contractAddress: '0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0', // AccessResolver
  chain: 'ethereum',
  functionName: 'isAuthorizedSignerForIpnft',
  functionParams: [':userAddress', '42'],
  functionAbi: {
    name: 'isAuthorizedSignerForIpnft',
    inputs: [
      { name: 'signer',  type: 'address' },
      { name: 'ipnftId', type: 'uint256' }
    ],
    outputs: [{ name: '', type: 'bool' }],
    stateMutability: 'view',
    type: 'function'
  },
  returnValueTest: { key: '', comparator: '=', value: 'true' }
};

// Encryption with Lit Protocol
const encrypted = await litClient.encrypt({
  accessControlConditions: [accessCondition],
  dataToEncrypt: fileBuffer
});
```

#### Direct Contract Call

Read a predicate directly with viem to check access outside of an encryption flow.

```js
import { createPublicClient, http } from 'viem';
import { mainnet } from 'viem/chains';

const client = createPublicClient({ chain: mainnet, transport: http() });

const isAuthorized = await client.readContract({
  address: '0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0', // AccessResolver
  abi: accessResolverAbi,
  functionName: 'isAuthorizedSignerForIpnft',
  args: [userAddress, 42n]
});
```

### Predicates Available as Access Control Conditions

Use any of the predicates below as the `functionName` of an `EvmContractCondition` pointing at this contract.

| Predicate                                                                | Gates                                                              | Role-aware |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------ | :--------: |
| `isAuthorizedSignerForIpnft(address signer, uint256 ipnftId)`            | Direct + recursive ownership of the IP-NFT (Safe / Ownable / TBA). |            |
| `isAuthorizedSignerForTba(address signer, address account)`              | Authorized signer of an ERC-6551 TBA, including its bound NFT owner. |            |
| `hasRole(bytes32 oclId, address account, uint8 role)`                    | Active, non-expired role grant on the lab; honours Owner > Contributor > Viewer hierarchy. |     ✓      |
| `isApprovedLock(address tokenAddress, address signer)`                   | Holds and is approved on a locked token (used by locked-token gating). |            |

The legacy Lit helper string IDs `ipnft_read` / `authorized_ipnft_signer` resolve to these same predicates internally and are retained for legacy Lit-encrypted files only — new integrations should write the `EvmContractCondition` shape directly.

### Security Considerations

* **Read-only nature**: The contract performs authorization checks but does not modify ownership or access.
* **Failure handling**: Access is denied if contract call fails.
* **Network verification**: Ensure querying on the correct network.

### Related Contracts

* IP-NFT: The contract queried for ownership details.
* Tokenizer: Converts IP-NFTs into IPTokens.

### Resources

* **ABI**: Available from the verified contract on [BaseScan](https://basescan.org/address/0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B) (source lives in the Molecule Labs contracts repository — contact the team for access)
* **Lit Protocol Documentation**: [Lit Protocol Developer Docs](https://developer.litprotocol.com/)
