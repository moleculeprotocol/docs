# 🔐 Authentication

All Molecule APIs require authentication. This page covers how to obtain credentials, which headers each API expects, and the specific authentication model for the Labs API.

You hold two secrets, and they do different jobs. Sending one where the other belongs fails in ways the error message doesn't explain, so keep this table in mind. If any term on this page is unfamiliar, the [Glossary](../references/glossary.md) defines it in a sentence.

| | **Consumer credential** (`mol_usr-…`) | **Service token** (a JWT) |
| -- | -- | -- |
| Answers | *Which application is calling?* | *Which wallet is acting, so what may it do?* |
| Header | `Authorization`, with **no `Bearer` prefix** | `X-Service-Token` |
| Checked | At the edge, on every request, before your operation runs | Per mutation, against the wallet's onchain role on the lab |
| Needed for | Every request your code makes (signed-in browser sessions use a Privy token instead); reads need only this | Write mutations, together with the consumer credential |
| How you get it | Create it on the **API credentials** page of the Labs app, see [Obtaining API access](#obtaining-api-access) | Sign a message with your wallet, see [Obtaining a Service Token](#obtaining-a-service-token) |

They are not alternatives. A read-only integration needs only the credential. An agent that writes to a lab needs both: the credential to get through the door, the token to be allowed to act.

## Obtaining API access

### Create a credential

1. Sign in to the Labs app: [app.molecule.xyz](https://app.molecule.xyz) for production, or [sandbox.app.molecule.xyz](https://sandbox.app.molecule.xyz) for the [sandbox](sandbox.md), where every tutorial in these docs runs. **Credentials are per environment**: a sandbox credential doesn't authenticate against production, and the other way round.
2. Open the **API credentials** page from the navigation. In production the menu entry may still read **API Keys**.
3. Click **Create**, give the credential a name you'll recognise later (optional, at most 64 characters, for example `Data sync script`), and pick how long it should last: 7, 30, 90 days, 1 year, or a custom date. The default is 90 days.
4. The first time, and again whenever the terms change, tick the box to accept the API Terms of Service and the Acceptable Use Policy. See [Accepting the API terms](#accepting-the-api-terms).
5. **Copy the credential now.** It is shown exactly once. Molecule stores only a salted hash of it, so a lost credential can't be recovered or shown again: create a new one and revoke the lost one.

What you copy is a single string:

```
mol_usr-<id>_<secret>
```

The `usr-<id>` part is the credential's public identifier, the `consumerId`. It's what the Labs app and `myConsumerCredentials` list, and what you pass to revoke one. The `<secret>` after it is never returned again. Still, treat the **whole string** as the secret: send it whole, store it whole, and never split it up.

Send it as the `Authorization` header value directly, with **no `Bearer` prefix**:

```bash
Authorization: mol_usr-<id>_<secret>
```

Adding `Bearer` in front of a consumer credential makes the request fail authentication. `Bearer` belongs to the Privy session path [below](#protected-mutations-write-operations).

Write mutations also need a service token, which you get by signing a message with your wallet. See [Service Tokens](labs-api/service-tokens.md#obtaining-a-token), or [Step 1 of Create a lab and upload a public file](getting-started/create-lab-and-upload-file.md#step-1-get-a-service-token) for the runnable version.

{% hint style="info" %}
**Partner credentials.** Integrations under a partner agreement can get a credential with a higher, separately agreed budget, managed by Molecule rather than from the Labs app. Its consumerId has no `usr-` prefix. If you need limits above what a personal credential allows, ask the team (see the help box at the bottom of this page).
{% endhint %}

### Accepting the API terms

No credential is created until you've accepted the current **Molecule API Terms of Service** and **Acceptable Use Policy**:

* [API Terms of Service](https://molecule.xyz/api-terms-of-service)
* [API Acceptable Use Policy](https://molecule.xyz/api-acceptable-use-policy)

You accept them once, the first time you create a credential, and again only when Molecule publishes a new version. Accepting a new version doesn't affect the credentials you already have: they keep working. Listing and revoking credentials never ask for acceptance, so you can always see and remove a credential whatever you've accepted.

In the Labs app this is the checkbox on the create form. This is what the Labs app does:

1. It calls `myApiTermsStatus`. If `isCurrentVersionAccepted` is `true`, it calls `issueConsumerCredential` **without** `acceptedTermsVersion`.
2. Otherwise, it shows every document in `documents` (each `{ type, version, url }`) next to an unticked checkbox, which you tick yourself.
3. Only then does it call `issueConsumerCredential` with `acceptedTermsVersion` set to the `termsVersion` it showed you. That call records your acceptance.

If the user hasn't accepted the current version and the call is made without `acceptedTermsVersion`, `issueConsumerCredential` refuses with `FAILED_PRECONDITION`, `details.reason: TERMS_ACCEPTANCE_REQUIRED`, and the same information in `details.termsVersion`, `details.documents` and `details.hint`. Nothing is issued and nothing is recorded.

{% hint style="warning" %}
**Never send `acceptedTermsVersion` on every call.** The argument is the user's acceptance, and a client that fills it in automatically records an acceptance nobody gave. On the normal path it is omitted. Send it only right after showing the documents and getting the user's tick, and only with the version you showed. Any other value, including an older version the user accepted before, is refused with `TERMS_ACCEPTANCE_REQUIRED`.
{% endhint %}

### Managing your credentials

Everything happens on the **API credentials** page of the Labs app.

* **List.** The page shows your credentials that you haven't revoked, newest first, with their name, creation date and expiry. The secret is never shown again.
* **Expiry.** Every personal credential expires: after 90 days by default, and at most 365 days after creation (at least 1 day). An expired credential stops authenticating requests. Create a new one before it expires; it can't be extended.
* **At most 5 active credentials.** Give each script or agent its own credential, so that revoking one doesn't take down the others. An expired credential no longer counts toward the 5. It stays on the list until you revoke it or create your next credential, which clears expired ones away.
* **Revoke.** Revoking takes effect on the very next request made with that credential. It can't be undone, by you or by Molecule. Revoke immediately if a credential may have leaked.
* **No rotate button: create, switch, revoke.** To replace a credential, create a new one, move your code to it, then revoke the old one. You control both ends, so there's no need to coordinate a cut-over with anyone, and two credentials can be live side by side while you switch.

### Limits on personal credentials

Personal (`usr-`) credentials have a lower [cost budget](rate-limits.md#budgets) than partner credentials. The budget belongs to **you**, not to the credential: every credential you create draws on the same window, so creating another credential, or revoking one and creating a replacement, doesn't add budget. Once rate limits are enforced (see [Rate & Query Limits](rate-limits.md)), requests over the budget get [`RATE_LIMITED`](errors.md#rate-limited-requests); wait `details.retryAfterSeconds` before retrying. The same page has the current numbers and how requests are priced.

### Credential operations reference

The Labs app calls four GraphQL fields to do all of the above. They're listed here so you recognise them in the schema. They need a signed-in session: `Authorization: Bearer <Privy token>` plus `x-wallet-address`, the same as the [Privy path](#protected-mutations-write-operations). That session is issued to a browser at sign-in, so use the Labs app to manage credentials rather than scripting these calls.

Two rules apply to all four:

* **A service token can't manage credentials.** A request that carries `X-Service-Token` is refused with `UNAUTHENTICATED`, `details.reason: SERVICE_TOKEN_NOT_ACCEPTED`, even alongside a valid session. A machine holding a token must not be able to mint credentials that outlive it.
* **A consumer credential can't manage credentials either.** A request without a signed-in session gets `UNAUTHENTICATED`, `details.reason: AUTH_FAILED`.

#### `issueConsumerCredential`

{% include "../.gitbook/includes/api/mutation-issueconsumercredential.md" %}

The credential string is in `credential`, in this response only. Failures: `VALIDATION_FAILED` with `details.field` `label` or `expiresInDays`; `CONFLICT` with `details.reason: CREDENTIAL_QUOTA_EXCEEDED` and `details.limit` when you already have 5 active credentials; `FAILED_PRECONDITION` with `details.reason: TERMS_ACCEPTANCE_REQUIRED` (see [Accepting the API terms](#accepting-the-api-terms)).

#### `myConsumerCredentials`

{% include "../.gitbook/includes/api/query-myconsumercredentials.md" %}

#### `revokeConsumerCredential`

{% include "../.gitbook/includes/api/mutation-revokeconsumercredential.md" %}

A `consumerId` that doesn't exist and one that belongs to someone else both get `NOT_FOUND` with `details.reason: CREDENTIAL_NOT_FOUND`, so the answer never confirms that another user's credential exists.

#### `myApiTermsStatus`

{% include "../.gitbook/includes/api/query-myapitermsstatus.md" %}

## Authentication Headers

| API                                       | Required Headers                                                                                                  | Example                                                                                                                       |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Labs API (queries)**                    | `Authorization`                                                                                                   | `Authorization: mol_<consumerId>_<secret>`                                                                                   |
| **Labs API (mutations, service token)**   | <p><code>Authorization</code><br><code>X-Service-Token</code></p>                                                 | <p><code>Authorization: mol_&lt;consumerId&gt;_&lt;secret&gt;</code><br><code>X-Service-Token: YOUR_SERVICE_TOKEN</code></p>        |
| **Labs API (mutations, Privy user)**      | <p><code>Authorization</code><br><code>x-wallet-address</code></p>                                                | <p><code>Authorization: Bearer PRIVY_TOKEN</code><br><code>x-wallet-address: 0x…</code></p>                                  |
| **Tokenization API**                      | `Authorization`                                                                                                   | `Authorization: mol_<consumerId>_<secret>`                                                                                   |

> **No `Bearer` prefix on consumer credentials.** `mol_<consumerId>_<secret>` goes directly in the `Authorization` header. Only a Privy user token uses `Authorization: Bearer <token>`.

***

## Labs API Authentication

The Labs API has different authentication requirements depending on the operation type:

> **Rule of thumb**: **reads are public** (consumer credential only). Write **mutations** are authenticated, and most accept **either** a Service Token **or** a Privy user session — pick whichever fits your caller. The exceptions are called out below: the two Service Token lifecycle mutations are service-token-only, and `generateServiceToken` bootstraps a token with a wallet signature or a Privy session.

Summary of the model:

* **Reads are public**: consumer credential only for the queries listed below.
* **Write mutations are authenticated, with two interchangeable paths**: consumer credential plus **either** `X-Service-Token` (machine callers — services, bots, agents) **or** `Authorization` + `x-wallet-address` (Privy user session — browser and app callers). Authorization is then evaluated against the caller's identity either way.
* **Exceptions**: `extendServiceToken` and `revokeServiceToken` accept **only** a Service Token. `generateServiceToken` accepts **only** a consumer credential plus a wallet signature or a Privy session, since it mints the token in the first place.
* **A Service Token is bound to a wallet, not to a lab.** It carries the wallet's identity; what it may do on a given lab is resolved per request from that wallet's onchain role. See [What a Service Token actually authorizes](#what-a-service-token-actually-authorizes).
* File-level access control is handled via Molecule's Onchain-Verified Envelope Encryption, not query authentication — see [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md).

### Public Queries (Read-Only)

The Labs read surface is public — these queries need only a consumer credential:

- `labs` - List all labs with pagination
- `labWithDataRoomAndFiles` - Get lab details and files
- `labActivity` - Get the file-event activity feed for a lab
- `activities` - Get the global file-event activity feed
- `dataRoomFile` - Get file by path
- `searchLabs` - Search across labs and files
- `fileCategoriesAndTags` - List valid file categories and their tags
- `getServiceSignInMessage` - Get the message a service signs to obtain a token
- `getDidLinkStatus` - Get background DID-linking status for a lab
- `onChainActivity` - Onchain event feed for a lab or wallet
- `listLabMembers` - List a lab's members

```bash
Authorization: YOUR_CONSUMER_CREDENTIAL
```

Sending `X-Service-Token` on a public query is unnecessary, and sending an empty one is worse than omitting the header entirely.

### Protected Mutations (Write Operations)

All write mutations require a **consumer credential** plus proof of caller identity. For most mutations there are **two interchangeable ways** to prove identity — the resolver accepts a Service Token if one is present, and otherwise falls back to authenticating the Privy user:

**Option 1 — Service Token** (services, bots, agents, CI/CD):

```bash
Authorization: YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

**Option 2 — Privy user session** (browser and app callers acting as a signed-in user):

```bash
Authorization: Bearer YOUR_PRIVY_TOKEN
x-wallet-address: YOUR_WALLET_ADDRESS
```

Either way, the caller still has to be authorized for the target lab, and the check is the same on both paths: the wallet's onchain role on that lab (LabNFT owner, authorized multisig signer, or an active Contributor/Viewer grant on `AccessResolver`). Supplying neither path returns an `UNAUTHENTICATED` error with `details.reason` `NO_AUTH`, naming both.

**Mutations accepting either path:**

| Mutation | Minimum role | Notes |
| -------- | ------------ | ----- |
| `createLab` - Create a lab (data room) for an onchain lab (OCL) | Owner of the OCL | 💳 also pay-per-call via [x402](x402-gateway.md) |
| `initiateCreateOrUpdateFile` - Initiate file upload | Contributor | 💳 also pay-per-call via [x402](x402-gateway.md) |
| `finishCreateOrUpdateFile` - Complete file upload | Contributor | 💳 also pay-per-call via [x402](x402-gateway.md) |
| `updateFileMetadata` - Update file metadata | Contributor | |
| `deleteDataRoomFile` - Delete a file | Contributor | |
| `moveEntry` - Move a file or folder | Contributor | |
| `updateLabNftMetadata` - Update LabNFT display metadata | **Owner only** | |
| `generateLabImageUploadUrl` - Presigned URL for a LabNFT image | **Owner only** | |
| `generateDataEncryptionKey` - Generate a standalone data encryption key | Authenticated, no role | Takes no `oclId`, so there is no lab to check against. 💳 also pay-per-call via [x402](x402-gateway.md) |
| `decryptDataKey` - Decrypt a file's data key | **Viewer**, *and* the file's own conditions | Two gates: the Viewer check on the lab, then a live onchain evaluation of the file's `accessControlConditions`. 💳 also pay-per-call via [x402](x402-gateway.md) |

The Owner passes every check; a Contributor passes Contributor and Viewer checks. Full capability matrix: [Roles & Permissions](../technical-deep-dive/roles-and-permissions.md).

**Service-Token-only mutations** — these manage token lifecycle and reject Privy sessions:

- `extendServiceToken` - Extend service token expiration
- `revokeServiceToken` - Revoke a service token

```bash
Authorization: YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

Both are scoped to the caller's **own** tokens: the token presented must own the `tokenId` it names, so one caller cannot extend or revoke another's. A `tokenId` belonging to someone else returns the same `NOT_FOUND` as one that does not exist.

> **`generateServiceToken` is the bootstrap exception**, in the opposite direction: it mints a Service Token, so it accepts *only* a consumer credential plus either a wallet signature or a Privy session — not a pre-existing Service Token. See [Obtaining a Token](labs-api/service-tokens.md#obtaining-a-token).

> **Pay-per-call alternative.** Mutations tagged 💳 above can also be called through the [x402 Gateway](x402-gateway.md), which settles a USDC payment on Base per request and mints a short-lived service token on the fly — no long-lived credentials required. Useful for autonomous AI agents and third-party tools that pay for users.

### Obtaining a Service Token

Self-service, two calls, no human in the loop. Full reference with parameters and failure modes: [Service Tokens](labs-api/service-tokens.md#obtaining-a-token). Runnable: [Step 1 of Create a lab and upload a public file](getting-started/create-lab-and-upload-file.md#step-1-get-a-service-token).

1. **`getServiceSignInMessage(walletAddress, serviceName)`** — a public query returning the message to sign, plus the `expiresAt` of the nonce embedded in it.
2. **Sign it verbatim** with the wallet, as a plain personal message (EIP-191 `personal_sign` — **not** typed data). Re-wording or re-formatting the string breaks verification.
3. **`generateServiceToken(serviceName, walletAddress, messageSignature, expiresIn)`** — returns the JWT to send as `X-Service-Token`, plus a `tokenId` for lifecycle operations.

> **The sign-in message is single-use and short-lived — fetch a fresh one before every signing.** It embeds a server-issued nonce and an expiry, so it is **not** deterministic and a signature over it cannot be replayed or cached. The nonce is valid for **10 minutes**, is consumed by the first successful `generateServiceToken`, and there is one outstanding nonce per `(walletAddress, serviceName)` pair — fetching a new message supersedes the previous one. Never reconstruct the string client-side; sign exactly what the query returned. Failure reasons: [Obtaining a Token](labs-api/service-tokens.md#obtaining-a-token).

| `expiresIn` | Value |
| ----------- | ----- |
| Default when omitted | `180d` |
| Format | `<integer><unit>`, unit one of `s` `m` `h` `d` `w` `M` `y` (e.g. `"30d"`, `"720h"`, `"6M"`) |
| Minimum | 1 hour |
| Maximum | 2 years |

**Validate this before you send it.** `expiresIn` is not checked by the resolver: a value outside those bounds, or in another format, fails inside token generation and comes back as `INTERNAL_ERROR` with `details.reason: TOKEN_GENERATION_FAILED` and a masked message, not as `VALIDATION_FAILED`. `INTERNAL_ERROR` is flagged `retryable: true`, but this one is permanent — fix the value rather than retrying.

Issuance is **not** gated on holding a role on any lab — any wallet can mint a token for itself. The role is what makes the token useful.

### What a Service Token actually authorizes

A Service Token is **wallet-bound, not lab-bound.** It says "this wallet is calling"; it does not carry a list of labs.

On every request, the API resolves what the token's wallet may do on the lab named in the call from that wallet's **live onchain role**. This has three practical consequences:

* **One token works across every lab the wallet has a role on.** You do not issue a token per lab.
* **A role granted after the token was issued takes effect without re-issuing it.** Likewise a revoked role stops the token on that lab immediately, while leaving it valid elsewhere.
* **A token for a wallet with no role authenticates but cannot write.** You will see `UNAUTHENTICATED` become `UNAUTHORIZED`: the caller is known, just not permitted.

This is why an agent can be handed access to a lab it does not own — the human grants the agent's wallet a role, and the agent's own token starts working on that lab. See [Agent access](getting-started/agent-as-a-lab-contributor.md).

> Because role state reaches the API through an event indexer, there is a short window after a role grant confirms onchain in which a write can still return `UNAUTHORIZED`. Retry with backoff; re-issuing the token does not help.

### The three wallets, side by side

A working integration has up to three addresses in play at once, and they are not interchangeable. Sending the wrong one is the most common way a request fails for a reason the error message does not explain.

| | **Owner wallet** | **Agent wallet** | **OCL account** (the Lab's own wallet) |
| -- | -- | -- | -- |
| What it is | The human's wallet — typically a Privy embedded wallet created at email sign-in, but any wallet that holds the LabNFT | An EOA belonging to the agent, generated by the agent and never shared | The Lab itself: an [ERC-6551 Token Bound Account](../technical-deep-dive/onchain-lab.md) permanently bound to the LabNFT |
| Who holds the private key | The human (Privy custodies the embedded case) | The agent, and only the agent | **Nobody.** It has no key of its own — its authority derives from whoever currently owns the LabNFT |
| How it gets its rights | Implicitly: holding the LabNFT makes it **Owner** | An explicit onchain grant of **Contributor** or **Viewer** from the Owner | It is the lab — rights are resolved *against* it, not held by it |
| How it authenticates | `Authorization: Bearer <Privy token>` + `x-wallet-address` | Signs the sign-in message → its own [service token](labs-api/service-tokens.md#obtaining-a-token) in `X-Service-Token` | It never authenticates. It signs nothing and is issued no token |
| Where its address goes | `x-wallet-address` | `walletAddress` when issuing a token, `changeBy` on writes, and whatever `:userAddress` resolves to at condition-evaluation time | `labAccountAddress` — including the `account` argument of `isAuthorizedSignerForTba` in [access conditions](getting-started/upload-encrypted-file.md#step-4c-write-the-access-conditions) |
| What it cannot do | — | Transfer the LabNFT, call `updateLabNftMetadata` or `generateLabImageUploadUrl` — those stay Owner-only | Act as a caller: never pass it as `walletAddress` or `changeBy` |

Two failure modes this prevents:

* **Passing the owner's address where the OCL account belongs** in `accessControlConditions`. Condition evaluation **fails closed**, so the file uploads fine and then nobody can decrypt it — with no error saying why. The `account` argument wants `labAccountAddress`; `:userAddress` is substituted with the caller's wallet automatically.
* **Expecting the agent to inherit the human's reach.** The agent authenticates as itself, so its permissions come from its own grant. That is the point — the human never hands over a key — and it is why a few Owner-only mutations stay out of reach. Walkthrough: [Agent access](getting-started/agent-as-a-lab-contributor.md).

#### `oclId` is not a wallet address

`oclId` identifies a lab and looks like an address, but it is a 32-byte value that **packs the OCL account address inside it**, together with the LabNFT `tokenId`:

```
oclId  0x 01     01        000000000000000005f6      f923ca46329c8fcb2fcf8a03512f1483c52c63c5
          ^^     ^^        ^^^^^^^^^^^^^^^^^^^^      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          version namespace tokenId (1526)           the OCL account address, verbatim
```

So the trailing 40 hex characters of an `oclId` are the lab's `labAccountAddress` — which is why a zeroed one is rejected with `VALIDATION_FAILED` / `"embedded address is zero"` rather than a not-found. Pass `oclId` wherever a lab is named; never pass it as a wallet address, and never truncate it to one.

### Using Your Credentials

**For all queries** (read-only operations):

```bash
Authorization: YOUR_CONSUMER_CREDENTIAL
```

**For mutations** (write operations) — as a service:

```bash
Authorization: YOUR_CONSUMER_CREDENTIAL
X-Service-Token: YOUR_SERVICE_TOKEN
```

**For mutations** — as a signed-in user (accepted by all mutations except `extendServiceToken` and `revokeServiceToken`):

```bash
Authorization: Bearer YOUR_PRIVY_TOKEN
x-wallet-address: YOUR_WALLET_ADDRESS
```

**Why more than one header for mutations?**

- **Consumer credential**: Authenticates you as a valid Molecule API consumer
- **Service Token**: Identifies the *wallet* the request acts as; its permissions come from that wallet's onchain role on the target lab
- **Privy token + wallet address**: Identifies a human caller instead, whose write access is derived from the same onchain role model

Which path to choose: use a Service Token for unattended callers (backends, bots, agents, CI/CD) where there is no user session to draw on. Use the Privy path when a signed-in user is driving the request, so the action is attributed to their wallet.

**Security Warnings:**

- Consumer credentials and service tokens are each returned only once, when created — store them securely immediately
- Never commit tokens or consumer credentials to version control
- Never log credentials in application logs
- Store in environment variables or secure secret management systems
- Give an agent's token an `expiresIn` matching its role grant's expiry rather than taking the 180-day default
- Rotate tokens regularly, and `revokeServiceToken` immediately on compromise
- Replace a consumer credential before it expires, and revoke it in the Labs app immediately on compromise

> Service Token lifecycle operations (extending, revoking) are documented in [Service Tokens](labs-api/service-tokens.md).

## Related

* [Service Tokens](labs-api/service-tokens.md) — issue, extend and revoke tokens
* [Getting Started](getting-started/README.md) — the two things you need before your first call
* [Agent access](getting-started/agent-as-a-lab-contributor.md) — an agent with its own wallet and token
* [Roles & Permissions](../technical-deep-dive/roles-and-permissions.md) — what each onchain role can do
* [x402 Gateway](x402-gateway.md) — pay per call instead of holding a token

{% include "../.gitbook/includes/support.md" %}
