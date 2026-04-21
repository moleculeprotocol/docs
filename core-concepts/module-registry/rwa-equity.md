---
description: >-
  The RWA Equity module bridges onchain IP Token ownership with off-chain
  shareholder rights, enabling a compliant pathway from token holder to
  real-world equity participant
icon: envelope-ribbon
---

# RWA Equity

## RWA Equity

The RWA Equity module will provide a compliant pathway for IP Token holders to qualify for real-world equity in the research projects they support. It bridges the gap between onchain token ownership — which provides governance participation and economic exposure — and off-chain shareholder rights, which confer legal ownership in the underlying entity.

This is an opt-in framework. Token holders who prefer the liquidity and simplicity of standard IP Tokens continue operating exactly as before. Only those who choose to pursue equity enter the lock-and-qualify process.

### The Problem

IP Tokens give holders meaningful participation in a research project: governance voice, economic exposure to the IP's value, and access to token-gated data rooms. But IP Tokens are not equity. They do not confer legal ownership, board representation, dividend rights, or the protections that come with being a registered shareholder in a legal entity.

For research projects that reach a stage where real-world equity matters — licensing deals, acquisition negotiations, regulatory milestones — there needs to be a mechanism that connects committed onchain supporters to the traditional equity stack without compromising regulatory compliance.

### How It Works

The RWA Equity module implements a lock-and-qualify process managed through the Lab's modular account:

**Token locking.** A holder swaps their liquid IP Tokens one-to-one for locked tokens through a locking contract. The locked tokens represent the same underlying value but cannot be freely transferred. This creates a commitment signal — the holder is signalling long-term alignment with the project by sacrificing immediate liquidity.

**Identity verification.** The holder completes a KYC/AML process to verify their identity. Upon successful verification, they receive an onchain credential (a non-transferable soulbound token) that attests to their verified status. This credential is required before any equity qualification can proceed.

**Equity qualification.** With locked tokens and a verified identity, the holder is eligible to enter into traditional legal agreements with the project's legal entity. The actual equity grant is executed off-chain through standard corporate processes — share purchase agreements, subscription documents, or equivalent instruments depending on the jurisdiction.

**Ongoing enforcement.** Once a holder is designated as a shareholder, their locked tokens cannot be unlocked. Exit from the shareholder position requires an off-chain request and administrative approval, ensuring that equity holders maintain their token commitment throughout their shareholder status.

Locking tokens does not automatically grant equity. It establishes eligibility. The locked token serves as proof of commitment — it prevents a holder from completing KYC, receiving equity, and immediately selling their tokens on the open market.

### Role in the Module Architecture

The RWA Equity framework is designed to operate as a module within the Lab's modular TBA. The onchain components — token locking, credential verification, and shareholder status tracking — would be implemented as a combination of fallback modules (adding equity-related query functions to the Lab) and executor modules (enabling authorised compliance services to trigger lock/unlock operations and update shareholder status).

This modular approach means that RWA equity is an installable capability, not a mandatory feature of every Lab. Projects that operate purely in the token economy never install the module. Projects that want to offer an equity pathway install it when ready, and the ERC-7484 Module Registry ensures the equity module contracts are attested and audited before activation.

### Legal Design

The framework maintains a deliberate separation: the token is not equity. This is not a limitation — it is the core design principle that enables regulatory compliance.

Holding locked tokens is a prerequisite for pursuing equity through separate legal agreements. The onchain components (locking, credential verification, status tracking) provide the infrastructure, but the equity grant itself occurs through traditional legal channels. This structure means the token does not need to be classified as a security in jurisdictions where that would create compliance burdens for the broader holder base. Those who want equity follow the qualification path. Those who do not want equity continue holding standard, liquid IP Tokens with no additional obligations.

### Current Status

The RWA Equity framework is in design. The `TimelockedToken.sol` contract in the IPNFT repository provides a foundation for time-based token locking, and the existing IP Token infrastructure supports the token economics that underpin the framework. The specific locking contract, credential system, and shareholder management logic for the equity qualification pathway are planned for future development.

The modular account infrastructure that would host the equity module — the `SelectorManager`, `ExecutorManager`, the ERC-7484 registry, and the `installModule` flow — is implemented and deployed on Sepolia, providing the foundation for installation when the equity module contracts are ready.
