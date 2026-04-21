---
description: >-
  How the protocol's smart contracts are structured, deployed, and composed at
  the implementation level
icon: microchip
---

# Architecture

### Contract Topology

The protocol is implemented as a set of interconnected Solidity contracts deployed on Ethereum and EVM-compatible chains. At the center is the `Modular_OnChainLab` — the account implementation that unifies the standards described in the On-Chain Lab section into a single deployable contract.

&#x20;Surrounding it are the factory, registry, proxy, and validation contracts that handle deployment, upgradeability, and security gating. The diagram below shows how these components relate to one another.

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-031553.png" alt=""><figcaption></figcaption></figure>

#### `OnChainLabFactory`&#x20;

The singular entry point for Lab creation. It deploys the `Modular_OnChainLab` implementation at construction time (making itself the only address authorized to call `initialize()` on new accounts). It optionally deploys a Beacon and Router for the upgradeable pattern. It exposes two creation paths: `createAccount` for binding an existing LabNFT to an account, and `mintAndCreateAccount` for minting the NFT and creating the bound account in a single atomic transaction.&#x20;

#### `Modular_OnChainLab`&#x20;

The account implementation. It inherits from `ERC4337Account`, `EIP712`, `ERC7739`, `IERC6551Account`, `IERC6551Executable`, `IERC7579Account`, `ValidationManager`, and `ERC7484RegistryAdapter`. It exposes three overloaded `execute()` signatures (one for each standard) and routes all execution through a shared `ExecLib`. Each execution increments a public `state` counter that external contracts (such as marketplaces) can use to detect front-running during NFT sales. The `owner()` function dynamically calls `ownerOf()` on the bound LabNFT contract on every invocation — ownership is never cached or stored.

#### `ERC-6551 Registry`&#x20;

Deploys Lab accounts as EIP-1167 minimal proxies via CREATE2. The token binding data (chain ID, token contract, token ID) is appended to the proxy's bytecode at deployment and read at runtime via `extcodecopy`. This data is immutable.

#### `OnChainLabRouter` and `OnChainLabBeacon`&#x20;

The optional upgradeability layer. The Router is a delegatecall proxy that reads the current implementation address from the Beacon. The Beacon is an ownable contract; when the owner updates the implementation, all Lab accounts upgrade simultaneously. For non-upgradeable deployments, the factory sets the Router to point directly at the implementation.

#### `RootValidator`&#x20;

The default validation module installed during account initialization. It resolves the current LabNFT owner and validates ERC-4337 UserOperation signatures against that address. The validator must be attested in the ERC-7484 Module Registry before the factory can install it.

#### `ERC-7484 Module Registry`&#x20;

The attestation-based gating layer. Every executor and fallback module must be attested by a trusted Molecule attestor and meet the configured attestation threshold before it can be installed on any Lab account.

### **Proxy Delegation Chain**

Each Lab account is deployed as a layered proxy. Understanding the call path is essential for developers interacting with or extending the protocol.

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-032106.png" alt=""><figcaption></figcaption></figure>

Storage resides in the minimal proxy. Logic resides in the implementation. The Beacon owner can upgrade the implementation for all accounts in a single transaction, without requiring any per-account migration. The token binding data (chain ID, token contract, token ID) is embedded in the proxy's bytecode at offset `0x4d` and is read via `extcodecopy` — it is never stored in contract storage and cannot be modified after deployment.

### Lab Creation

The canonical path for creating a Lab is `mintAndCreateAccount`, which executes the entire flow in a single transaction.

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-032310.png" alt=""><figcaption></figcaption></figure>

The factory also supports `createAccount` for cases where the LabNFT already exists (e.g., minted through a separate flow). Both paths converge on the same internal `_createAccount` and `_initializeAccount` logic, ensuring consistent initialization regardless of entry point.

### Execution Paths

The account supports two execution paths: direct calls by the NFT owner, and UserOperations mediated by the ERC-4337 EntryPoint.

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-032520.png" alt=""><figcaption></figcaption></figure>

Path B enables gasless transactions (via Paymasters), transaction batching, and off-chain signature relay — none of which require the user to hold ETH or manage gas directly. The ERC-7579 `execute(ExecMode, bytes)` variant is restricted to EntryPoint-only access, ensuring modular execution is always mediated by ERC-4337 validation.

### Ownership Transfer

Ownership transfer is not a dedicated protocol function — it is an emergent property of the dynamic ownership resolution. When the LabNFT is transferred via standard ERC-721 `transferFrom`, the Lab account's `owner()` function immediately resolves to the new holder on the next call. No migration, no re-initialization, no key rotation.

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-032643.png" alt=""><figcaption></figcaption></figure>

The `state` counter provides a mechanism for marketplace contracts to guard against front-running. A buyer can record the Lab's `state` value at the time of listing and verify it has not changed at the time of purchase — if the seller executed any transactions between listing and sale (e.g., draining assets), the `state` will have incremented.

### Module Installation

Modules extend Lab functionality without modifying the core account contract. Installation is gated by both the EntryPoint (requiring a valid UserOperation) and the ERC-7484 Module Registry (requiring attestation), forming a defense-in-depth model.

<figure><img src="../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-032738.png" alt=""><figcaption></figcaption></figure>

The account currently supports two module types: executors (which can call `executeFromExecutor` on the Lab) and fallback modules (which are routed via the `fallback()` function based on function selector). Both types are subject to registry attestation checks at installation time and at execution time. Hook modules are stubbed in the architecture but not yet fully implemented.

### Signature Verification

The account implements a two-tier signature validation strategy via `isValidSignature` (ERC-1271). It first attempts ERC-7739 validation, which uses nested EIP-712 typed data with chain-specific context for cross-chain replay protection. If ERC-7739 does not recognize the signature format, the account falls back to standard ECDSA recovery against the current NFT owner. This dual approach provides strong cross-chain security while maintaining backward compatibility with contracts that use simple signatures.
