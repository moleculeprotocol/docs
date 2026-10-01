---
description: >-
  What people build on Molecule, and the path through the docs for each: which
  API, guide and tutorial to use, in order.
icon: list-check
---

# Common Use Cases

Each use case below describes something people build on Molecule. It says who it's for, what it uses, and the order to read the docs in. Pick the one closest to your goal and follow its steps. For a single operation, like deleting a file or reading an error code, use the sidebar instead.

New to the terms (Lab, `oclId`, service token)? The [Glossary](references/glossary.md) explains each in a sentence or two. Everything below runs against **staging** (Base Sepolia, testnet funds) first. Moving to mainnet means changing a few constants: see [Running in Production](api-reference/getting-started/README.md#running-in-production).

| I want to…                                                   | Use case                                                                                  |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Push data from instruments, pipelines or CI into a Lab       | [Publish research data from a pipeline](#publish-research-data-from-a-pipeline)           |
| Keep files confidential but readable by chosen collaborators | [Share confidential data with collaborators](#share-confidential-data-with-collaborators) |
| Let an AI agent read and write into a Lab I own              | [Give an AI agent access to a Lab](#give-an-ai-agent-access-to-a-lab)                     |
| Have my coding agent do the whole Lab workflow for me        | [Let a coding agent run the workflow](#let-a-coding-agent-run-the-workflow)               |
| Show Labs, files and activity in my own app                  | [Build a dashboard over Lab data](#build-a-dashboard-over-lab-data)                       |
| Turn a Lab into a token and fund the research                | [Tokenize a Lab and raise funding](#tokenize-a-lab-and-raise-funding)                     |
| Offer a service that writes to Labs and pays per call        | [Run a pay-per-call agent service](#run-a-pay-per-call-agent-service)                     |
| Add new onchain capabilities to Labs                         | [Extend Labs with a smart contract module](#extend-labs-with-a-smart-contract-module)     |

---

## Publish research data from a pipeline

**What you end up with:** a script, cron job or CI step that uploads results into a Lab automatically, with tags and searchable text, and with no human signing in.

**For:** research teams with instruments, analysis pipelines or CI jobs that produce data. **Uses:** Labs API, a dedicated pipeline wallet, a service token.

1. **Get a Lab and a credential.** Request a `mol_` consumer credential and create a Lab, in the [Molecule app](user-guides/scientists-researchers.md#creating-a-lab) or [from code](api-reference/getting-started/create-lab-and-upload-file.md). See [Prerequisites](api-reference/getting-started/README.md#prerequisites).
2. **Give the pipeline its own wallet.** Uploading files needs the **Contributor** role, so the Lab owner grants it to the pipeline wallet. The [Agent access](api-reference/getting-started/agent-as-a-lab-contributor.md) tutorial shows the grant. It works the same for a bot as for an agent.
3. **Have the pipeline issue its own service token.** It signs a message with its wallet, so no human is involved. Choose `expiresIn` to fit the job (the default is 180 days), and renew it with `extendServiceToken` before it expires. See [Service Tokens](api-reference/labs-api/service-tokens.md).
4. **Upload each result.** Uploading is three steps: initiate, PUT, finish. Set `tags`, `categories` and `contentText` on each file so it can be found through `searchLabs`. See [Files](api-reference/labs-api/files.md) and [Metadata Best Practices](api-reference/labs-api/README.md#metadata-best-practices).
5. **Plan for failures and limits.** Retry only when `retryable` is `true`, and log each `requestId`. Keep batch sizes within the [Rate & Query Limits](api-reference/rate-limits.md). See [Errors](api-reference/errors.md#retrying).

## Share confidential data with collaborators

**What you end up with:** files that are encrypted before they leave your machine and that only people holding a role on the Lab can decrypt. Access can be granted or revoked without re-encrypting the files.

**For:** scientists with unpublished or sensitive data, and the developers building for them. **Uses:** Labs API, local AES-256-GCM encryption, `AccessResolver` roles.

1. **Understand the model.** The backend stores only ciphertext. Before it releases a file key, it checks the file's access conditions against live onchain state. See [Data Privacy & Access](technical-deep-dive/data/data-privacy-and-access.md).
2. **Upload the file encrypted.** Get a data key, encrypt the file locally, and attach access conditions that allow the Lab owner and anyone with a role. See [Upload an encrypted file](api-reference/getting-started/upload-encrypted-file.md).
3. **Grant roles to the people who should read it.** **Viewer** can decrypt. **Contributor** can also upload. Grant roles from the app ([How Invites Work in the App](technical-deep-dive/roles-and-permissions.md#how-invites-work-in-the-app)) or onchain with `grantRole`. You can set an expiry on each grant. See the [Capability Matrix](technical-deep-dive/roles-and-permissions.md#capability-matrix).
4. **Collaborators decrypt with their own wallet.** `decryptDataKey` returns the key only if their role is still active. See [Verify by decrypting it](api-reference/getting-started/upload-encrypted-file.md#step-5-verify-by-decrypting-it).
5. **Revoke access when it should end.** `revokeRole` takes effect on the next decrypt. See [AccessResolver](references/contracts/accessresolver.md).

## Give an AI agent access to a Lab

**What you end up with:** an AI agent with its own wallet and its own time-limited role on a Lab that a human owns. It reads the Lab's data, runs its analysis and writes findings back as new files.

**For:** teams running research agents, such as [BioAgents](user-guides/developers-ai-agents.md#building-with-bioagents), on real Lab data. **Uses:** Labs API, `AccessResolver.grantRole` with `isAgent = true`, a service token the agent issues itself.

1. **The agent reports its wallet address, and the owner grants a role.** Grant **Viewer** for read-only access and **Contributor** if the agent writes back. Set `isAgent = true` and an expiry that matches the agent's session. See [Agent access](api-reference/getting-started/agent-as-a-lab-contributor.md).
2. **The agent issues its own service token**, with a lifetime that matches its role grant. See [Step 3 of Agent access](api-reference/getting-started/agent-as-a-lab-contributor.md#step-3-the-agent-self-issues-a-service-token).
3. **The agent reads, analyses and writes back.** It lists files with `labWithDataRoomAndFiles`, decrypts what its role allows, and uploads its results. The [Agent one-pager](api-reference/getting-started/for-agents.md) puts the whole flow on one page, ready to paste into the agent's system prompt.
4. **Revoke access when the job is done**, or let the grant expire. See [Revoking the agent](api-reference/getting-started/agent-as-a-lab-contributor.md#revoking-the-agent).

For the design behind this, see [Deploying an AI Research Agent](user-guides/developers-ai-agents.md#deploying-an-ai-research-agent).

## Let a coding agent run the workflow

**What you end up with:** Claude Code, Codex, Cursor or another agent tool that creates Labs, uploads public or encrypted files and manages roles when you ask it to, without you writing API calls.

**For:** developers and scientists who would rather describe the task than script it. **Uses:** the Molecule Skill plugin (a skill plus a local MCP server), with x402 for paid calls.

1. **Check it's the right tool.** The Skill is for agents that _act_ on Labs. To chat about Molecule instead, use MIRA. See [Which Tool Do I Need?](ai-tooling/README.md)
2. **Install the plugin and choose a wallet backend.** See [Getting the Plugin](ai-tooling/molecule-skill.md#getting-the-plugin) and [Wallet Backends](ai-tooling/molecule-skill.md#wallet-backends).
3. **Configure the environment.** Set staging or production, your `mol_` credential, and fund the wallet with gas and USDC for x402. See [Configuration](ai-tooling/molecule-skill.md#configuration).
4. **Ask for the outcome.** For example: "Create a Lab and upload `results.csv` encrypted." The agent works through the Skill's phases one after another. See [What the Skill Does](ai-tooling/molecule-skill.md#what-the-skill-does).

## Build a dashboard over Lab data

**What you end up with:** a frontend, explorer or analytics tool that lists Labs, shows their files and activity, and searches across them.

**For:** frontend developers, ecosystem explorers and analysts. **Uses:** Labs API read queries. Public data needs only a consumer credential, with no wallet and no service token.

1. **List Labs and page through them** with the public `labs` query. See [List All Projects](api-reference/labs-api/browse-and-search.md#list-all-projects).
2. **Open a single Lab** with its files and download URLs. See [Get Single Project with Files](api-reference/labs-api/lab-management.md#get-single-project-with-files).
3. **Add activity feeds**: per Lab ([Project Activity Feed](api-reference/labs-api/browse-and-search.md#project-activity-feed)), across all Labs ([Global Activity Feed](api-reference/labs-api/browse-and-search.md#global-activity-feed)) and onchain ([Onchain Activity Feed](api-reference/labs-api/browse-and-search.md#onchain-activity-feed)).
4. **Add search** across every Lab's files with `searchLabs`. See [Searching Labs](api-reference/labs-api/browse-and-search.md#searching-labs).
5. **Generate types and respect the limits.** Introspection is off in production, so generate types against staging. Always pass paging arguments. See [Getting the schema](api-reference/getting-started/README.md#getting-the-schema) and [Rate & Query Limits](api-reference/rate-limits.md).

For live onchain state the indexer doesn't cover, such as balances or allowances, read the contracts directly. Addresses are on [Supported Networks & Contracts](references/contracts/README.md).

## Tokenize a Lab and raise funding

**What you end up with:** an ERC-20 token tied to your Lab, issued under a signed membership agreement, which you can use to fund the research and grow a community around it.

**For:** scientists and Lab owners who want funding beyond grants and venture capital. **Uses:** Tokenization API, the `OclTokenizer` contract on Base.

1. **Understand what you're creating.** The token is a fractional claim tied to the Lab. The Lab's assets never leave your control, and control of the token's supply follows the LabNFT. See [Lab Tokenization](technical-deep-dive/onchain-lab.md#lab-tokenization).
2. **Generate and sign the membership agreement.** Use `generateOclMembershipAgreement`, then `getOclTermsMessage`, then sign the terms with a plain `personal_sign`. See the [Tokenization API](api-reference/tokenization-api.md).
3. **Tokenize onchain.** The Lab's current owner calls `OclTokenizer.tokenize()` and pays gas only. To attach an ERC-20 you already have instead, use `attachToken`. Each Lab can be tokenized once. See [Tokenizer](references/contracts/tokenizer.md) and the [token contract](references/contracts/ipt.md).
4. **Raise funds and grow the community.** See [Raising Funds](user-guides/scientists-researchers.md#raising-funds) for researchers and [Funders](user-guides/investors.md) for the backer's side.
5. **Plan the path to a company.** See the [Coin-to-Company Model](legal-framework/rwa-equity.md).

## Run a pay-per-call agent service

**What you end up with:** an agent or tool that writes to Labs whose owners have added its wallet as a member. It pays in USDC for each call instead of holding long-lived credentials, and every write is recorded under the wallet that paid.

**For:** builders of third-party tools and agents that serve external users. **Uses:** the x402 Gateway (USDC on Base).

1. **Check x402 fits.** Use it when you have no service token set up in advance, or when you want to pay per call. See [When to use x402](api-reference/x402-gateway.md#when-to-use-x402).
2. **Point at the gateway and fund the payer wallet.** Staging uses Base Sepolia USDC from the Circle faucet. See [Gateway base URLs](api-reference/x402-gateway.md#gateway-base-urls).
3. **Make sure the payer wallet has a role on each target Lab.** Paying doesn't grant a role. Content writes need **Contributor**, and `createLab` and LabNFT-metadata changes need **Owner**. Without the role, the call returns `200` with `UNAUTHORIZED` and you are still charged. Check with the public, free `listLabMembers` query before signing. See the warning in [Reading the 402 challenge](api-reference/x402-gateway.md#reading-the-402-challenge).
4. **Handle the payment challenge.** Read the 402 response, sign the payment and retry. The gateway then issues a short-lived service token that covers only that one mutation. See [Payment Flow](api-reference/x402-gateway.md#payment-flow) and [Agent Usage Pattern](api-reference/x402-gateway.md#agent-usage-pattern).
5. **Handle both ways a call can fail.** The gateway can reject the payment, or the mutation it forwards can fail. See [x402 Gateway errors](api-reference/errors.md#x402-gateway-errors).

## Extend Labs with a smart contract module

**What you end up with:** a module, attested in the registry, that any Lab owner can install. It can distribute funds, enforce agent spending limits, add governance, or add new functions to a Lab.

**For:** Solidity developers. **Uses:** the Lab's modular smart account, the ERC-7484 registry, Foundry.

1. **Pick the module type.** An **executor** triggers actions from the Lab's account. A **fallback** adds new functions to the Lab. See [Module Registry](technical-deep-dive/module-registry/README.md), [Executor Modules](technical-deep-dive/module-registry/executor-modules.md) and [Fallback Modules](technical-deep-dive/module-registry/fallback-modules.md).
2. **Learn how the account executes calls.** Modules are called as regular external calls, never through `delegatecall`. See [Architecture](technical-deep-dive/architecture.md).
3. **Build and test against Base Sepolia.** The registry, factory and validator addresses are on [Supported Networks & Contracts](references/contracts/README.md). For the development cycle, see [Extending Labs with Smart Contract Modules](user-guides/developers-ai-agents.md#extending-labs-with-smart-contract-modules).
4. **Get it attested, then installed.** Submit the module to the Molecule team for attestation through [Molecule Discord](https://t.co/L0VEiy4Bjk). Once attested, a Lab owner can install it.

---

Is your use case missing? Ask on the [Molecule Discord](https://t.co/L0VEiy4Bjk). Describe what you're building and we'll point you to the right place.

{% include ".gitbook/includes/support.md" %}
