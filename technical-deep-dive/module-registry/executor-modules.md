---
description: >-
  Executor modules are external smart contracts that can trigger transactions
  from a Lab's account, enabling automation, cross-protocol integrations, and
  programmable actions without requiring
icon: up-right-from-square
---

# Executor Modules

## Executor Modules

Executor modules are smart contracts that can initiate transactions on behalf of a Lab. Unlike validator modules (which verify who can act) and fallback modules (which add new interfaces to the Lab), executor modules operate from the outside — they call into the Lab's `executeFromExecutor` function to trigger actions using the Lab's account as the sender.

This is what makes Labs programmable. An executor module can swap tokens, claim rewards, distribute funds, record data, or interact with any external contract — all as the Lab — without requiring the Lab owner to sign a UserOperation for each action. The Lab owner only needs to approve the executor's installation; after that, the executor can act within its configured scope.

### How Execution Works

The execution flow for executor modules is the inverse of the normal account flow. In a standard transaction, the Lab owner signs a UserOperation that the EntryPoint validates and executes. With an executor module, the module itself initiates the action by calling the Lab directly.

The sequence works as follows:

1. **Trigger** — An external event triggers the executor module. This could be a keeper network detecting a condition, a protocol callback after a DeFi operation, a scheduled automation, or a direct call from an authorised party.
2. **Registry check** — The executor calls `executeFromExecutor` on the Lab's TBA, passing an encoded execution mode and calldata. The Lab verifies the caller (`msg.sender`) against the ERC-7484 Module Registry to confirm it is attested as `MODULE_TYPE_EXECUTOR`.
3. **Installation check** — The Lab looks up the executor's configuration in the `ExecutorManager` storage. If the executor is not installed (`installed == false`), the call reverts with `InvalidExecutor()`. The Lab additionally verifies that its current owner matches the owner recorded at installation time — if the LabNFT has changed hands since the executor was installed, the call reverts with `ExecutorOwnerMismatch()`, so a new owner must explicitly re-approve inherited executors.
4. **Execution** — The Lab delegates to `ExecLib.execute`, which decodes the execution mode and performs the transaction from the Lab's account. The Lab is the `msg.sender` for the resulting call, meaning the target contract sees the action as coming directly from the Lab.

```solidity
function executeFromExecutor(ExecMode execMode, bytes calldata executionCalldata)
    external
    payable
    returns (bytes[] memory returnData)
```

The function currently supports `CALLTYPE_SINGLE` execution — one target address, one value, one calldata payload per call. Batch execution (`CALLTYPE_BATCH`) is planned but not yet implemented.

### Installation

Executor modules are installed through the `installModule` function, callable only via the EntryPoint (meaning the installation itself must be a validated UserOperation signed by the Lab owner). The installation requires:

* **Module type** — `MODULE_TYPE_EXECUTOR` (type ID `2`)
* **Module address** — The executor contract address
* **Initialization data** — an ABI-encoded `InstallExecutorData` struct: `{ bytes executorData }`, passed through to the executor's `onInstall`

During installation, the system performs a registry check via ERC-7484 to verify the module is attested for the executor type. The `ExecutorManager` marks the executor as installed, records the current Lab owner, and calls `onInstall` on the executor contract with the provided initialization data, allowing the executor to set up any internal state it needs.

A `ModuleInstalled` event is emitted on successful installation.

### Executor Configuration

Unlike fallback modules (which map individual function selectors to module addresses), executor modules are registered by their contract address. The `ExecutorManager` maintains a mapping from executor addresses to their configuration:

```solidity
struct ExecutorConfig {
    bool installed;         // whether the executor is active
    address ownerAtInstall; // the Lab owner who approved the installation
}
```

The configuration is intentionally minimal: an installed flag, plus a snapshot of the owner who approved the executor. The snapshot powers the owner-change guard — executors installed by a previous owner stop working the moment the LabNFT transfers.

### Executors vs Other Module Types

Understanding when to use an executor versus a fallback module is important for module developers:

**Executor modules** are the right choice when an external contract or service needs to trigger an action _from_ the Lab. The executor calls into the Lab, and the Lab performs the resulting transaction as itself. The flow is: external trigger → executor → Lab executes. Use cases include automated operations (keeper-triggered rebalancing, scheduled distributions), protocol callbacks (post-swap hooks, liquidation responses), and any scenario where the Lab needs to act without the owner being online.

**Fallback modules** are the right choice when the Lab needs to _respond_ to calls that target function selectors it doesn't natively implement. The flow is: external caller → Lab's fallback → module handles. Use cases include implementing token receiver interfaces, supporting flash loan callbacks, and adding queryable view functions.

The key distinction is directionality: executors push actions through the Lab, while fallback modules handle actions arriving at the Lab.

### Use Cases

**Automated treasury management.** An executor module connected to a keeper network (such as Chainlink Automation or Gelato) can rebalance Lab holdings, harvest yield, or execute DCA strategies on a schedule or when market conditions are met.

**Cross-protocol integration.** When a DeFi protocol needs to call back into the Lab after an operation — for example, after a swap completes or a lending position is liquidated — an executor module can handle the callback and trigger the Lab's response.

**Scheduled distributions.** A vesting or milestone-based distribution executor can release tokens from the Lab's treasury on a predetermined schedule, executing transfers as the Lab without requiring manual intervention from the owner.

**Agent-driven operations.** AI agents or research automation services can be granted executor access to perform data operations, submit transactions, or interact with protocols on the Lab's behalf within defined boundaries.

### Security Considerations

Executor modules are high-trust components. An installed executor can trigger arbitrary transactions from the Lab's account, making it functionally equivalent to a co-signer for execution purposes. Several safeguards constrain this power:

The ERC-7484 Module Registry ensures only attested executor contracts can be installed. The registry check occurs both at installation time and at every execution, so even if an executor is compromised after installation, revoking its registry attestation will prevent further actions.

Installation requires a validated UserOperation through the EntryPoint, meaning the Lab owner must explicitly approve every executor. No executor can self-install.

Lab owners can revoke an executor's access at any time via `uninstallModule`, which clears its `ExecutorManager` registration and calls the executor's `onUninstall` handler.

### Contract Reference

| Contract                 | Role                                                                        | Source                         |
| ------------------------ | --------------------------------------------------------------------------- | ------------------------------ |
| `ExecutorManager.sol`    | Manages executor installation and configuration storage                     | `src/core/ExecutorManager.sol` |
| `OnChainLab.sol` | Contains `executeFromExecutor` and `installModule` for executor type        | `src/OnChainLab.sol`   |
| `ExecLib.sol`            | Provides `execute` helper that decodes execution mode and performs the call | `src/utils/ExecLib.sol`        |
| `IERC7579Modules.sol`    | Defines the `IExecutor` interface (extends `IModule`)                       |                                |
