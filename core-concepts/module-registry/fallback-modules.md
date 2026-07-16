---
description: >-
  Fallback modules extend Lab functionality by handling function calls that are
  not natively defined on the account contract, dispatched through the Solidity
  fallback function via selector-based
icon: square-binary
---

# Fallback Modules

## Fallback Modules

Fallback modules extend the native functionality of a Lab by handling calls to function selectors that are not defined on the account contract itself. When a transaction targets a selector that the Lab does not recognize, the Solidity `fallback()` function intercepts it and routes the call to a registered external module based on a per-selector configuration managed by the `SelectorManager`.

This mechanism allows Labs to support new interfaces, respond to new protocol interactions, and integrate with external systems — all without modifying the core account logic.

### How It Works

Every Lab inherits from `SelectorManager`, which maintains a mapping from 4-byte function selectors to a `SelectorConfig` struct:

```solidity
struct SelectorConfig {
    address module;            // Fallback module contract address
    CallType callType;         // Execution mode: CALLTYPE_SINGLE
    CallerPolicy callerPolicy; // Who may invoke the selector: ENTRYPOINT_ONLY
}
```

When an external call arrives at the Lab with a selector that does not match any native function, the `fallback()` handler executes the following sequence:

1. **State increment** — The Lab increments an internal state counter to track executions and support anti-fraud mechanisms.
2. **Selector lookup** — The handler reads the `SelectorConfig` for `msg.sig` from the selector storage slot.
3. **Installation check** — If `config.module` is the zero address, the selector has no registered module and the call reverts with `InvalidSelector()`.
4. **Access control** — The `callerPolicy` is enforced. The only policy today is `ENTRYPOINT_ONLY`: unless `msg.sender` is the ERC-4337 EntryPoint, the call reverts with `InvalidCaller()`.
5. **Registry verification** — The module address is checked against the ERC-7484 Module Registry to confirm it is an attested, approved fallback module (`MODULE_TYPE_FALLBACK`).
6. **Dispatch** — The call is forwarded to the module as a `CALLTYPE_SINGLE` external call.

### Call Types

**CALLTYPE\_SINGLE (0x00)** — the only supported execution mode. The call is forwarded to the module contract as a standard external call using the ERC-2771 trusted forwarder pattern: the Lab appends the original `msg.sender` address to the calldata before calling the module, allowing the module to identify the true caller even though the call is proxied through the Lab. Modules maintain their own storage.

**CALLTYPE\_DELEGATECALL (0xFF)** exists as a constant in the type system but is **disallowed for fallback modules** — installation with it reverts `InvalidCallType`, and dispatch would revert `UnsupportedCallType`. This is a deliberate security decision (hardened during the 2026 audit): module code never runs inside the Lab's storage context.

### Installation

Fallback modules are installed through the `installModule` function on the Lab account, callable only via the EntryPoint (ensuring the operation has been validated). The installation requires:

* **Module type** — `MODULE_TYPE_FALLBACK` (type ID `3`)
* **Module address** — The address of the fallback module contract
* **Initialization data** — an ABI-encoded `InstallFallbackData` struct: `{ bytes4 selector; CallType callType; CallerPolicy callerPolicy; bool overwrite; bytes selectorData }`. The `callType` must be `CALLTYPE_SINGLE` and the `callerPolicy` must be `ENTRYPOINT_ONLY`; `selectorData` is passed through to the module's `onInstall`.

During installation, the system performs a registry check via ERC-7484 to verify the module is attested for the fallback type. The `SelectorManager` then stores the configuration and emits a `ModuleInstalled` event.

### Hooks

ERC-7579 defines optional pre/post-execution hooks around module calls. In the current contracts, hooks exist only as an unused `IHook` interface — `SelectorConfig` carries no hook field and there is no hook dispatch. Per-selector hooks are a possible future extension, not a present capability.

### Use Cases

Fallback modules enable OnChain Labs to support capabilities that are not part of the core account contract. Example use cases include:

* **Token receiving** — Implementing `onERC721Received`, `onERC1155Received`, or similar callback interfaces so the Lab can receive NFTs and multi-token transfers
* **Flash loan participation** — Implementing flash loan callback interfaces to allow Labs to act as flash loan receivers
* **Protocol integrations** — Supporting interaction interfaces required by external DeFi protocols, governance systems, or marketplace contracts
* **Custom query interfaces** — Exposing read-only view functions that aggregate or compute data from the Lab's state for external consumers

### Security Considerations

Fallback modules are security-critical because they can execute arbitrary logic in response to any unrecognized function call on the Lab. Several safeguards are built into the architecture:

The ERC-7484 Module Registry provides attestation-based trust. Every fallback module must be attested as type `MODULE_TYPE_FALLBACK` before it can be dispatched, ensuring only reviewed and approved modules can handle calls.

The `ENTRYPOINT_ONLY` caller policy ensures fallback selectors can only be invoked through the EntryPoint — meaning every call must pass UserOperation validation first. This prevents unauthorized contracts or EOAs from directly triggering fallback logic.

Because `delegatecall` dispatch is disallowed, fallback modules can never read or corrupt the Lab's storage directly — they interact with the Lab only through its explicit external interfaces.

The state counter increment on every fallback invocation provides an additional anti-replay and execution tracking mechanism.

### Contract Reference

| Contract              | Role                                                         | Source                         |
| --------------------- | ------------------------------------------------------------ | ------------------------------ |
| `SelectorManager.sol` | Manages per-selector fallback configuration and storage      | `src/core/SelectorManager.sol` |
| `OnChainLab.sol`      | Contains the `fallback()` handler and `installModule` logic  | `src/OnChainLab.sol`           |
| `ExecLib.sol`         | Provides the `doFallback2771Call` dispatch helper            | `src/utils/ExecLib.sol`        |
| `Constants.sol`       | Defines module types, call types, and storage slots          | `src/types/Constants.sol`      |
