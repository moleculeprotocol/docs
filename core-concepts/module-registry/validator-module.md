---
description: >-
  How Onchain Labs authenticate and authorise operations through modular,
  NFT-ownership-derived signature verification
icon: hexagon-nodes
---

# Validator Module

### Validator Module

A Validator Module determines who is authorised to act on behalf of an Onchain Lab. It is the first and most fundamental security gate in the system — without a functioning validator, a Lab cannot process any transaction.

Every time a transaction is submitted through the ERC-4337 EntryPoint, the Lab's account contract delegates the signature check to its installed validator. The validator resolves the current owner of the Lab NFT onchain, recovers the signer from the submitted signature, and compares the two. If they match, the operation proceeds. If not, it is rejected before any execution takes place.

The validator answers one question: who controls this Lab right now?

### The Validation Flow

When a Lab owner wants to perform any action — installing a module, sending assets, or executing a function — the operation follows a deterministic path through four components:

**1. UserOperation Submission** The owner constructs and signs an ERC-4337 `UserOperation`, then submits it to a Bundler, which forwards it to the global `EntryPoint` contract.

**2. Account Delegation** The EntryPoint calls `validateUserOp` on the Lab's account contract (`Modular_OnChainLab`). The account extracts the `ValidationId` from the operation and delegates to the `ValidationManager`, which routes the call to the installed root validator.

**3. Ownership Resolution** The validator calls `signer()` on the Lab account, which internally calls `owner()`. This function invokes the ERC-6551 `token()` method to retrieve the bound NFT's contract address and token ID, then queries `ownerOf(tokenId)` on the NFT contract. The result is the current owner's address — resolved live from onchain state, never cached.

**4. Signature Verification** The validator recovers the signer from the UserOperation's signature using ECDSA and compares it against the resolved owner address. The implementation checks two encoding formats for maximum wallet compatibility:

* **Raw hash recovery** — the `userOpHash` is used directly, as some ERC-4337 signing flows produce signatures over the raw hash.
* **EIP-191 prefixed recovery** — the hash is wrapped with the standard Ethereum signed message prefix (`\\x19Ethereum Signed Message:\\n32`), as wallet UIs like MetaMask and libraries like ethers.js commonly apply this prefix.

If either recovery produces a match, the operation is approved. If neither does, it is rejected.

This dual-check approach ensures Labs are accessible from the widest range of wallet software and signing libraries without requiring users to configure anything.

### The Root Validator

Every Lab is created with a Root Validator installed during the factory's initialisation process. This is not optional — it is the bootstrap mechanism that makes all subsequent operations possible. A Lab without a validator cannot sign any transaction, which means it cannot install modules through the normal flow (since module installation itself requires a valid signature).

The Root Validator ships with the protocol and implements a straightforward ownership model: the wallet that currently owns the Lab NFT is the sole authorised signer.

When the Root Validator is installed via `onInstall`, it receives the Lab's account address as the first 20 bytes of the calldata and stores a mapping between the calling smart account and the Lab contract reference. From that point on, every time the validator is asked to verify a signature, it looks up the Lab, resolves the current NFT owner through the `signer()` function, and checks the signature against that address.

This design has an important property: if the Lab NFT is transferred to a new wallet, control of the Lab transfers immediately and automatically. The new owner can sign transactions, install modules, and manage assets. The previous owner loses all authority the moment the NFT leaves their wallet. No migration, no key rotation, no administrative action is required.

### ERC-1271 Signature Verification

Beyond validating UserOperations, the Root Validator also implements ERC-1271 contract signature verification through the `isValidSignatureWithSender` function (the ERC-7579 variant of the standard `isValidSignature`).

This allows external contracts to ask the Lab whether a given signature was produced by its authorised controller. The validator performs the same dual ECDSA check — raw hash recovery, then EIP-191 prefixed recovery — and returns the ERC-1271 magic value (`0x1626ba7e`) on success or an invalid marker on failure.

This capability is critical for the Lab's composability within the broader Ethereum ecosystem. It enables the Lab to sign off-chain messages, approve token permits, participate in governance votes, interact with marketplaces, and integrate with any protocol that supports ERC-1271 contract signatures — all while deriving authority from NFT ownership.

### Validator vs. Other Module Types

Validator modules occupy a unique position in the module architecture. Executor modules and fallback modules add capabilities — they extend what a Lab can do. Validators, by contrast, govern access — they decide who gets to do anything at all.

This distinction has practical consequences. A Lab's root validator is installed at creation time by the factory contract, before the Lab owner has signed any transactions. It cannot be installed through the normal module installation flow because that flow itself requires a valid signature, which requires a working validator. The root validator is the trust anchor that makes everything else possible.

The `ValidationManager` supports two validation types in its type system: `VALIDATION_TYPE_ROOT` for the primary validator that controls all access, and `VALIDATION_TYPE_PERMISSION` for future permission-based validators that could grant scoped, limited authority to specific contracts or agents. Currently, only root validation is active — the permission system is architected but not yet enabled.

### Why This Matters for Onchain Labs

The validator module design reflects the core principle of the Onchain Labs architecture: the Lab's identity is persistent, but its controller is transient. A research project's onchain history, reputation, treasury, and data belong to the Lab. The validator simply determines who holds the keys at any given moment.

This separation makes ownership transfers clean and complete. When a DAO acquires a Lab by purchasing its NFT, the validator immediately recognises the new owner. When a research team transitions leadership, they transfer the NFT and the new lead gains full control. The Lab's accumulated history, installed modules, and recorded data remain intact and uninterrupted throughout.

Making validation a modular component also opens the door to alternative authentication schemes. Future validator modules — each a separate contract, installable through the same registry system without any changes to the core Lab contract — could include:

* **Multisig validators** requiring multiple signatures for high-value operations
* **Session key validators** granting temporary, scoped access to automated services or AI agents
* **Social recovery validators** allowing trusted parties to recover access if keys are lost
* **Threshold validators** requiring m-of-n approval from a defined set of signers

### Contract Reference

The Root Validator is deployed and verified on Sepolia testnet:

<table><thead><tr><th width="182.12109375">Contract</th><th>Address</th></tr></thead><tbody><tr><td>RootValidator</td><td><code>0x4F63252944ac8C7846F73fD057A114E71A374585</code></td></tr></tbody></table>

Source code: `src/modules/validator/RootValidator.sol`

#### IValidator Interface

The Root Validator implements the `IValidator` interface from ERC-7579. Any custom validator module must implement the following functions:

* `validateUserOp(PackedUserOperation calldata userOp, bytes32 userOpHash)` — Called by the account during ERC-4337 validation. Returns `0` for success, `1` for failure.
* `isValidSignatureWithSender(address sender, bytes32 hash, bytes calldata sig)` — ERC-1271 signature check. Returns `0x1626ba7e` on success.
* `onInstall(bytes calldata data)` — Called when the module is installed on an account. Used to initialise storage.
* `onUninstall(bytes calldata data)` — Called when the module is removed. Used to clean up storage.
* `isModuleType(uint256 typeId)` — Returns `true` for `MODULE_TYPE_VALIDATOR` (`1`).
* `isInitialized(address smartAccount)` — Returns whether the module has been initialised for a given account.

