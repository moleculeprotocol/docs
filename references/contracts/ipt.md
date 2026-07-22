---
icon: coin-front
---

# IPT

## IP Token (IPT) Contract Documentation

### Overview

An **IP Token (IPT)** is a fungible ERC-20 token representing a fractional claim tied to a Lab. Each tokenized Lab has exactly one IPT, deployed by the [OclTokenizer](tokenizer.md) as an EIP-1167 clone of the `LabToken` contract on Base. IPT holders gain economic exposure to the Lab and participate in its community — while control of the token's supply always follows the Lab's current owner.

### Contract Details

| Property     | Value                                                        |
| ------------ | ------------------------------------------------------------ |
| **Contract** | LabToken (EIP-1167 clone per Lab)                             |
| **Standard** | ERC-20 (Burnable)                                             |
| **Owner**    | The OclTokenizer, which enforces lab-controller rules         |
| **Decimals** | 18                                                            |
| **Chain**    | Base (8453) / Base Sepolia (84532)                            |
| **Solidity** | 0.8.33                                                        |
| **License**  | MIT                                                           |

Clone templates: Base mainnet [`0xd13a5D5ab80c15c344aA0eAa36f16ABae940dB0c`](https://basescan.org/address/0xd13a5D5ab80c15c344aA0eAa36f16ABae940dB0c), Base Sepolia [`0x8038d3B220C635b3B7A7B4bcba0928a2D5B0B8F6`](https://sepolia.basescan.org/address/0x8038d3B220C635b3B7A7B4bcba0928a2D5B0B8F6). Each Lab's IPT is a distinct clone with its own address — resolve it via `OclTokenizer.tokenized(labId)`.

### Naming & Metadata

The token name is derived automatically at tokenization — `Lab Tokens of Lab #<labId>` — and the symbol is chosen by the Lab's controller. Every IPT carries onchain metadata linking it to its origin:

```solidity
struct Metadata {
    uint256 labId;         // the LabNFT tokenId this IPT derives from
    address originalOwner; // the Lab controller at tokenization time
    string  s3Key;         // membership agreement document key
    bytes32 contentHash;   // SHA-256 of the agreement, binding terms to exact bytes
}
```

`uri()` returns contract-level metadata as a base64 data URL (ERC-1155-compatible format), so explorers and integrations can render the token's Lab context without an offchain service.

### Supply Model

* **Issuance** — new tokens are minted via `issue(receiver, amount)`, callable only by the tokenizer or the Lab's current controller. `totalIssued` tracks everything ever minted (it can exceed the live supply, since tokens are burnable).
* **Capping** — `cap()` permanently freezes issuance. It is irreversible and idempotent (calling it again simply re-emits `Capped`). Once capped, `issue` reverts `TokenCapped()` forever, giving holders supply certainty.
* **Burning** — any holder can `burn` their own tokens (or `burnFrom` with allowance), reducing live supply.
* **Transfers** — standard ERC-20 transfers; there are no onchain transfer restrictions.

Because the contract's owner is the OclTokenizer and controller checks resolve live against `LabNFT.ownerOf`, supply authority moves automatically with the Lab: transfer the LabNFT and the new owner controls `issue`/`cap` immediately.

### Functions

```solidity
// Mint new tokens (tokenizer or Lab controller only; reverts once capped)
function issue(address receiver, uint256 amount) external

// Permanently freeze issuance (tokenizer or Lab controller only)
function cap() external

// Burn tokens
function burn(uint256 amount) external
function burnFrom(address account, uint256 amount) external

// Reads
function metadata() external view returns (Metadata memory)
function totalIssued() external view returns (uint256)
function capped() external view returns (bool)
function uri() external view returns (string memory)
```

### Events & Errors

* **`Capped(uint256 atSupply)`** — issuance frozen at `totalIssued`
* Standard ERC-20 `Transfer` / `Approval`
* **`TokenCapped()`** — issuance attempted after capping
* **`MustControlLab()`** — caller is neither the tokenizer nor the Lab's controller

### Wrapped IPTs (Bring Your Own Token)

A Lab can attach a pre-existing ERC-20 as its IPT instead of minting a new one. The tokenizer wraps it in a **`WrappedLabToken`** — a read-only decorator that carries the Lab's `Metadata` and `uri()` while delegating economics to the underlying token. `issue` and `cap` on a wrapped IPT revert: supply of the underlying token remains governed by its own contract.

### Related Contracts

* [Tokenizer](tokenizer.md) — deploys and controls IPTs
* [Molecule Labs](../../core-infrastructure/onchain-lab.md) — the Lab the IPT is tied to

### Resources

* **Tokenization flow & API**: [Tokenization API](../../api-reference/tokenization-api.md)
* **Query existing IPTs**: [Data API](../../api-reference/data-api.md) (`ipts`, `markets`)
