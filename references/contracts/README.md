---
icon: book
---

# Contracts

Molecule Protocol consists of smart contracts deployed across Ethereum Mainnet and Base L2. These contracts enable the creation, tokenization, and trading of intellectual property assets.

#### Ethereum/Base Mainnet

| Network          | Contract       | Address                                    | Verified URL                                                                         | Function                     |
| ---------------- | -------------- | ------------------------------------------ | ------------------------------------------------------------------------------------ | ---------------------------- |
| Ethereum Mainnet | IPNFT          | 0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1 | [Etherscan](https://etherscan.io/address/0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1) | IP-NFT minting and ownership |
| Ethereum Mainnet | CrowdSale      | 0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2 | [Etherscan](https://etherscan.io/address/0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2) | Token sales                  |
| Ethereum Mainnet | SchmackoSwap   | 0xc09b8577c762b5e97a7d640f242e1d9bfaa7eb9d | [Etherscan](https://etherscan.io/address/0xc09b8577c762b5e97a7d640f242e1d9bfaa7eb9d) | IP-NFT trading               |
| Ethereum Mainnet | AccessResolver | 0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0 | [Etherscan](https://etherscan.io/address/0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0) | File access control (v2 — signer predicates only; the role system lives on Base) |
| Base             | LaunchFactory  | 0xD915e2bA3E22352344de245aC227EcaC0776a1A4 | [BaseScan](https://basescan.org/address/0xD915e2bA3E22352344de245aC227EcaC0776a1A4)  | Catalyst token launches      |

#### Base Mainnet — Molecule Labs core (v0.1.0)

The Lab smart-account stack is deployed on Base (chain ID 8453), the canonical chain for Molecule Labs:

| Contract               | Address                                    | Verified URL                                                                        | Function                                    |
| ---------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------- |
| OnChainLabFactory      | 0xECdF4f05384056507485C90aeAb0a83268760D6E | [BaseScan](https://basescan.org/address/0xECdF4f05384056507485C90aeAb0a83268760D6E) | Lab account creation (mint & create)        |
| LabNFT (proxy)         | 0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92 | [BaseScan](https://basescan.org/address/0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92) | Lab ownership NFT (ERC-721)                 |
| ERC7484Registry        | 0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5 | [BaseScan](https://basescan.org/address/0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5) | Module attestation registry                 |
| RootValidator          | 0xb31d39ECc0cb26478E258C8f7e9C906115f494f6 | [BaseScan](https://basescan.org/address/0xb31d39ECc0cb26478E258C8f7e9C906115f494f6) | Default signature validator                 |
| MoleculeOclDidRegistry (proxy) | 0x6cd3Cf3c34a18Bf48F90590c3a57708F175b2eE3 | [BaseScan](https://basescan.org/address/0x6cd3Cf3c34a18Bf48F90590c3a57708F175b2eE3) | DID linking for Labs                |
| AccessResolver (v3)    | 0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B | [BaseScan](https://basescan.org/address/0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B) | Roles & file access control                 |
| OclTokenizer (proxy)   | 0x62F532C3f563D974deEc103AAb8cC597f4f9c84E | [BaseScan](https://basescan.org/address/0x62F532C3f563D974deEc103AAb8cC597f4f9c84E) | Lab tokenization (IPT factory)              |
| OclTermsPermissioner   | 0x125A12C880934826c80A54ce216B6c1F542603eE | [BaseScan](https://basescan.org/address/0x125A12C880934826c80A54ce216B6c1F542603eE) | Membership-agreement signature verification |

#### Sepolia Testnet

<table><thead><tr><th>Contract</th><th width="297.61328125">Address</th><th>Verified Link</th></tr></thead><tbody><tr><td>IPNFT</td><td>0x152B444e60C526fe4434C721561a077269FcF61a</td><td><a href="https://sepolia.etherscan.io/address/0x152B444e60C526fe4434C721561a077269FcF61a">View on Etherscan</a></td></tr><tr><td>CrowdSale</td><td>0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037</td><td><a href="https://sepolia.etherscan.io/address/0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037">View on Etherscan</a></td></tr><tr><td>SchmackoSwap</td><td>0x9e4c638e703d0Af3a3B9eb488dE79A16d402698f</td><td><a href="https://sepolia.etherscan.io/address/0x9e4c638e703d0Af3a3B9eb488dE79A16d402698f">View on Etherscan</a></td></tr></tbody></table>

#### Base Sepolia Testnet

| Contract            | Address                                    | Verified Link                                                                                |
| ------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------- |
| AccessResolver (v3) | 0x5493F472602C87318EA5Eff753cDD593bf9bF559 | [View on BaseScan](https://sepolia.basescan.org/address/0x5493F472602C87318EA5Eff753cDD593bf9bF559) |
| OclTokenizer (proxy) | 0xEe19e0Db8a7e59538710FAF6ed3ab655BCfCdB24 | [View on BaseScan](https://sepolia.basescan.org/address/0xEe19e0Db8a7e59538710FAF6ed3ab655BCfCdB24) |
| OclTermsPermissioner | 0x2196e7181393b8045F34b66f571F1566922Aa4dB | [View on BaseScan](https://sepolia.basescan.org/address/0x2196e7181393b8045F34b66f571F1566922Aa4dB) |

### Upgrade Pattern

The IPNFT and OclTokenizer contracts use the **UUPS (Universal Upgradeable Proxy Standard)** pattern:

* Proxy contracts hold state and delegate calls to implementation contracts
* Only the contract owner can authorize upgrades
* Contract addresses remain stable across upgrades

CrowdSale and SchmackoSwap are **not upgradeable** - they are deployed as standard contracts.

### IP Token Cloning

IP Tokens (IPTs) are deployed using the **EIP-1167 Minimal Proxy** pattern:

* Each Lab tokenization creates a new `LabToken` clone
* Clones share implementation code but have independent state
* No fixed deployment address - each IPT has a unique address
* Query `OclTokenizer.tokenized(labId)` or use the indexer to find IPT addresses

### Security Considerations

* Legacy core contracts (IPNFT, CrowdSale, TimelockedToken) have been audited by pashov
* The Molecule Labs core (OnChainLab account stack) was audited by Cyfrin in 2026 — see [Audits](../../security/audits.md)
* Admin functions are protected by `onlyOwner` access control
* See individual contract pages for specific security notes
