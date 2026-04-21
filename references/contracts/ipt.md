---
description: >-
  The ERC-20 token that represents fractionalized ownership of an IP-NFT's
  intellectual property rights
icon: battery-bolt
---

# IPT

#### Overview

IPToken is an ERC-20 token that represents a transferable share of intellectual property rights derived from an IP-NFT. Each IP-NFT can be tokenized exactly once, producing a unique IPToken contract. The resulting tokens can be held, transferred, and traded — giving holders economic exposure, governance participation, and token-gated access to the Lab's data room.

IPToken contracts are not deployed individually. They are created as EIP-1167 minimal proxy clones by the Tokenizer contract, which means every IPToken shares the same implementation code but maintains independent state. There is no fixed deployment address for IPTokens — each one is unique and must be discovered through `Tokenizer.synthesized(ipnftId)` or via the subgraph.

#### Contract Details

<table><thead><tr><th width="286.29296875">Property</th><th>Value</th></tr></thead><tbody><tr><td>Contract</td><td>IPToken v1.3</td></tr><tr><td>Standard</td><td>ERC-20 (with Burnable)</td></tr><tr><td>Type</td><td>EIP-1167 Minimal Proxy Clone</td></tr><tr><td>Created By</td><td>Tokenizer</td></tr><tr><td>Owner</td><td>Tokenizer contract (enforces IP-NFT holdership)</td></tr><tr><td>Solidity</td><td>0.8.18</td></tr><tr><td>License</td><td>MIT</td></tr></tbody></table>

#### Discovering IPToken Addresses

IPTokens do not have fixed addresses. To find the IPToken for a given IP-NFT:

**On-chain:**

solidity

```solidity
IIPToken ipToken = Tokenizer.synthesized(ipnftId);
```

**Via subgraph:**

graphql

```graphql
{
  ipt(where: { ipnftId: "123" }) {
    id
    tokenContract
  }
}
```

#### How It Works

**Creation**

When an IP-NFT owner calls `Tokenizer.tokenizeIpnft()`, the Tokenizer deploys a new IPToken clone via `initialize()`. The token is configured with a name, symbol, and a Metadata struct that permanently links it to the originating IP-NFT ID, the original owner's address, and the IPFS CID of the governing legal agreement (FAM agreement).

**Supply Control**

The IP-NFT controller (the current owner of the underlying IP-NFT, resolved through the Tokenizer) can issue additional tokens at any time by calling `issue()` through the Tokenizer. Once `cap()` is called, the supply is permanently frozen — no new tokens can ever be minted. Capping is irreversible.

The `totalIssued` counter tracks all tokens ever minted and may differ from `totalSupply()` if tokens have been burned.

**Controller Model**

The IPToken's owner is always the Tokenizer contract, not the IP-NFT holder directly. Write operations (`issue` and `cap`) are gated by the `onlyTokenizerOrIPNFTController` modifier, which allows calls from either the Tokenizer itself or the current controller of the associated IP-NFT (as resolved by `Tokenizer.controllerOf(ipnftId)`). This means IPToken control automatically transfers when the IP-NFT changes hands — there is no separate ownership transfer needed.

**Burning**

IPToken inherits from ERC20BurnableUpgradeable. Any holder can burn their own tokens via `burn(amount)` or `burnFrom(account, amount)` with approval. Burning permanently removes tokens from circulation and reduces `totalSupply()`, but `totalIssued` remains unchanged.

#### Functions

**Write Functions**

**issue** Mints new tokens to a specified address. Only callable by the Tokenizer or the current IP-NFT controller. Reverts if the token has been capped.

solidity

```solidity
function issue(address receiver, uint256 amount) external
```

Parameters:

* `receiver` (address): The address to receive newly minted tokens.
* `amount` (uint256): The number of tokens to mint (18 decimals).

**cap** Permanently freezes the token supply. After calling, no new tokens can ever be issued. Irreversible.

solidity

```solidity
function cap() external
```

Emits: `Capped(uint256 atSupply)`

**burn** (inherited from ERC20Burnable) Burns tokens from the caller's balance.

solidity

```solidity
function burn(uint256 amount) public
```

**burnFrom** (inherited from ERC20Burnable) Burns tokens from another account (requires approval).

solidity

```solidity
function burnFrom(address account, uint256 amount) public
```

**transfer / transferFrom / approve** (standard ERC-20) Standard ERC-20 transfer and approval functions. No restrictions — IPTokens are freely transferable.

**Read Functions**

**metadata** Returns the Metadata struct linking this token to its originating IP-NFT.

solidity

```solidity
function metadata() external view returns (Metadata memory)
```

