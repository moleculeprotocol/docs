---
description: >-
  The programmable onchain identity that owns, controls, and extends everything
  in a research project
icon: flask-vial
---

# On-Chain Lab

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-033551.png" alt=""><figcaption></figcaption></figure>

### What is a Lab?

An Onchain Lab is the core primitive of Molecule Protocol V3. It is an NFT (ERC-721) that is permanently bound to its own smart contract wallet (ERC-6551), enhanced with account abstraction (ERC-4337), and extensible through a modular plugin architecture (ERC-7579) secured by an onchain attestation registry (ERC-7484).&#x20;

Together, these standards give each Lab its own persistent onchain identity, a fully functional wallet capable of holding any onchain asset, a frictionless user experience that abstracts away gas and key management, and the ability to gain new capabilities over time through installable modules.

When a user creates a Lab, they mint a LabNFT that automatically receives its own smart contract account. This account can own assets, sign transactions, and interact with other protocols — independent of whoever currently holds the LabNFT. Transferring the NFT transfers control of the account and everything inside it in a single transaction.

### The Standards

An Onchain Lab converges five Ethereum standards into a single primitive.

_ERC-721_ makes the Lab a tradable, transferable NFT compatible with the entire NFT ecosystem — marketplaces, wallets, and any contract that understands the ERC-721 interface. The LabNFT is implemented using ERC-721A for gas-efficient minting.

_ERC-6551_ gives the NFT its own smart contract wallet, known as a Token Bound Account. The wallet has no private key of its own — it derives its authority entirely from the current NFT owner. The account address is deterministic, computed from the chain ID, the NFT contract address, the token ID, and a salt. This means the address is known before deployment and is consistent across any chain where the ERC-6551 Registry is deployed.

_ERC-4337_ introduces account abstraction, enabling gasless transactions through paymasters, transaction batching (multiple actions in a single signature), and a validation pipeline that decouples signature verification from execution. Users interact with Labs by signing intents offchain; the protocol handles gas, relay, and execution.

_ERC-7579_ defines a modular account interface. Labs can install executor modules (which perform actions on behalf of the Lab) and fallback modules (which extend the Lab with new function selectors) without changing the core contract. This means Labs can gain new capabilities — licensing logic, governance, oracle integrations, automated royalty distribution — through modules developed by Molecule or third-party developers.

_ERC-7484_ secures the module system through an onchain attestation registry. Before any module can be installed on a Lab, it must be attested by a trusted Molecule attestor. This prevents malicious or unaudited code from being installed while keeping the module ecosystem permissionless for development.

### Identity & Ownership

A Lab maintains two distinct concepts that are often conflated in traditional systems: a persistent identity and a transferable controller.

The Lab's _identity_ is its Token Bound Account address. This address is permanent and deterministic — it is derived from the LabNFT's contract address and token ID at deploy time and never changes, regardless of who owns the Lab. Reputation, transaction history, data provenance, and onchain relationships all accrue to this address permanently.

The Lab's _controller_ is the wallet currently holding the LabNFT. This can change through sales, transfers, or marketplace transactions. When it does, the new holder immediately gains full control of the Lab account — the account's `owner()` function dynamically resolves the current NFT holder on every call. No migration, no re-keying, no governance vote.

This separation is what makes Labs composable. A Lab can be sold on a marketplace and the buyer receives not just an NFT, but the entire project: its treasury, its data references, its IP-NFTs, its transaction history, and its onchain reputation. The identity persists; only the controller changes.

### What Labs Own

<table><thead><tr><th width="174.08203125">Asset Type</th><th>Description</th></tr></thead><tbody><tr><td>IP-NFTs</td><td>Onchain representation of intellectual property rights</td></tr><tr><td>IP Tokens</td><td>Fungible tokens created from tokenizing IP-NFTs</td></tr><tr><td>Treasury</td><td>ETH, stablecoins, or any ERC-20 tokens</td></tr><tr><td>Data References</td><td>CIDs pointing to encrypted research data stored off-chain</td></tr><tr><td>Child Labs</td><td>Nested Labs for hierarchical research programs</td></tr><tr><td>Licenses</td><td>Rentable NFTs (ERC-4907) for time-bound IP access</td></tr><tr><td>External Assets</td><td>Any ERC-20/721/1155 from the broader ecosystem</td></tr></tbody></table>

Because the Token Bound Account is a general-purpose smart contract wallet, a Lab can hold any asset that any Ethereum account can hold. The asset types listed above represent the assets that are semantically meaningful within the Molecule ecosystem, but the wallet is not limited to these.

### Module Registry

Labs extend functionality through installable modules, governed by the ERC-7579 modular account standard and secured by the ERC-7484 attestation registry.

Modules come in two types. _Executor modules_ can initiate transactions from the Lab account — they call into the Lab to execute actions on its behalf. This is the mechanism by which third-party logic (automated royalty distribution, milestone-based fund release, governance voting) can operate on a Lab without requiring the owner to manually trigger each action. _Fallback modules_ extend the Lab's interface by registering new function selectors, allowing the Lab to respond to function calls it does not natively support.

All modules must be attested by a trusted Molecule attestor in the ERC-7484 registry before they can be installed. This creates a permissionless development model with a security gate: anyone can build a module, but it must pass attestation before any Lab can use it. Module installation requires a valid UserOperation through the ERC-4337 EntryPoint, meaning only the Lab's owner can authorize the installation.

### Role-Based Access Control

The modular architecture enables Labs to support delegated permissions without transferring ownership. Through executor modules, Lab owners can authorize scoped access — for example, allowing an AI agent to read data, run analyses, or spend from the treasury within defined budget limits and expiration dates.

The target permission model supports roles such as Owner (full control, can transfer the LabNFT), Admin (manage roles, configure access, manage treasury), Contributor (upload data, create announcements), and Viewer (read-only access to permitted data). These roles are implemented at the application and module layer, not in the core Lab contract, which enforces only two access levels: the NFT owner and the ERC-4337 EntryPoint.

### Onchain Activity Log

Every transaction executed through a Lab creates a permanent, timestamped onchain record. Because all execution flows through the Token Bound Account, the Lab's transaction history forms a complete audit trail of its lifecycle.

<table><thead><tr><th width="219.4296875">Event</th><th>What's Recorded</th></tr></thead><tbody><tr><td>Data upload</td><td>Timestamp, uploader, data reference, version</td></tr><tr><td>Funding received</td><td>Source address, amount, token type</td></tr><tr><td>IP-NFT minted</td><td>Metadata, linked data, ownership</td></tr><tr><td>License issued</td><td>Licensee, duration, terms, payment</td></tr><tr><td>Treasury transaction</td><td>Recipient, amount, purpose</td></tr><tr><td>Role change</td><td>Address, role granted/revoked, timestamp</td></tr></tbody></table>

This creates a verifiable chain of custody for the entire research lifecycle. Reputation accrues to the Lab's permanent address, not to a CV, not to an institution. When collaborators or funders evaluate a project, the evidence is cryptographic and publicly auditable.

### Lab Tokenization _(Coming soon)_

Labs will be able to affiliate with or mint a Group Token to create an economic layer around the Lab's assets. The planned model allows Lab owners to select or mint an ERC-20 token, attach a fee router to direct revenue from the Lab's assets (IP licensing royalties, dataset access payments, trading fees, DeFi yield), enable staking for token holders to receive a share of fee flows, and optionally configure automatic token buybacks from revenue.&#x20;

_\*This feature is part of the V3 roadmap and is not yet available in the current protocol deployment._
