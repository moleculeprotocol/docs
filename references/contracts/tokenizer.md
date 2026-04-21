---
icon: computer-classic
---

# Tokenizer

## Tokenizer Protocol Documentation

### Overview

The Tokenizer contract converts IP-NFTs (ERC-721) into fungible IPTokens (ERC-20), enabling fractional ownership of intellectual property. Each IP-NFT can be tokenized only once, and the resulting IPToken supply is managed by the IP-NFT owner.

### Contract Details

| Property | Value                     |
| -------- | ------------------------- |
| Contract | Tokenizer v1.4            |
| Type     | UUPS Upgradeable Proxy    |
| Pattern  | Factory (EIP-1167 Clones) |
| Solidity | 0.8.18                    |
| License  | MIT                       |

### Deployments

* **Ethereum Mainnet**
  * Address: [0x58EB89C69CB389DBef0c130C6296ee271b82f436](https://etherscan.io/address/0x58EB89C69CB389DBef0c130C6296ee271b82f436)
  * Start Block: 17464363
* **Sepolia Testnet**
  * Address: [0xca63411FF5187431028d003eD74B57531408d2F9](https://sepolia.etherscan.io/address/0xca63411FF5187431028d003eD74B57531408d2F9)
  * Start Block: 5300776

### How It Works

#### Tokenization Flow

Only IP-NFT owners can tokenize their IP-NFTs. Tokenization creates a new IPToken contract via minimal proxy (clone). Owners receive the initial token supply and can issue more tokens until calling `cap()`.

#### Key Mechanics

* **Tokenize IP-NFT**: Creates a new IPToken.
* **Attach IPT**: Attaches an existing ERC-20 as IPToken.
* **Issue Tokens**: Issues more tokens; only uncapped tokens are issuable.
* **Cap Tokens**: Permanently caps the IPToken supply.

### Functions

#### Write Functions

**`tokenizeIpnft`**

Creates a new IPToken from an IP-NFT.

**Parameters:**

* `ipnftId` (uint256): IP-NFT token ID
* `tokenAmount` (uint256): Initial token supply
* `tokenSymbol` (string): Token symbol
* `agreementCid` (string): IPFS CID of the legal agreement
* `signedAgreement` (bytes): Signed legal agreement

**Returns:** `IPToken` — Newly created IPToken contract address

**`attachIpt`**

Attaches an existing ERC-20 token as the IPToken.

**Parameters:**

* `ipnftId` (uint256): IP-NFT token ID
* `agreementCid` (string): IPFS CID of the agreement
* `signedAgreement` (bytes): Signed legal agreement
* `tokenContract` (address): ERC-20 token address

**Returns:** `IIPToken` — Wrapped IPToken for the existing ERC-20

**`issue`**

Issues additional IPTokens.

**Parameters:**

* `ipToken` (IPToken): IPToken address
* `amount` (uint256): Token amount to issue
* `receiver` (address): Receiver address

**`cap`**

Caps the IPToken supply, preventing further issuance.

**Parameters:**

* `ipToken` (IPToken): IPToken address

#### Read Functions

**`synthesized`**

Returns the IPToken address for a given IP-NFT ID.

**Parameters:**

* `ipnftId` (uint256): IP-NFT token ID

**Returns:** `IIPToken` — IPToken address or zero address if un-tokenized

**`controllerOf`**

Returns the current owner address of the IP-NFT.

**Parameters:**

* `ipnftId` (uint256): IP-NFT token ID

### Events

* **TokensCreated**: Emitted when a new IPToken is created.
* **TokenWrapped**: Emitted when an existing ERC-20 is wrapped as an IPToken.

### Security Considerations

* Each IP-NFT can only be tokenized once.
* IPToken control transfers automatically with IP-NFT ownership.
* Capping is irreversible and prevents further token issuance.

### Resources

* **Source Code:** [Tokenizer.sol](https://github.com/moleculeprotocol/IPNFT/blob/main/src/Tokenizer.sol)
* **ABI:** Available in `@moleculexyz/common/abis`
* **Audit:** [Audit Report](https://github.com/moleculeprotocol/IPNFT/tree/main/audit)

For further details and integration guidelines, consult the [subgraph](https://github.com/moleculeprotocol/IPNFT/blob/main/subgraph) and other related documentation.