Returns:

* `ipnftId` (uint256): The ID of the parent IP-NFT.
* `originalOwner` (address): The IP-NFT owner at the time of tokenization.
* `agreementCid` (string): IPFS CID of the legal agreement governing this token.

**totalIssued** Returns the total number of tokens ever minted. This may exceed `totalSupply()` if tokens have been burned.

solidity

```solidity
function totalIssued() external view returns (uint256)
```

**capped** Returns whether the token supply has been permanently frozen.

solidity

```solidity
function capped() external view returns (bool)
```

**uri** Returns a base64-encoded data URL containing ERC-1155-compatible JSON metadata, including the IP-NFT ID, agreement CID, original owner, token contract address, and current supply.

solidity

```solidity
function uri() external view returns (string memory)
```

**totalSupply / balanceOf / allowance / name / symbol / decimals** (standard ERC-20) Standard ERC-20 read functions. Decimals is always 18.

#### Metadata Struct

solidity

```solidity
struct Metadata {
    uint256 ipnftId;       // The ID of the parent IP-NFT
    address originalOwner; // The IP-NFT owner at tokenization time
    string agreementCid;   // IPFS CID of the governing legal agreement
}
```

#### Events

| Event                                                     | Description                                                                                       |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `Capped(uint256 atSupply)`                                | Emitted when the token supply is permanently frozen. `atSupply` is the final `totalIssued` value. |
| `Transfer(address from, address to, uint256 value)`       | Standard ERC-20 transfer event (includes mints from `address(0)` and burns to `address(0)`).      |
| `Approval(address owner, address spender, uint256 value)` | Standard ERC-20 approval event.                                                                   |

#### Errors

<table><thead><tr><th width="211.9140625">Error</th><th>Description</th></tr></thead><tbody><tr><td><code>TokenCapped()</code></td><td>Attempted to call <code>issue()</code> after the token has been capped.</td></tr><tr><td><code>MustControlIpnft()</code></td><td>Caller is not the Tokenizer contract or the current IP-NFT controller.</td></tr></tbody></table>

#### Security Considerations

**Controller-gated supply.** Only the Tokenizer or the current IP-NFT holder can issue or cap tokens. This is enforced on-chain by the `onlyTokenizerOrIPNFTController` modifier.

**Irreversible capping.** Once `cap()` is called, the supply is permanently frozen. There is no mechanism to uncap a token. This protects holders from unexpected dilution.

**Automatic controller transfer.** IPToken control follows IP-NFT ownership. Transferring the IP-NFT to a new address immediately grants that address the ability to issue and cap tokens. No separate IPToken ownership transfer is needed.

**Clone architecture.** IPTokens are EIP-1167 minimal proxy clones. The implementation is shared, reducing deployment gas, but each clone has independent storage. The implementation contract cannot be called directly (initializers are disabled in the constructor).

**Freely transferable.** IPTokens have no transfer restrictions. They can be sent to any address, traded on DEXes, or used in DeFi protocols.

**Audited.** IPToken is covered by the Pashov security review.

#### Related Contracts

<table><thead><tr><th width="207.390625">Contract</th><th>Relationship</th></tr></thead><tbody><tr><td>Tokenizer</td><td>Creates IPToken clones and enforces IP-NFT holdership rules for <code>issue</code> and <code>cap</code>.</td></tr><tr><td>IPNFT</td><td>The ERC-721 token that IPTokens are derived from. IP-NFT ownership determines who controls the IPToken.</td></tr><tr><td>AccessResolver</td><td>Uses IP-NFT ownership to resolve file access permissions for IPToken holders.</td></tr><tr><td>TimelockedToken</td><td>Can lock IPTokens for a configurable duration (used in vesting and crowdsales).</td></tr></tbody></table>

#### Resources

<table><thead><tr><th width="206.4453125">Resource</th><th>Link</th></tr></thead><tbody><tr><td>Source Code</td><td><a href="https://github.com/moleculeprotocol/IPNFT/blob/main/src/IPToken.sol">IPToken.sol</a></td></tr><tr><td>Interface</td><td><a href="https://github.com/moleculeprotocol/IPNFT/blob/main/src/IIPToken.sol">IIPToken.sol</a></td></tr><tr><td>ABI</td><td>Available in <code>@moleculexyz/common/abis</code></td></tr><tr><td>Audit</td><td><a href="https://github.com/pashov/audits/blob/master/solo/pdf/IPNFT-security-review.pdf">Pashov Security Review</a></td></tr><tr><td>Subgraph</td><td>Query IPTokens via the Molecule Subgraph</td></tr></tbody></table>
