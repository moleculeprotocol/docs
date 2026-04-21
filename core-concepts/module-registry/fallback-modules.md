---
description: >-
  Fallback modules extend OnChain Lab functionality by handling function calls
  that are not natively defined on the account contract, dispatched through the
  Solidity fallback function via selector-based
icon: square-binary
---

# Fallback Modules

## Fallback Modules

Fallback modules extend the native functionality of an OnChain Lab by handling calls to function selectors that are not defined on the account contract itself. When a transaction targets a selector that the Lab does not recognize, the Solidity `fallback()` function intercepts it and routes the call to a registered external module based on a per-selector configuration managed by the `SelectorManager`.

This mechanism allows Labs to support new interfaces, respond to new protocol interactions, and integrate with external systems — all without modifying the core account logic.

### How It Works

Every OnChain Lab inherits from `SelectorManager`, which maintains a mapping from 4-byte function selectors to a `SelectorConfig` struct:

solidity

```solidity
struct SelectorConfig {
    IHook hook;       // Hook address (or sentinel value)
    address module;   // Fallback module contract address
    CallType callType; // Execution mode: CALLTYPE_SINGLE or CALLTYPE_DELEGATECALL
}
```

When an external call arrives at the Lab with a selector that does not match any native function, the `fallback()` handler executes the following sequence:

1. **State increment** — The Lab increments an internal state counter to track executions and support anti-fraud mechanisms.
2. **Selector lookup** — The handler reads the `SelectorConfig` for `msg.sig` from the selector storage slot.
3. **Installation check** — If the hook field equals `HOOK_MODULE_NOT_INSTALLED` (address zero), the selector has no registered module and the call reverts with `InvalidSelector()`.
4. **Access control** — If the hook field equals `HOOK_ONLY_ENTRYPOINT`, only the ERC-4337 EntryPoint contract may invoke the selector. Direct calls from any other address revert with `InvalidCaller()`.
5. **Registry verification** — The module address is checked against the ERC-7484 Module Registry to confirm it is an attested, approved fallback module (`MODULE_TYPE_FALLBACK`).
6. **Dispatch** — The call is forwarded to the module using one of two execution modes depending on the configured `CallType`.

### Call Types

Fallback modules support two execution modes, each suited to different use cases:

**CALLTYPE\_SINGLE (0x00)** — The call is forwarded to the module contract as a standard external call using the ERC-2771 trusted forwarder pattern. The Lab appends the original `msg.sender` address to the calldata before calling the module, allowing the module to identify the true caller even though the call is proxied through the Lab. This is the recommended mode for modules that maintain their own storage.

**CALLTYPE\_DELEGATECALL (0xFF)** — The call is executed via `delegatecall`, meaning the module's code runs in the context of the Lab's storage. This enables modules to directly read and modify Lab state, but must be used with extreme care to avoid storage slot collisions. Delegatecall modules skip the `onInstall` lifecycle callback since they operate within the Lab's own storage context.

### Installation

Fallback modules are installed through the `installModule` function on the Lab account, callable only via the EntryPoint (ensuring the operation has been validated). The installation requires:

* **Module type** — `MODULE_TYPE_FALLBACK` (type ID `3`)
* **Module address** — The address of the fallback module contract
* **Initialization data** — At least 24 bytes encoding the function selector (4 bytes), the hook address (20 bytes), and the selector-specific configuration data (which includes the `CallType` byte followed by any module-specific init parameters)

During installation, the system performs a registry check via ERC-7484 to verify the module is attested for the fallback type. The `SelectorManager` then stores the configuration and emits a `ModuleInstalled` event. If no hook is explicitly provided (hook address is zero), the system defaults to `HOOK_ONLY_ENTRYPOINT`, restricting invocation to validated UserOperations only.

### Hook Integration

Each fallback selector can optionally be associated with a hook module that executes before the fallback call is dispatched. Hooks provide an additional control layer for enforcing constraints such as spending limits, access policies, or audit logging on a per-selector basis.

Hook support in the current implementation uses sentinel values to indicate the hook state for each selector:

| Sentinel Value           | Constant                    | Meaning                                |
| ------------------------ | --------------------------- | -------------------------------------- |
| `address(0)`             | `HOOK_MODULE_NOT_INSTALLED` | No module registered for this selector |
| `address(1)`             | `HOOK_MODULE_INSTALLED`     | Hook module is installed and active    |
| `address(0xFFfF...FfFf)` | `HOOK_ONLY_ENTRYPOINT`      | No hook; only EntryPoint may call      |

> **Note:** Hook execution logic is currently defined in the architecture but not yet active. The `HookManager` is a stub awaiting full implementation. When hooks are fully supported, they will run pre-execution checks before fallback dispatch.

### Use Cases

Fallback modules enable OnChain Labs to support capabilities that are not part of the core account contract. Example use cases include:

* **Token receiving** — Implementing `onERC721Received`, `onERC1155Received`, or similar callback interfaces so the Lab can receive NFTs and multi-token transfers
* **Flash loan participation** — Implementing flash loan callback interfaces to allow Labs to act as flash loan receivers
* **Protocol integrations** — Supporting interaction interfaces required by external DeFi protocols, governance systems, or marketplace contracts
* **Custom query interfaces** — Exposing read-only view functions that aggregate or compute data from the Lab's state for external consumers

### Security Considerations

Fallback modules are security-critical because they can execute arbitrary logic in response to any unrecognized function call on the Lab. Several safeguards are built into the architecture:

The ERC-7484 Module Registry provides attestation-based trust. Every fallback module must be attested as type `MODULE_TYPE_FALLBACK` before it can be dispatched, ensuring only reviewed and approved modules can handle calls.

The default hook sentinel (`HOOK_ONLY_ENTRYPOINT`) ensures that when no explicit hook is configured, fallback selectors can only be invoked through the EntryPoint — meaning every call must pass UserOperation validation first. This prevents unauthorized contracts or EOAs from directly triggering fallback logic.

Delegatecall modules operate in the Lab's storage context and require particular caution. A malicious or buggy delegatecall module could corrupt Lab state, modify ownership, or drain funds. Only thoroughly audited modules should be installed with `CALLTYPE_DELEGATECALL`.

The state counter increment on every fallback invocation provides an additional anti-replay and execution tracking mechanism.

### Contract Reference

| Contract                 | Role                                                                 | Source                         |
| ------------------------ | -------------------------------------------------------------------- | ------------------------------ |
| `SelectorManager.sol`    | Manages per-selector fallback configuration and storage              | `src/core/SelectorManager.sol` |
| `Modular_OnChainLab.sol` | Contains the `fallback()` handler and `installModule` logic          | `src/Modular_OnChainLab.sol`   |
| `ExecLib.sol`            | Provides `doFallback2771Call` and `executeDelegatecall` helpers      | `src/utils/ExecLib.sol`        |
| `HookManager.sol`        | Hook lifecycle management (stub — not yet active)                    | `src/core/HookManager.sol`     |
| `Constants.sol`          | Defines module types, call types, sentinel values, and storage slots | `src/types/Constants.sol`      |
