---
icon: arrows-split-up-and-left
---

# Tokenizer

## OclTokenizer Contract Documentation

### Overview

The `OclTokenizer` turns a Lab into a liquid asset: it deploys one fractional ERC-20 **IP Token (IPT)** per Lab, gated by a signed membership agreement. The caller must be the Lab's controller — the current LabNFT owner — and each Lab can be tokenized exactly once. It also supports attaching a pre-existing ERC-20 ("bring your own token") as the Lab's IPT via a read-only wrapper.

### Contract Details

| Property     | Value                                             |
| ------------ | ------------------------------------------------- |
| **Contract** | OclTokenizer (UUPS proxy) — deploys `LabToken` clones |
| **Type**     | UUPS Upgradeable Proxy + EIP-1167 clone factory   |
| **Chain**    | Base (8453) / Base Sepolia (84532)                |
| **Solidity** | 0.8.33                                            |
| **License**  | MIT                                               |

### Deployments

**Base Mainnet:**

| Contract                      | Address                                                                                                                       |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| OclTokenizer (proxy)          | [`0x62F532C3f563D974deEc103AAb8cC597f4f9c84E`](https://basescan.org/address/0x62F532C3f563D974deEc103AAb8cC597f4f9c84E)         |
| OclTermsPermissioner          | [`0x125A12C880934826c80A54ce216B6c1F542603eE`](https://basescan.org/address/0x125A12C880934826c80A54ce216B6c1F542603eE)         |
| LabToken (clone template)     | [`0xd13a5D5ab80c15c344aA0eAa36f16ABae940dB0c`](https://basescan.org/address/0xd13a5D5ab80c15c344aA0eAa36f16ABae940dB0c)         |
| WrappedLabToken (clone template) | [`0x48Ab68bB775e4AC50eA0a9939dBcc71ae4A69817`](https://basescan.org/address/0x48Ab68bB775e4AC50eA0a9939dBcc71ae4A69817)      |

**Base Sepolia:**

| Contract                      | Address                                                                                                                              |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| OclTokenizer (proxy)          | [`0xEe19e0Db8a7e59538710FAF6ed3ab655BCfCdB24`](https://sepolia.basescan.org/address/0xEe19e0Db8a7e59538710FAF6ed3ab655BCfCdB24)         |
| OclTermsPermissioner          | [`0x2196e7181393b8045F34b66f571F1566922Aa4dB`](https://sepolia.basescan.org/address/0x2196e7181393b8045F34b66f571F1566922Aa4dB)         |
| LabToken (clone template)     | [`0x8038d3B220C635b3B7A7B4bcba0928a2D5B0B8F6`](https://sepolia.basescan.org/address/0x8038d3B220C635b3B7A7B4bcba0928a2D5B0B8F6)         |
| WrappedLabToken (clone template) | [`0x1Ba1Bd7d8824d6e7DB4dDF15aa41E00cE841abE2`](https://sepolia.basescan.org/address/0x1Ba1Bd7d8824d6e7DB4dDF15aa41E00cE841abE2)      |

### How It Works

Tokenizing a Lab is a single transaction, authorized by two things:

1. **Lab control** — `controllerOf(labId)` resolves live to `LabNFT.ownerOf(labId)`. Only the Lab's current controller can tokenize, issue, or cap.
2. **A signed membership agreement** — the caller signs the exact terms text for the agreement (identified by its S3 `s3Key` and SHA-256 `contentHash`) with a plain `personal_sign` (EIP-191) signature. The configured `OclTermsPermissioner` reconstructs the terms onchain via `specificTermsV1()` and verifies the signature — ECDSA first, then ERC-1271 for smart-account signers.

On success, the tokenizer deploys an EIP-1167 clone of the `LabToken` template, mints the initial supply to the caller, records it in the `tokenized(labId)` registry, and emits `TokensCreated`. The token name is derived automatically as `Lab Tokens of Lab #<labId>`; the caller chooses only the symbol. A second tokenization attempt for the same Lab reverts.

The Lab's assets are never taken into custody — the IPT is a fractional claim token associated with the Lab, and the LabNFT stays exactly where it is.

### Functions

#### Write Functions

**`tokenize`**

Deploys the Lab's IP Token and mints the initial supply to the caller.

{% code overflow="wrap" %}
```solidity
function tokenize(uint256 labId, uint256 amount, string calldata symbol, string calldata s3Key, bytes32 contentHash, bytes calldata signature) external returns (LabToken token)
```
{% endcode %}

**`attachToken`**

"Bring your own token": registers a pre-existing ERC-20 (must have code and ≤ 18 decimals) as the Lab's IPT by wrapping it in a read-only `WrappedLabToken` that carries the Lab's metadata. `issue`/`cap` on a wrapped token revert.

{% code overflow="wrap" %}
```solidity
function attachToken(uint256 labId, string calldata s3Key, bytes32 contentHash, bytes calldata signature, IERC20Metadata tokenContract) external returns (ILabToken)
```
{% endcode %}

**`issue`**

Mints additional supply of a Lab's IPT. Controller-only; reverts once the token is capped.

```solidity
function issue(LabToken labToken, uint256 amount, address receiver) external
```

**`cap`**

Permanently freezes issuance for a Lab's IPT. Controller-only, irreversible.

```solidity
function cap(LabToken labToken) external
```

#### Read Functions

* **`controllerOf(uint256 labId)`** — the Lab's current controller (`LabNFT.ownerOf(labId)`)
* **`tokenized(uint256 labId)`** — the Lab's IPT contract, or the zero address if not tokenized
* **`labNft()`**, **`permissioner()`**, **`labTokenImplementation()`**, **`wrappedLabTokenImplementation()`** — current configuration

#### Admin Functions (owner-only)

`setPermissioner`, `setLabTokenImplementation`, `setWrappedLabTokenImplementation` — configuration updates by the owner multisig. Ownership transfers are two-step (`Ownable2Step`) with an explicit `cancelTransferOwnership`, and `renounceOwnership` is permanently disabled.

### Events

* **TokensCreated**(labId, oclId, tokenContract, emitter, amount, s3Key, contentHash, name, symbol) — a Lab was tokenized
* **TokenWrapped**(underlyingToken, wrappedLabToken, labId) — an existing ERC-20 was attached
* **PermissionerUpdated / LabTokenImplementationUpdated / WrappedLabTokenImplementationUpdated** — admin configuration changes

### Errors

* `MustControlLab()` — caller is not the Lab's current controller
* `AlreadyTokenized()` — the Lab already has an IPT
* `InvalidTokenContract()` / `InvalidTokenDecimals()` — attached token has no code or more than 18 decimals
* `InvalidS3Key()` — empty agreement key
* `ZeroAddress()`, `LabTokenNotControlledByTokenizer()`, `RenounceOwnershipDisabled()`

### Security Considerations

* **One token per Lab** — the `tokenized` registry is committed before external calls (checks-effects-interactions), and re-tokenization reverts.
* **Live controller checks** — authority follows the LabNFT: if the Lab changes hands, the new owner controls issuance and capping immediately.
* **Terms binding** — the signed agreement is bound to the exact document bytes via its SHA-256 `contentHash`, and the terms text is reconstructed onchain, so the signature can't be replayed against a different document.
* **Hardened ownership** — two-step transfers, renounce disabled, administered by a Molecule multisig.

### Related Contracts

* [IPT](ipt.md) — the ERC-20 IP Token this factory deploys
* [Molecule Labs](../../core-infrastructure/onchain-lab.md) — the Lab primitive being tokenized

### Resources

* **Tokenization flow & API**: [Tokenization API](../../api-reference/tokenization-api.md)
* **ABI**: Available from the verified contract on [BaseScan](https://basescan.org/address/0x62F532C3f563D974deEc103AAb8cC597f4f9c84E)
