---
icon: transformer-bolt
---

# IP-NFT

## IP-NFT Contract Documentation

### Overview

The **IP-NFT** contract is an ERC-721 NFT that represents onchain intellectual property ownership. Each IP-NFT is a unique token that enables trading, fractionalization into IPTokens, and use as collateral for funding.&#x20;

### Contract Details

* **Contract**: IPNFT v2.5.1
* **Standard**: ERC-721 (with URI Storage, Burnable)
* **Type**: UUPS Upgradeable Proxy
* **Solidity**: 0.8.18
* **License**: MIT

### Deployments

* **Ethereum Mainnet**: [0xcaD88677CA8...](https://etherscan.io/address/0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1) (Start Block: 17463429)
* **Sepolia Testnet**: [0x152B444e60C5...](https://sepolia.etherscan.io/address/0x152B444e60C526fe4434C721561a077269FcF61a) (Start Block: 5300057)

### How It Works

#### IP-NFT Lifecycle

1. **RESERVE**: `reserve()`
2. **MINT**: `mintReservation()`
3. **USE**:
   * Transfer/Trade
   * Tokenize (IPT)
   * Grant Access
   * Sell via Swap

#### Minting Paths

* **Reserve + Mint**: Call `reserve()`, then `mintReservation()`.
* **POI Mint (Direct)**: Use `mintReservation()` with Proof of Idea hash.

### Functions

#### Write Functions

**reserve**

Reserves a new token ID.

```solidity
function reserve() external whenNotPaused returns (uint256 reservationId)
```

**mintReservation**

Mints an IP-NFT using a reserved ID or POI hash.

{% code overflow="wrap" %}
```solidity
function mintReservation(address to, uint256 reservationId, string calldata _tokenURI, string calldata _symbol, bytes calldata authorization) external payable whenNotPaused returns (uint256)
```
{% endcode %}

**grantReadAccess** _(legacy)_

Grants time-limited read access to encrypted files. Deprecated as a data-access path — current data-room access is managed via Lab roles on the [AccessResolver](accessresolver.md); only files with legacy access conditions still evaluate this.

```solidity
function grantReadAccess(address reader, uint256 tokenId, uint256 until) external
```

**amendMetadata**

Updates the metadata URI of an IP-NFT.

{% code overflow="wrap" %}
```solidity
function amendMetadata(uint256 tokenId, string calldata _newTokenURI, bytes calldata authorization) external
```
{% endcode %}

#### Read Functions

**canRead** _(legacy)_

Checks if an address has read access. Evaluated only for files with legacy access conditions — see the note on `grantReadAccess` above.

```solidity
function canRead(address reader, uint256 tokenId) external view returns (bool)
```

**symbol**

Returns the symbol of an IP-NFT.

```solidity
function symbol(uint256 tokenId) external view returns (string memory)
```

**tokenURI**

Returns the metadata URI.

```solidity
function tokenURI(uint256 tokenId) public view returns (string memory)
```

### Events

* **Reserved**: New token ID reserved
* **IPNFTMinted**: IP-NFT minted
* **ReadAccessGranted**: Read access granted
* **AuthorizerUpdated**: Mint authorizer contract changed (admin)

### Errors

* `NotOwningReservation(uint256 id)`: Caller doesn't own the reservation.
* `Unauthorized()`: Caller not authorized.
* `InsufficientBalance()`: Caller doesn't own the IP-NFT.
* `BadDuration()`: Expiration in the past.
* `MintingFeeTooLow()`: Less than 0.001 ETH sent.
* `ToZeroAddress()`: Cannot transfer to zero address.

### Security Considerations

* **Symbolic Fee**: 0.001 ETH minting fee.
* **Authorization**: Required for all mints.
* **Read Access Expiration**: Time-limited.
* **Pausable**: Contract operations can be paused.
* **UUPS Upgradeable**: Owner-upgradable implementation.

### Constants

* **SYMBOLIC\_MINT\_FEE**: 0.001 ETH
* **MAX\_RESERVATION\_ID**: 2^128 - 1

### Related Contracts

* [Tokenizer](tokenizer.md): Fractionalize IP-NFTs into IPTokens.
* [SchmackoSwap](./): Trade IP-NFTs (see the address table).
* [AccessResolver](accessresolver.md): File access integration.
* [CrowdSale](crowdsale.md): Fundraise via IPTokens.

### Resources

* **Source Code**: [GitHub](https://github.com/moleculeprotocol/IPNFT/blob/main/src/IPNFT.sol)
* **ABI**: Generated from the contract source in the [IPNFT repository](https://github.com/moleculeprotocol/IPNFT)
* **Audit**: [Audit Report](https://github.com/moleculeprotocol/IPNFT/tree/main/audit)
