---
icon: book
---

# Contracts

Molecule Protocol consists of smart contracts deployed across Ethereum Mainnet and Base L2. These contracts enable the creation, tokenization, and trading of intellectual property assets.

#### Ethereum/Base Mainnet

| Network          | Contract       | Address                                    | Verified URL                                                                         | Function                     |
| ---------------- | -------------- | ------------------------------------------ | ------------------------------------------------------------------------------------ | ---------------------------- |
| Ethereum Mainnet | IPNFT          | 0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1 | [Etherscan](https://etherscan.io/address/0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1) | IP-NFT minting and ownership |
| Ethereum Mainnet | Tokenizer      | 0x58EB89C69CB389DBef0c130C6296ee271b82f436 | [Etherscan](https://etherscan.io/address/0x58EB89C69CB389DBef0c130C6296ee271b82f436) | IPToken creation factory     |
| Ethereum Mainnet | CrowdSale      | 0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2 | [Etherscan](https://etherscan.io/address/0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2) | Token sales                  |
| Ethereum Mainnet | SchmackoSwap   | 0xc09b8577c762b5e97a7d640f242e1d9bfaa7eb9d | [Etherscan](https://etherscan.io/address/0xc09b8577c762b5e97a7d640f242e1d9bfaa7eb9d) | IP-NFT trading               |
| Ethereum Mainnet | AccessResolver | 0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0 | [Etherscan](https://etherscan.io/address/0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0) | File access control          |
| Base             | LaunchFactory  | 0xD915e2bA3E22352344de245aC227EcaC0776a1A4 | [BaseScan](https://basescan.org/address/0xD915e2bA3E22352344de245aC227EcaC0776a1A4)  | Catalyst token launches      |

#### Sepolia Testnet

<table><thead><tr><th>Contract</th><th width="297.61328125">Address</th><th>Verified Link</th></tr></thead><tbody><tr><td>IPNFT</td><td>0x152B444e60C526fe4434C721561a077269FcF61a</td><td><a href="https://sepolia.etherscan.io/address/0x152B444e60C526fe4434C721561a077269FcF61a">View on Etherscan</a></td></tr><tr><td>Tokenizer</td><td>0xca63411FF5187431028d003eD74B57531408d2F9</td><td><a href="https://sepolia.etherscan.io/address/0xca63411FF5187431028d003eD74B57531408d2F9">View on Etherscan</a></td></tr><tr><td>CrowdSale</td><td>0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037</td><td><a href="https://sepolia.etherscan.io/address/0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037">View on Etherscan</a></td></tr><tr><td>SchmackoSwap</td><td>0x9e4c638e703d0Af3a3B9eb488dE79A16d402698f</td><td><a href="https://sepolia.etherscan.io/address/0x9e4c638e703d0Af3a3B9eb488dE79A16d402698f">View on Etherscan</a></td></tr></tbody></table>

### Upgrade Pattern

IPNFT and Tokenizer contracts use the **UUPS (Universal Upgradeable Proxy Standard)** pattern:

* Proxy contracts hold state and delegate calls to implementation contracts
* Only the contract owner can authorize upgrades
* Contract addresses remain stable across upgrades

CrowdSale and SchmackoSwap are **not upgradeable** - they are deployed as standard contracts.

### IPToken Cloning

IPTokens are deployed using the **EIP-1167 Minimal Proxy** pattern:

* Each IP-NFT tokenization creates a new IPToken clone
* Clones share implementation code but have independent state
* No fixed deployment address - each IPToken has a unique address
* Query `Tokenizer.synthesized(ipnftId)` or use the indexer to find IPToken addresses

### Security Considerations

* Core contracts (IPNFT, CrowdSale, Tokenizer, TimelockedToken) have been audited by pashov
* Admin functions are protected by `onlyOwner` access control
* See individual contract pages for specific security notes
