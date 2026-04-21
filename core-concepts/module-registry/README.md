---
description: >-
  How Onchain Labs discover, install, and secure modular extensions through the
  ERC-7484 attestation registry
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

#### (i) Executor modules&#x20;

Contracts that can initiate transactions _from_ a Lab account. When installed, an executor gains the ability to call into the Lab and execute actions on its behalf. This is the mechanism for automated and third-party logic: a royalty distribution module can move funds from the Lab's treasury, a milestone module can release payments when conditions are met, an AI agent can execute research workflows within defined boundaries. Executors operate under the authority of the module installation — the Lab owner authorised the module's capabilities when they installed it.

#### (ii) Fallback modules&#x20;

Extend a Lab's interface by registering new function selectors. When a call arrives at the Lab for a function the core contract doesn't recognise, the Lab routes it to whichever fallback module is registered for that selector. This enables Labs to respond to entirely new interfaces without modifying the core contract. A fallback module might add ERC-1155 receiver support, implement a custom governance voting interface, or expose data querying functions.

Both module types can optionally have **hooks** associated with them — pre-execution and post-execution checks that run before and after the module's logic. Hooks enable cross-cutting concerns like access control, rate limiting, or audit logging to be applied to any module without modifying the module itself.

### The Attestation Model

Before a module can be installed on any Lab, it must be attested in the ERC-7484 Registry. An attestation is a signed onchain record created by a trusted attestor that vouches for a specific module contract.

Each attestation record contains the creation timestamp, an optional expiration time, the attester's address, the module type (executor or fallback), and an optional field for custom attestation data. Attestations can be revoked by the attester at any time, which records a revocation timestamp and immediately prevents new installations of that module.

Labs are configured with an attester policy: a list of trusted attestor addresses and a threshold specifying how many must have attested a module before it can be installed. In the current deployment, Molecule acts as the sole attestor, but the architecture supports multi-attestor configurations where, for example, two out of three independent auditors must approve a module.

This model separates _development_ from _approval_. A third-party developer can write, test, and deploy a module contract without needing permission from anyone. But before any Lab can install it, the module must pass the attestation gate. This keeps the ecosystem open for innovation while protecting Labs from malicious or unaudited code.

### Module Installation

Installing a module is an onchain transaction that must be authorised by the Lab owner through the ERC-4337 EntryPoint. The installation flow proceeds as follows:

The Lab owner submits a UserOperation requesting module installation with the module's contract address, its type (executor or fallback), and initialisation data. The Lab's account contract calls the ERC-7484 Registry to verify that the module has a valid, non-expired, non-revoked attestation from the required attestors. If the attestation check passes, the module is registered in the Lab's internal storage: executor modules are recorded in the ExecutorManager with an optional hook association, and fallback modules are registered in the SelectorManager with a mapping from function selectors to the module address.

From that point forward, the module is active. Executor modules can call into the Lab, and calls to the fallback module's registered selectors are routed to it automatically.

Module uninstallation is planned but not yet implemented in the current deployment.

### How Modules Work

When a call is made to a Lab's smart account, the account first checks whether the requested function is defined internally. If not, it consults the Module Registry to find a registered module that handles that function selector. The call is then delegated to the appropriate module, which executes on behalf of the Lab.

Two primary module types exist within this architecture:

Fallback Modules handle calls initiated by the Lab owner. When an owner wants to perform an action not natively supported by the Lab contract — such as uploading data or configuring settings — the call routes to a Fallback Module. These modules extend what the owner can do with their Lab.

Executor Modules handle calls from external entities such as other contracts or externally owned accounts. These enable third parties to trigger actions on a Lab when certain conditions are met — for example, automated distributions, scheduled operations, or cross-contract integrations.

### Module Installation and Security

Installing a module registers it with the Lab's Module Registry, mapping specific function selectors to\
the module's address. The Lab owner controls which modules are installed, and can remove or replace modules as needed.

Security considerations are critical in this architecture. When a module executes via delegate call, it\
operates within the context of the Lab's account — meaning it can access the Lab's storage and assets. The Module Registry must carefully manage state increments to prevent reentrancy attacks and ensure that modules cannot interfere with each other's storage.

### Security Model

The module system has several layers of security that work together.

#### Attestation gating&#x20;

Ensures that only reviewed and approved modules can be installed. The ERC-7484 Registry is the first line of defence — a module that hasn't been attested, or whose attestation has expired or been revoked, cannot be installed on any Lab.

#### Owner-only installation&#x20;

Ensures that only the Lab owner can decide which modules to install. Module installation requires a valid UserOperation through the ERC-4337 EntryPoint, authenticated against the current LabNFT owner.

#### EntryPoint execution gating&#x20;

Ensures that critical execution paths (including module-initiated actions) flow through the ERC-4337 EntryPoint, which validates every operation before execution.

#### Beacon upgradeability&#x20;

Provides a system-wide upgrade mechanism. All Lab accounts share a single implementation behind a beacon proxy. The beacon owner (a Molecule-controlled address) can upgrade the implementation, affecting all Labs simultaneously. This is distinct from module installation — it upgrades the core account logic, not the installed modules. Upgrade authority should be understood as a trust assumption of the current deployment.

### Example Modules

The module architecture is designed to support a growing ecosystem of capabilities. Examples of module categories include automated royalty distribution, milestone-based fund release, IP licensing and rental logic, governance and voting mechanisms, data storage and retrieval, oracle integrations for external data, cross-lab collaboration protocols, and AI agent execution boundaries.

Each of these would be deployed as an independent contract, attested in the ERC-7484 Registry, and installable by any Lab owner who needs that capability.
