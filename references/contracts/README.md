---
icon: network-wired
---

# Supported Networks & Contracts

Molecule Labs runs on **Base**. Base mainnet is the canonical chain, where Lab state, LabNFT ownership and role grants live. Base Sepolia hosts the [sandbox](../../api-reference/sandbox.md), the testnet copy of production for development. The legacy IP-NFT contracts on Ethereum are listed at the bottom for reference only.

### Networks

| Network      | Chain ID | CAIP-2         | viem chain    | Environment                        | Explorer                                       |
| ------------ | -------- | -------------- | ------------- | ---------------------------------- | ---------------------------------------------- |
| Base         | `8453`   | `eip155:8453`  | `base`        | Production (canonical chain)       | [basescan.org](https://basescan.org)           |
| Base Sepolia | `84532`  | `eip155:84532` | `baseSepolia` | Sandbox (testnet)                  | [sepolia.basescan.org](https://sepolia.basescan.org) |

The API environments map one-to-one onto these networks: the **sandbox** GraphQL endpoint and x402 gateway target Base Sepolia, and **production** targets Base. Credentials are per environment. To switch a script from one to the other, see [Running in Production](../../api-reference/getting-started/README.md#running-in-production).

#### Base Mainnet — Molecule Labs core (v0.1.0)

The Lab smart-account stack deployed on Base (chain ID `8453`):

| Contract                         | Address                                    | Verified URL                                                                        | Function                                    |
| -------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------- |
| OnChainLabFactory                | 0xECdF4f05384056507485C90aeAb0a83268760D6E | [BaseScan](https://basescan.org/address/0xECdF4f05384056507485C90aeAb0a83268760D6E) | Lab account creation (mint & create)        |
| LabNFT (proxy)                   | 0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92 | [BaseScan](https://basescan.org/address/0x9F96027eeAFb9ad5F2b5d7043B36Ee96B2EeBE92) | Lab ownership NFT (ERC-721)                 |
| ERC7484Registry                  | 0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5 | [BaseScan](https://basescan.org/address/0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5) | Module attestation registry                 |
| RootValidator                    | 0xb31d39ECc0cb26478E258C8f7e9C906115f494f6 | [BaseScan](https://basescan.org/address/0xb31d39ECc0cb26478E258C8f7e9C906115f494f6) | Default signature validator                 |
| MoleculeOclDidRegistry (proxy)   | 0x6cd3Cf3c34a18Bf48F90590c3a57708F175b2eE3 | [BaseScan](https://basescan.org/address/0x6cd3Cf3c34a18Bf48F90590c3a57708F175b2eE3) | DID linking for Labs                        |
| OdfCoAttestVerifier              | 0x5a7a22bbad7B8B3c91EdB7FC2Af2DE5de9060D8B | [BaseScan](https://basescan.org/address/0x5a7a22bbad7B8B3c91EdB7FC2Af2DE5de9060D8B) | `did:odf` co-attestation verifier           |
| AccessResolver (v3)              | 0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B | [BaseScan](https://basescan.org/address/0x89a14Be8f7824d4775053Edad0f2fA2d6767b72B) | Roles & file access control                 |
| OclTokenizer (proxy)             | 0x62F532C3f563D974deEc103AAb8cC597f4f9c84E | [BaseScan](https://basescan.org/address/0x62F532C3f563D974deEc103AAb8cC597f4f9c84E) | Lab tokenization (IPT factory)              |
| OclTermsPermissioner             | 0x125A12C880934826c80A54ce216B6c1F542603eE | [BaseScan](https://basescan.org/address/0x125A12C880934826c80A54ce216B6c1F542603eE) | Membership-agreement signature verification |
| LabToken (clone template)        | 0xd13a5D5ab80c15c344aA0eAa36f16ABae940dB0c | [BaseScan](https://basescan.org/address/0xd13a5D5ab80c15c344aA0eAa36f16ABae940dB0c) | IPT implementation (EIP-1167 template)      |
| WrappedLabToken (clone template) | 0x48Ab68bB775e4AC50eA0a9939dBcc71ae4A69817 | [BaseScan](https://basescan.org/address/0x48Ab68bB775e4AC50eA0a9939dBcc71ae4A69817) | Bring-your-own-token IPT wrapper            |

#### Base Sepolia Testnet (sandbox) — Molecule Labs core (v0.1.0)

The same stack and the same contract versions on Base Sepolia (chain ID `84532`). The sandbox API and every [Getting Started](../../api-reference/getting-started/README.md) tutorial run against these addresses:

| Contract                         | Address                                    | Verified URL                                                                                | Function                                    |
| -------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------- | ------------------------------------------- |
| OnChainLabFactory                | 0x66CCDDbe7d5EF968d0EFD8F6f7B2A3Ee527825DB | [BaseScan](https://sepolia.basescan.org/address/0x66CCDDbe7d5EF968d0EFD8F6f7B2A3Ee527825DB) | Lab account creation (mint & create)        |
| LabNFT (proxy)                   | 0x0FCACD29bE871688EeF853e9fBc6DCEA78343a14 | [BaseScan](https://sepolia.basescan.org/address/0x0FCACD29bE871688EeF853e9fBc6DCEA78343a14) | Lab ownership NFT (ERC-721)                 |
| ERC7484Registry                  | 0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5 | [BaseScan](https://sepolia.basescan.org/address/0x1Ab5Ba4300613F7346835b2BE7E2D10Ce6125eF5) | Module attestation registry                 |
| RootValidator                    | 0xb31d39ECc0cb26478E258C8f7e9C906115f494f6 | [BaseScan](https://sepolia.basescan.org/address/0xb31d39ECc0cb26478E258C8f7e9C906115f494f6) | Default signature validator                 |
| MoleculeOclDidRegistry (proxy)   | 0xFD478b192eA86d38899a0D45a85E0b141A12aE87 | [BaseScan](https://sepolia.basescan.org/address/0xFD478b192eA86d38899a0D45a85E0b141A12aE87) | DID linking for Labs                        |
| OdfCoAttestVerifier              | 0x0400F5F61B49F8C3E5C9fF72CC83D73375300205 | [BaseScan](https://sepolia.basescan.org/address/0x0400F5F61B49F8C3E5C9fF72CC83D73375300205) | `did:odf` co-attestation verifier           |
| AccessResolver (v3)              | 0xE4Dc90D9C5a9331046aFC2Cd569D9f055f3d90fe | [BaseScan](https://sepolia.basescan.org/address/0xE4Dc90D9C5a9331046aFC2Cd569D9f055f3d90fe) | Roles & file access control                 |
| OclTokenizer (proxy)             | 0x70F9A9E28C58f78049768F503CCd90E6f6e26BcD | [BaseScan](https://sepolia.basescan.org/address/0x70F9A9E28C58f78049768F503CCd90E6f6e26BcD) | Lab tokenization (IPT factory)              |
| OclTermsPermissioner             | 0x7E72A30F92652da825CB3458DFcD7C71eE0EA010 | [BaseScan](https://sepolia.basescan.org/address/0x7E72A30F92652da825CB3458DFcD7C71eE0EA010) | Membership-agreement signature verification |
| LabToken (clone template)        | 0xBeA1A862cC19fD2c0Fe2c52EE63DbeA0766A48bE | [BaseScan](https://sepolia.basescan.org/address/0xBeA1A862cC19fD2c0Fe2c52EE63DbeA0766A48bE) | IPT implementation (EIP-1167 template)      |
| WrappedLabToken (clone template) | 0xbB1fc39750b4bD664896B790f7C3A81f9Bb4D31f | [BaseScan](https://sepolia.basescan.org/address/0xbB1fc39750b4bD664896B790f7C3A81f9Bb4D31f) | Bring-your-own-token IPT wrapper            |

Need testnet funds? Get Base Sepolia ETH from a [Base Sepolia faucet](https://docs.base.org/base-chain/tools/network-faucets). You also need testnet USDC from the [Circle faucet](https://faucet.circle.com/), but only if you pay through the [x402 gateway](../../api-reference/x402-gateway.md).

### Contract details

* [IPT](ipt.md) — the per-Lab ERC-20 IP Token
* [Tokenizer](tokenizer.md) — `OclTokenizer`, which deploys and controls IPTs
* [AccessResolver](accessresolver.md) — roles and file-access predicates, including the v2 deployments on Ethereum

### Legacy IP-NFT contracts (unmaintained)

These predate Molecule Labs and are no longer maintained. New integrations should target the Base deployments above.

#### Ethereum Mainnet (chain ID `1`)

| Network          | Contract       | Address                                    | Verified URL                                                                         | Function                     |
| ---------------- | -------------- | ------------------------------------------ | ------------------------------------------------------------------------------------ | ---------------------------- |
| Ethereum Mainnet | IPNFT          | 0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1 | [Etherscan](https://etherscan.io/address/0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1) | IP-NFT ownership             |
| Ethereum Mainnet | CrowdSale      | 0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2 | [Etherscan](https://etherscan.io/address/0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2) | Token sales                  |
| Ethereum Mainnet | AccessResolver | 0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0 | [Etherscan](https://etherscan.io/address/0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0) | File access control (v2 — signer predicates only; the role system lives on Base) |

#### Ethereum Sepolia (chain ID `11155111`)

<table><thead><tr><th>Contract</th><th width="297.61328125">Address</th><th>Verified Link</th></tr></thead><tbody><tr><td>IPNFT</td><td>0x152B444e60C526fe4434C721561a077269FcF61a</td><td><a href="https://sepolia.etherscan.io/address/0x152B444e60C526fe4434C721561a077269FcF61a">View on Etherscan</a></td></tr><tr><td>CrowdSale</td><td>0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037</td><td><a href="https://sepolia.etherscan.io/address/0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037">View on Etherscan</a></td></tr></tbody></table>

### Upgrade Pattern

The OclTokenizer contracts use the **UUPS (Universal Upgradeable Proxy Standard)** pattern:

* Proxy contracts hold state and delegate calls to implementation contracts
* Only the contract owner can authorize upgrades
* Contract addresses remain stable across upgrades

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

### Related

* [Architecture](../../technical-deep-dive/architecture.md) — how the Lab contracts fit together
* [Audits](../../security/audits.md) — audit reports
* [Tokenization API](../../api-reference/tokenization-api.md) — tokenizing a Lab through the API

{% include "../../.gitbook/includes/support.md" %}
