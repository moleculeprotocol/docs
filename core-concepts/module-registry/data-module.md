---
description: >-
  The Data Module adds data-referencing capabilities to an OnChain Lab's smart
  account, enabling the TBA to record, query, and verify references to off-chain
  research
icon: binary-circle-check
---

# Data Module

## Data Module

The Data Module extends an OnChain Lab's Token Bound Account with the ability to record and manage references to off-chain research data. As a fallback module installed through the Lab's modular account architecture, it adds function selectors to the TBA that allow data references — content identifiers, version pointers, and provenance metadata — to be written to and read from the Lab's onchain state.

This module is the onchain interface between the Lab's smart account and the broader data infrastructure described in the Data section. While the off-chain stack handles storage, encryption, versioning, and access control, the Data Module gives the TBA itself the ability to hold a verifiable index of its research output.

### The Trust Problem

Without onchain data references, the mapping between a Lab and its research data exists only in Kamu's provenance database — a centralised dependency. The Lab's TBA would hold assets, manage treasury, and control IP-NFTs, but it would have no verifiable link to the actual scientific work those assets represent. If Kamu is unavailable or compromised, the association between a Lab and its research output becomes unverifiable. Anyone claiming a dataset belongs to a particular Lab would need to trust an off-chain service to confirm it.

The Data Module solves this by mirroring critical references onchain. Every file uploaded to a Lab produces a content identifier (CID) — a cryptographic hash derived from the file's contents. The Data Module records that CID, along with a path and timestamp, directly in the Lab's TBA. This creates a redundant, trustless index: even if every off-chain service goes down, anyone can query the Lab's smart account to discover what data it contains, when each file was committed, and what its content hash is. The actual file content can then be retrieved from IPFS using the CID.

This onchain anchoring also unlocks composability that would be impossible with a purely off-chain data index. Because the references live in a smart contract, other contracts can read them programmatically — creating trustless integrations between data and the rest of the onchain ecosystem.

### Role in the Module Architecture

The Data Module is a **fallback module** (ERC-7579 module type 3). It is installed on the Lab's TBA through the `SelectorManager`, which maps specific function selectors to the module contract. When the Lab receives a call targeting one of these data-related selectors — for example, recording a new file reference or querying existing references — the TBA's `fallback()` function routes the call to the Data Module for execution.

This follows the same pattern as all fallback modules in the Molecule architecture: the selector is looked up in the `SelectorConfig` mapping, the module is verified against the ERC-7484 Module Registry, and the call is dispatched using either `CALLTYPE_SINGLE` (ERC-2771 forwarded call) or `CALLTYPE_DELEGATECALL` depending on the module's configuration. See Fallback Modules for the full dispatch mechanism.

### What the Data Module Does

The Data Module is responsible for the onchain reference layer — the set of operations that anchor off-chain data to the Lab's blockchain identity:

* **Recording data references.** When a file is uploaded through the data pipeline (via the Labs API, through Kamu, to IPFS), the final step is recording a reference in the Lab's TBA. The Data Module handles this write operation, storing the content identifier (CID), the file path or dataset DID, and a timestamp within the Lab's onchain state.
* **Querying data references.** External contracts, protocols, and interfaces can call the Data Module's view functions to discover what data a Lab contains, when each file was committed, and what its content hash is. This is where onchain composability becomes concrete: a governance contract can verify that required documentation exists before allowing a proposal to pass, a funding mechanism can check that milestone deliverables have been recorded before releasing funds, a marketplace can evaluate a Lab's data footprint as part of due diligence before acquisition, and external protocols can query a Lab's published datasets to build services on top of them.
* **Version tracking.** When a file is updated, the Data Module records the new CID alongside the previous one, creating an append-only chain of version references within the TBA. The full version history remains retrievable from the Lab's onchain state, independent of any off-chain service.
* **Provenance anchoring.** Every data write through the module is an onchain transaction — timestamped, signed, and permanently associated with the Lab's address. This creates an immutable audit trail of all data operations that forms part of the Lab's verifiable research record. Dataset uploads, version updates, access level changes, and agent interactions all generate verifiable records, transforming the Lab from a static asset container into a living, auditable log of its entire research lifecycle.

### Why a Module

Making data referencing a module rather than a core account function follows the protocol's design philosophy of keeping the base TBA minimal and extending functionality through installable, auditable, registry-verified modules.

This separation provides several advantages. The data referencing logic can be upgraded independently of the core account contract — new reference formats, additional metadata fields, or optimised storage layouts can be deployed as a new module version without requiring a beacon upgrade that affects all Labs. Labs that do not need onchain data references (for example, Labs used purely for treasury management or governance) do not carry the gas overhead of unused storage slots and functions. The module must pass ERC-7484 registry attestation before installation, providing an additional security checkpoint beyond the core account audit.

### Current Status

The Data Module is in active design. The modular account infrastructure that supports it — the `SelectorManager`, the fallback dispatch mechanism, the ERC-7484 registry, and the `installModule` flow — is implemented and deployed on Sepolia. The data module contract itself, which will define the specific selectors for recording and querying data references, is planned as a future deployment.

The off-chain data infrastructure is operational. File uploads, encryption, versioning through Kamu, and access control through the AccessResolver are all functional in the current system. The Data Module will formalise the onchain anchoring step that currently occurs through the Labs API, bringing it fully under the TBA's modular account architecture.

