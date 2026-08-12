# Access Policies

Every Lab is role-gated by default: only its owner and contributors may write to the data room. An **access policy** optionally opens individual capabilities to a wider set of callers — permissionless contributions, a time-boxed submission window, or contributions gated on an onchain condition such as holding a token.

Policies are set when the Lab is created and changed with `updateLabAccessPolicy`. The onchain role model itself is untouched — see [Roles & Permissions](../../technical-deep-dive/roles-and-permissions.md) for the role hierarchy this layers on top of, and [Lab Management](lab-management.md) for creating the Lab in the first place.

***

## How a Policy Composes with Roles

A policy is a per-capability rule map that can only ever **add** access on top of membership:

```
allowed(caller, capability) =
     membership grants it              (role model — checked first)
  OR the Lab's policy rule grants it   (strictly additive fallback)
```

Three properties follow from that ordering:

* **A role grant always wins.** Owners and members never lose access, whatever the policy says. Locking the owner out of their own Lab is structurally impossible.
* **No policy means the classic behaviour.** A Lab created without an `accessPolicy` behaves exactly like a role-gated Lab, down to the error responses. Labs created before access policies existed are unaffected.
* **Permissionless is not unauthenticated.** Callers still authenticate — a Privy session or a Service Token, and any wallet can self-issue one through the [wallet-signature flow](service-tokens.md#obtaining-tokens). A policy drops the _membership_ requirement, never the _identity_ requirement.

***

## Capabilities

Four capabilities can be opened up. Each maps to the operations it guards:

| Capability             | Guards                                                                                                                | Role default       |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------- | ------------------ |
| `ADD_FILES`            | `finishCreateOrUpdateFile` with a `path` (new file), and `initiateCreateOrUpdateFile` — which only issues a presigned URL, so either capability qualifies there | Owner, Contributor |
| `MODIFY_FILES`         | `finishCreateOrUpdateFile` with a `ref` (new version) or overwriting an existing `path`, `updateFileMetadata`, `moveEntry` | Owner, Contributor |
| `DELETE_FILES`         | `deleteDataRoomFile`                                                                                                  | Owner, Contributor |
| `CREATE_ANNOUNCEMENTS` | `createAnnouncement`                                                                                                  | Owner, Contributor |

Each capability is set to one of three rule kinds:

| Kind         | Grants to                                                                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `ROLES`      | Membership only — identical to leaving the capability unconfigured. Stored policies never contain `ROLES` rules; passing it removes the rule |
| `ANYONE`     | Any authenticated caller                                                                                                                   |
| `CONDITIONS` | Any authenticated caller whose wallet satisfies the rule's `conditions` array                                                               |

`ANYONE` and `CONDITIONS` rules accept two optional qualifiers, both of which close the rule again:

| Qualifier    | Effect                                                                                                                                                                                                                                          |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openUntil`  | Unix **seconds**, compared against wall clock on every request. At and after that instant the rule falls back to `ROLES`. Must be in the future and within ten years. No background job flips anything                                           |
| `closedWhen` | A caller-independent condition array. While it evaluates false the rule grants; once it evaluates **true** the rule falls back to `ROLES` — "permissionless _until_ condition X is met". An evaluation failure counts as closed (fails closed) |

### Owner-Only Operations

These operations are hardcoded to the Lab owner and can never appear in a policy — openness cannot be escalated into administration:

| Operation                     | Notes                                                                                          |
| ----------------------------- | ---------------------------------------------------------------------------------------------- |
| `updateLabAccessPolicy`       | Changing or removing the policy itself                                                         |
| `createLab` with `accessPolicy` | Creating a Lab open requires owner proof; a bare `createLab` keeps its usual authorization     |
| `updateLabNftMetadata`        | LabNFT display metadata (name, description, image, external URL)                               |
| `generateLabImageUploadUrl`   | LabNFT image uploads                                                                           |
| `signLegalAgreement`          | Recording acceptance of a legal agreement                                                      |

### Gates That Still Apply

* **Legal agreement.** Every data-room write — including a contribution to a permissionless Lab — stays blocked until the Lab **owner** has signed the current Assignment Agreement. Opening a Lab does not bypass compliance. See [Legal Agreements](legal-agreements.md).
* **x402 tokens** keep their per-mutation scope, and that scope is checked _before_ membership and the policy. See [x402 Gateway](../x402-gateway.md).
* **Reads are unchanged.** `labs`, `labWithDataRoomAndFiles`, the activity feeds, `searchLabs`, and `listLabMembers` were public before and stay public. File confidentiality remains a matter of encryption, not of the access policy — see [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md).

***

## Set a Policy When Creating a Lab

`CreateLabInput` takes an optional `accessPolicy`. Omit it for the default role-gated Lab.

> **Authorization**: Passing `accessPolicy` restricts `createLab` to the OCL admin (LabNFT owner + multisig signers). On the Service Token path the token's wallet must be the Lab owner.

```graphql
mutation CreateOpenLab($oclId: String!, $accessPolicy: LabAccessPolicyInput) {
  createLab(input: { oclId: $oclId, accessPolicy: $accessPolicy }) {
    message
    lab {
      oclId
      shortname
    }
    error {
      message
      code
      retryable
    }
  }
}
```

**Variables:**

```json
{
  "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
  "accessPolicy": { "preset": "OPEN" }
}
```

The policy is written before the data room is registered, so retrying a failed `createLab` cannot strand a Lab with the wrong policy. A default (`GATED`) input stores no policy at all — the absence of a policy _is_ the gated state.

***

## Update a Policy

`updateLabAccessPolicy` sets the policy on an existing Lab: open it up, gate it back down with `preset: GATED`, or reconfigure individual capabilities. The input **replaces** the stored policy — it is not a patch.

> **Authorization**: Restricted to the OCL admin (LabNFT owner + multisig signers) on both auth paths.

```graphql
mutation UpdateLabAccessPolicy($oclId: String!, $input: LabAccessPolicyInput!) {
  updateLabAccessPolicy(oclId: $oclId, input: $input) {
    message
    accessPolicy {
      preset
      capabilities {
        capability
        kind
        openUntil
      }
      updatedAt
    }
    error {
      message
      code
      retryable
    }
  }
}
```

**Parameters:**

| Parameter | Type                 | Required | Description                                    |
| --------- | -------------------- | -------- | ---------------------------------------------- |
| oclId     | String               | Yes      | Canonical 32-byte oclId of the lab             |
| input     | LabAccessPolicyInput | Yes      | The complete replacement policy (not a patch)  |

Enforcement reads the stored policy through a short-lived cache of roughly 15 seconds, so a change takes effect within that window. Reading the policy back through `Lab.accessPolicy` always returns fresh state.

***

## Input Reference

`LabAccessPolicyInput` — accepted by both `createLab` and `updateLabAccessPolicy`:

| Field        | Type                          | Required | Description                                                                                                        |
| ------------ | ----------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------ |
| preset       | LabAccessPreset               | No       | `GATED` or `OPEN` — covers the common cases                                                                        |
| openUntil    | AWSTimestamp                  | No       | Shorthand: applied as `openUntil` to every rule the preset expands to. Requires `preset: OPEN`                     |
| closedWhen   | String                        | No       | Shorthand: applied as `closedWhen` to every rule the preset expands to. JSON-stringified array. Requires `preset: OPEN` |
| capabilities | \[LabCapabilityPolicyInput!]  | No       | Advanced per-capability rules, merged over the preset expansion                                                    |

`LabCapabilityPolicyInput`:

| Field      | Type              | Required | Description                                                                                     |
| ---------- | ----------------- | -------- | ----------------------------------------------------------------------------------------------- |
| capability | LabCapability     | Yes      | `ADD_FILES`, `MODIFY_FILES`, `DELETE_FILES`, or `CREATE_ANNOUNCEMENTS`                          |
| kind       | LabPolicyRuleKind | Yes      | `ROLES`, `ANYONE`, or `CONDITIONS`                                                              |
| openUntil  | AWSTimestamp      | No       | Unix seconds; `ANYONE` / `CONDITIONS` only                                                      |
| conditions | String            | No\*     | JSON-stringified access-control-conditions array. Required for `CONDITIONS`, rejected otherwise |
| closedWhen | String            | No       | JSON-stringified caller-independent condition array; `ANYONE` / `CONDITIONS` only                |

_\*`conditions` is required for `kind: CONDITIONS` and rejected for the other kinds._

The server expands the input deterministically — **preset → top-level shorthand → per-capability rules**, last write wins per capability, and a `ROLES` rule removes the capability's entry. The presets expand to:

| Preset  | Expands to                                                                                     |
| ------- | ---------------------------------------------------------------------------------------------- |
| `GATED` | `{}` — everything membership-gated. Use it to close a Lab back down                             |
| `OPEN`  | `ADD_FILES: ANYONE` — contributions only; modify, delete, and announce stay membership-gated    |

***

## Recipes

### Permissionless contributions

Any authenticated caller may add new files; edits, deletions, and announcements stay with members.

```graphql
accessPolicy: { preset: OPEN }
```

### Open until a deadline

Reverts to role-gated at the timestamp. There is no job to wait for — the instant is compared on each request.

```graphql
accessPolicy: { preset: OPEN, openUntil: 1790812800 }
```

### Open until an onchain condition is met

Contributions stay open while the condition evaluates false — for example, until a bounty contract reports itself closed. `closedWhen` must not reference `:userAddress`, because it is evaluated without a caller.

```graphql
accessPolicy: {
  preset: OPEN,
  closedWhen: "[{\"conditionType\":\"evmContract\",\"contractAddress\":\"0xBounty…\",\"chain\":\"base\",\"functionName\":\"isClosed\",\"functionParams\":[],\"functionAbi\":{\"name\":\"isClosed\",\"inputs\":[],\"outputs\":[{\"name\":\"\",\"type\":\"bool\"}],\"stateMutability\":\"view\",\"type\":\"function\"},\"returnValueTest\":{\"key\":\"\",\"comparator\":\"=\",\"value\":\"true\"}}]"
}
```

### Token-gated contributions

Only wallets holding a balance of a given ERC-20 or ERC-721 may contribute.

```graphql
accessPolicy: {
  capabilities: [{
    capability: ADD_FILES,
    kind: CONDITIONS,
    conditions: "[{\"conditionType\":\"evmBasic\",\"contractAddress\":\"0xToken…\",\"chain\":\"base\",\"method\":\"balanceOf\",\"parameters\":[\":userAddress\"],\"returnValueTest\":{\"key\":\"\",\"comparator\":\">\",\"value\":\"0\"}}]"
  }]
}
```

### Open collaboration including edits

Per-capability rules layered on top of the preset.

```graphql
accessPolicy: {
  preset: OPEN,
  capabilities: [
    { capability: MODIFY_FILES, kind: ANYONE },
    { capability: DELETE_FILES, kind: ANYONE }
  ]
}
```

### Gate a Lab back down

```graphql
mutation GateLab($oclId: String!) {
  updateLabAccessPolicy(oclId: $oclId, input: { preset: GATED }) {
    message
    accessPolicy {
      preset
      capabilities {
        capability
        kind
      }
    }
    error {
      message
      code
      retryable
    }
  }
}
```

***

## Reading a Policy

`accessPolicy` is a public field on `Lab` and `LabRef` — policies gate access, they are not secrets. Labs without a stored policy return the synthesized `GATED` default.

```graphql
query GetLabAccessPolicy($oclId: String!) {
  labWithDataRoomAndFiles(oclId: $oclId) {
    oclId
    accessPolicy {
      preset
      capabilities {
        capability
        kind
        openUntil
        conditions
        closedWhen
      }
      updatedAt
    }
  }
}
```

**Fields:**

| Field        | Type                      | Description                                                                                           |
| ------------ | ------------------------- | ----------------------------------------------------------------------------------------------------- |
| preset       | LabAccessPreset           | The preset the policy was configured with, or `null` for a custom configuration. A provenance hint     |
| capabilities | \[LabCapabilityPolicy!]!  | Non-default rules only — a capability absent from this list follows the role model                    |
| updatedAt    | AWSDateTime               | When the policy was last written                                                                      |

`conditions` and `closedWhen` come back as `AWSJSON` — the stored condition arrays.

> Render from `capabilities`, not from `preset`. `capabilities` is authoritative, and `preset` is `null` for any Lab configured per capability.

***

## Access-Control Conditions

`conditions` and `closedWhen` use the same JSON-string convention and schema as file encryption's [`accessControlConditions`](../../technical-deep-dive/data/data-privacy-and-access.md#condition-shape): an array alternating condition objects and boolean operators — `[condition, { "operator": "and" }, condition, …]`. Inside `conditions`, the placeholder `:userAddress` is substituted with the authenticated caller's wallet at request time.

Write-time validation is strict. A rejected policy fails with `VALIDATION_FAILED` and `details.reason: "INVALID_ACCESS_POLICY"`, with `details.field` naming the offender:

| Rule                                                                                                                                                     | Why                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Non-empty array, alternating condition/operator, ending on a condition                                                                                   | an empty array would deny everyone                           |
| Operators homogeneous — all `and` or all `or`, no nesting                                                                                                 | mixed chains are not supported by the evaluator              |
| `conditionType` must be `evmContract` or `evmBasic`                                                                                                      | —                                                            |
| `evmBasic.method` must be `balanceOf`, `ownerOf`, `allowance`, or `balanceOfBatch`                                                                        | evaluator ABI allowlist                                      |
| `returnValueTest.comparator` must be `=`, `!=`, `>`, `>=`, `<`, or `<=`; the numeric comparators need an integer `value`                                   | `contains` is not implemented                                |
| `chain` must be `base` or `baseSepolia`                                                                                                                  | policy conditions are pinned to the canonical chain and its testnet |
| `contractAddress` must be a valid address; `evmContract.functionAbi.name` must equal `functionName`; the ABI's `stateMutability` must be `view` or `pure` | —                                                            |
| `closedWhen` must not contain `:userAddress`                                                                                                             | it is evaluated without a caller                             |
| `CONDITIONS` requires `conditions`; `ANYONE` rejects it; `ROLES` rejects every qualifier                                                                  | —                                                            |
| At most 10 conditions per rule, 16 KB per array, 32 KB per policy document                                                                                | —                                                            |
| One entry per capability — no duplicates                                                                                                                 | —                                                            |

***

## Enforcement

1. **Order of checks** — authentication (Privy session or Service Token, x402 scope included) → membership → policy. The policy is consulted only when membership denies a caller on an _existing_ Lab; a malformed `oclId` or a Lab that does not exist is never rescued by a policy.
2. **Service tokens act as wallets on open Labs** — a valid token's wallet is treated like a user wallet, which is how autonomous agents contribute without being granted a role. On gated Labs the Service Token path keeps its owner-only semantics.
3. **A rule grants** when `openUntil` has not passed, `closedWhen` does not evaluate true, and the caller satisfies `conditions` (`ANYONE` needs no wallet check). Conditions are evaluated against live chain state on each request.
4. **Add-only means add-only** — a caller granted `ADD_FILES` but not `MODIFY_FILES` cannot write over an existing path. "Create or update" never silently becomes an overwrite.
5. **Attribution is pinned** — on a policy-granted write, `changeBy` is forced to the authenticated caller and a spoofed `changeBy` argument is ignored. Members keep their self-declared `changeBy`.
6. **Fail closed** — if a condition cannot be evaluated (RPC failure, evaluator unavailable), the request is denied with a retryable error rather than allowed.

***

## Error Responses

Access-policy failures reuse the existing error codes; the specific cause rides on `details.reason`:

| `error.code`                        | `details.reason`            | When                                                                                                                                            |
| ----------------------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `UNAUTHORIZED`                      | `UNAUTHORIZED`              | The caller is not a member and the policy does not open this capability — a gated Lab, a lapsed `openUntil`, or a `closedWhen` that has flipped |
| `UNAUTHORIZED`                      | `CAPABILITY_DENIED`         | The caller does not satisfy the rule's conditions, or an add-only caller targeted an existing path. `details.capability` names the capability     |
| `VALIDATION_FAILED`                 | `INVALID_ACCESS_POLICY`     | The policy input was rejected at write time; `details.field` pinpoints it                                                                       |
| `UPSTREAM_UNAVAILABLE` (retryable)  | `POLICY_CHECK_UNAVAILABLE`  | A condition could not be evaluated. Fails closed — retry with backoff                                                                            |
| `INTERNAL_ERROR`                    | `LAB_ACCESS_CHECK_FAILED`   | Unexpected failure inside the access check. Fails closed                                                                                        |

The first row is deliberately indistinguishable from the classic denial: a Lab whose window has lapsed returns exactly what a Lab that was never opened returns.

***

## See Also

* [Lab Management](lab-management.md) — creating a Lab, LabNFT metadata, members, DID linking
* [Files](files.md) — the upload flow and the mutations these capabilities guard
* [Roles & Permissions](../../technical-deep-dive/roles-and-permissions.md) — the onchain role model a policy layers on top of
* [Data Privacy & Access](../../technical-deep-dive/data/data-privacy-and-access.md) — the condition shape and how conditions are evaluated
* [Legal Agreements](legal-agreements.md) — the owner-signature gate that applies to every write

***
