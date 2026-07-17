---
description: >-
  How Labs discover, install, and secure modular extensions through the ERC-7484
  attestation registry
icon: cabinet-filing
---

# Module Registry

### What is the Module Registry?

The Module Registry is the onchain attestation system that governs which modules can be installed on Onchain Labs. It implements the ERC-7484 standard — a registry where trusted attestors vouch for module contracts before any Lab can use them. This creates a permissionless development model with a security gate: anyone can build a module, but it must be attested before it can be installed.

The Module Registry is a separate contract from the Labs themselves. It stores attestation records for every module in the ecosystem and exposes query functions that Labs call during module installation to verify that a module has been approved.

### Why Modules?

An Onchain Lab's core contract handles identity, ownership, validation, and execution. It does not contain application logic for fundraising, data management, licensing, governance, or any other domain-specific capability. These are provided by modules.

This separation matters because scientific research projects evolve. A Lab might start with basic data storage, later add fundraising capabilities to run a token sale, then install licensing logic when results are ready for commercialisation, and eventually add governance modules when the community needs decision-making tools. At no point does the Lab need to migrate to a new contract, re-register assets, or break its onchain history. The identity, treasury, data references, and reputation all persist — only the capabilities change.

The alternative — building every possible feature into the core Lab contract — would produce bloated, expensive deployments where every Lab pays gas for functionality it may never use. It would also mean that new capabilities require upgrading the core contract, affecting every Lab in the system simultaneously.

### Module Types

Onchain Labs support two module types, defined by the ERC-7579 modular account standard.

#### (i) Executor modules

Contracts that can initiate transactions _from_ a Lab account. When installed, an executor gains the ability to call into the Lab and execute actions on its behalf. This is the mechanism for automated and third-party logic: a royalty distribution module can move funds from the Lab's treasury, a milestone module can release payments when conditions are met, an AI agent can execute research workflows within defined boundaries. Executors operate under the authority of the module installation — the Lab owner authorised the module's capabilities when they installed it.

#### (ii) Fallback modules

Extend a Lab's interface by registering new function selectors. When a call arrives at the Lab for a function the core contract doesn't recognise, the Lab routes it to whichever fallback module is registered for that selector. This enables Labs to respond to entirely new interfaces without modifying the core contract. A fallback module might add ERC-1155 receiver support, implement a custom governance voting interface, or expose data querying functions.


### The Attestation Model

Before a module can be installed on any Lab, it must be attested in the ERC-7484 Registry. An attestation is a signed onchain record created by a trusted attestor that vouches for a specific module contract.

Each attestation record contains the creation timestamp, an optional expiration time, the attester's address, the module type (executor or fallback), and an optional field for custom attestation data. Attestations can be revoked by the attester at any time, which records a revocation timestamp and immediately prevents new installations of that module.

Labs are configured with an attester policy: a list of trusted attestor addresses and a threshold specifying how many must have attested a module before it can be installed. In the current deployment, Molecule acts as the sole attestor, but the architecture supports multi-attestor configurations where, for example, two out of three independent auditors must approve a module.

This model separates _development_ from _approval_. A third-party developer can write, test, and deploy a module contract without needing permission from anyone. But before any Lab can install it, the module must pass the attestation gate. This keeps the ecosystem open for innovation while protecting Labs from malicious or unaudited code.

### Module Installation

Installing a module is an onchain transaction that must be authorised by the Lab owner through the ERC-4337 EntryPoint. The installation flow proceeds as follows:

The Lab owner submits a UserOperation requesting module installation with the module's contract address, its type (executor or fallback), and initialisation data. The Lab's account contract calls the ERC-7484 Registry to verify that the module has a valid, non-expired, non-revoked attestation from the required attestors. If the attestation check passes, the module is registered in the Lab's internal storage: executor modules are recorded in the ExecutorManager (with a snapshot of the owner who installed them), and fallback modules are registered in the SelectorManager with a mapping from function selectors to the module address.

From that point forward, the module is active. Executor modules can call into the Lab, and calls to the fallback module's registered selectors are routed to it automatically.

Modules can also be removed: `uninstallModule` clears the executor or fallback registration (and calls the module's `onUninstall` handler), again authorised by the Lab owner through the EntryPoint.

### How Modules Work

When a call arrives at a Lab's smart account for a function the core contract doesn't define, the account looks up the selector in its **own SelectorManager storage** (the ERC-7484 Registry is consulted only for attestation, not for routing). If a fallback module is registered for that selector — and its attestation is still valid — the account forwards the call to the module as a regular external call with the original sender appended ERC-2771-style. Modules never run via `delegatecall`: they execute in their own storage context, and the Lab's assets move only through the account's explicit execution paths.

The two module types differ by direction:

**Fallback modules** extend the Lab's *inbound* interface. Calls to their registered selectors arrive through the ERC-4337 EntryPoint (the account's caller policy enforces EntryPoint-only dispatch), letting the Lab respond to interfaces the core contract doesn't natively implement — data-reference recording, custom query functions, receiver interfaces.

**Executor modules** drive *outbound* execution. An installed executor **contract** (EOAs cannot be executors) calls `executeFromExecutor` on the Lab to trigger actions on its behalf — automated distributions, scheduled operations, cross-protocol integrations. At execution time the account re-checks the executor's attestation and that the Lab's owner hasn't changed since installation.

### Module Installation and Security

Installing a module records it in the Lab's own module storage — selector→module mappings for fallback modules, an installed flag plus owner snapshot for executors. The Lab owner controls which modules are installed and can remove them via `uninstallModule`.

Because modules execute as external calls rather than `delegatecall`, they cannot touch the Lab's storage directly. The account increments its public `state` counter on module-routed calls and re-validates attestations at execution time, so a revoked module stops working immediately even if it remains installed.

### Security Model

The module system has several layers of security that work together.

#### Attestation gating

Ensures that only reviewed and approved modules can be installed. The ERC-7484 Registry is the first line of defence — a module that hasn't been attested, or whose attestation has expired or been revoked, cannot be installed on any Lab.

#### Owner-only installation

Ensures that only the Lab owner can decide which modules to install. Module installation requires a valid UserOperation through the ERC-4337 EntryPoint, authenticated against the current LabNFT owner.

#### EntryPoint execution gating

Ensures that critical execution paths (including module-initiated actions) flow through the ERC-4337 EntryPoint, which validates every operation before execution.

#### Beacon upgradeability

Provides a system-wide upgrade mechanism. All Lab accounts share a single implementation behind a beacon proxy. The beacon owner (a Molecule-controlled address) can upgrade the implementation, affecting all Labs simultaneously. This is distinct from module installation — it upgrades the core account logic, not the installed modules. Upgrade authority should be understood as a trust assumption of the current deployment.

### Example Modules

The module architecture is designed to support a growing ecosystem of capabilities. Examples of module categories include automated royalty distribution, milestone-based fund release, IP licensing and rental logic, governance and voting mechanisms, data storage and retrieval, oracle integrations for external data, cross-lab collaboration protocols, and AI agent execution boundaries.

Each of these would be deployed as an independent contract, attested in the ERC-7484 Registry, and installable by any Lab owner who needs that capability.
