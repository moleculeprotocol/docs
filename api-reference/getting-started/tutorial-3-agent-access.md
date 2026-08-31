---
description: >-
  A human owns the lab and never hands over a key: the agent gets its own
  wallet, the human grants it Contributor, the agent issues its own token.
icon: robot
---

# Tutorial 3: Give your agent access to a lab you created in the app

The most common real-world shape: a researcher created their Lab in the Labs app with an email address — no wallet, no code — and now wants an agent contributing to it. The agent gets its **own** identity rather than borrowing the human's; the human grants it a role; the agent authenticates itself from then on.

{% hint style="info" %}
**Before you start:** you need the [two prerequisites](README.md#prerequisites) — a `mol_` consumer credential and a funded Base Sepolia wallet — plus the [shared setup block](README.md#shared-setup), which defines the config constants and the `graphql()` / `assertOk()` helpers every snippet below uses. The [complete script](#complete-script) at the end of this page carries all of it inline and runs standalone.
{% endhint %}

**Who does what:**

| # | Actor | Action |
| - | ----- | ------ |
| 1 | Agent | Generate a wallet and report its address |
| 2 | Human | Add that address to the lab as **Contributor**, flagged as an agent, with an expiry |
| 3 | Agent | Self-issue a service token by signing the sign-in message |
| 4 | Agent | Upload files and announce; the human sees the result in the app |

The human never hands over a private key, a token, or their session. Revoking the agent is one onchain revoke, and it does not touch anything else.

## Step 1: The agent reports its address

With the plugin: run `wallet_address`. With viem:

```javascript
import { privateKeyToAccount, generatePrivateKey } from "viem/accounts";

// Generate once, store it as the agent's own secret — never the human's key.
const agentPrivateKey = process.env.AGENT_PRIVATE_KEY ?? generatePrivateKey();
const agentAccount = privateKeyToAccount(agentPrivateKey);
console.log("Agent wallet address:", agentAccount.address);
```

Give that address to the lab owner. The agent needs **no gas** for this tutorial — it never sends a transaction, only signs a message. (Persist `agentPrivateKey` if you generated it, or the next run is a different agent with no role.)

## Step 2: The human grants Contributor

In the Labs app, the lab owner adds the agent's address to the lab's members and grants it the **Contributor** role, setting:

* **`isAgent = true`** — informational metadata that marks the member as an agent identity in the members list and UI. It does not change authorisation.
* **an expiry** — typically the agent's session lifetime. When it lapses the agent loses access until it is re-granted; a permanent grant is possible but not the default you want for an agent.

Members can be invited by wallet address, ENS name or email, and the app sponsors the gas for the grant. Under the hood this is one onchain call on the `AccessResolver`, which the owner (or an existing Contributor, for Viewer grants) can also make directly:

```solidity
function grantRole(bytes32 oclId, address account, uint8 role, uint64 expiry, bool isAgent) external;
// role: 2 = ROLE_CONTRIBUTOR, 1 = ROLE_VIEWER
```

Only the **Owner** may grant Contributor. Contributors can grant Viewer, but not Contributor. Full capability matrix: [Roles & Permissions](../../technical-deep-dive/roles-and-permissions.md).

**Why Contributor and not Viewer:** a Viewer can decrypt and read but cannot write. Uploading files and posting announcements needs Contributor.

Both parties can confirm the grant landed with a public query — no authentication beyond the consumer credential:

```javascript
const members = await graphql(
  `query ListLabMembers($oclId: String!) {
    listLabMembers(oclId: $oclId) {
      members { walletAddress role isAgent expiry grantedAt }
    }
  }`,
  { oclId },
);
const grant = members.listLabMembers.members.find(
  (m) => m.walletAddress.toLowerCase() === agentAccount.address.toLowerCase(),
);
console.log("Agent role:", grant?.role, "expiry:", grant?.expiry ?? "permanent");
```

Expect `role: "CONTRIBUTOR"`. `isAgent` simply echoes the flag the owner set — `false` there changes nothing about what the agent may do, so do not treat it as a failed grant. `expiry` is unix seconds as a decimal string, or `null` for a permanent grant. Expired grants are excluded from this list entirely, so a missing entry after a while means the grant lapsed.

## Step 3: The agent self-issues a service token

Identical to Tutorial 1 Step 1, signed by the **agent's** wallet. Fetch the message and redeem it in one go — it embeds a single-use nonce valid for 10 minutes, so an agent that waits for the human's role grant between fetching and signing will hit `UNAUTHENTICATED` / `reason: NONCE_EXPIRED`. Poll for the grant first (Step 2), then sign in:

```javascript
const AGENT_SERVICE_NAME = "research-agent-1";

const signInMessage = await graphql(
  `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
    getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message expiresAt }
  }`,
  { walletAddress: agentAccount.address, serviceName: AGENT_SERVICE_NAME },
);

const messageSignature = await agentAccount.signMessage({
  message: signInMessage.getServiceSignInMessage.message,
});

const tokenResult = await graphql(
  `mutation GenerateServiceToken(
    $serviceName: String!
    $walletAddress: String!
    $messageSignature: String!
    $expiresIn: String
  ) {
    generateServiceToken(
      serviceName: $serviceName
      walletAddress: $walletAddress
      messageSignature: $messageSignature
      expiresIn: $expiresIn
    ) {
      token tokenId expiresAt
      error { code message requestId retryable details }
    }
  }`,
  {
    serviceName: AGENT_SERVICE_NAME,
    walletAddress: agentAccount.address,
    messageSignature,
    expiresIn: "30d", // match the role grant's expiry rather than taking the 180d default
  },
);
assertOk(tokenResult.generateServiceToken, "generateServiceToken");
serviceToken = tokenResult.generateServiceToken.token;
```

Issuance is **not** gated on holding a role — any wallet can mint a token for itself. The role is what makes the token *useful*: authorisation is resolved per request from the token's wallet against the lab you name. So a token issued before the grant lands keeps working once it does; you do not need to re-issue it.

## Step 4: The agent uploads and announces

From here the agent is an ordinary caller. Run [Tutorial 1 Step 4](tutorial-1-public-upload.md#step-4-upload-the-file) with `changeBy: agentAccount.address`, or [Tutorial 2](tutorial-2-encrypted-upload.md) for a confidential file — the `hasRole` branch of the team conditions is exactly what lets the agent decrypt too. Then [Tutorial 4](tutorial-4-announce.md) to surface it on the lab's feed.

Writes by a Contributor service token are gated per mutation, matching the Privy user path: `initiateCreateOrUpdateFile`, `finishCreateOrUpdateFile`, `createAnnouncement`, `deleteDataRoomFile`, `updateFileMetadata` and `moveEntry` all accept Contributor. A few surfaces remain **Owner-only** and an agent Contributor cannot reach them: `updateLabNftMetadata`, `generateLabImageUploadUrl` and the legal-agreement mutations.

{% hint style="warning" %}
**Retry on `UNAUTHORIZED` right after the grant.** Role state reaches the API through an event indexer, so there is a short window after `grantRole` confirms onchain in which a write still returns `UNAUTHORIZED` (`reason: NOT_CONTRIBUTOR`). It is not a permissions problem and re-issuing the token will not help — wait and retry:

```javascript
async function withIndexerLagRetry(fn, { codes = ["NOT_FOUND"], attempts = 5, baseMs = 2000 } = {}) {
  const laggy = new RegExp(codes.join("|"));
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      if (!laggy.test(String(err)) || i === attempts - 1) throw err;
      await new Promise((r) => setTimeout(r, baseMs * 2 ** i)); // 2s, 4s, 8s, 16s
    }
  }
}

// Same helper as Tutorial 1, with the code this step expects.
await withIndexerLagRetry(() => uploadFile(oclId, "./findings.csv"), { codes: ["UNAUTHORIZED"] });
```
{% endhint %}

## Step 5: Verify from both sides

**The agent** verifies as in Tutorial 1 Step 5 — the file is in `dataRoom.files` with `createdBy` set to the agent's address:

```javascript
const verify = await graphql(
  `query Verify($oclId: String!) {
    labWithDataRoomAndFiles(oclId: $oclId) {
      shortname
      dataRoom { files { path accessLevel version createdBy } }
      announcements { headline changeBy }
    }
  }`,
  { oclId },
);
```

**The human** verifies in the app: the file appears in the lab's data room and the announcement on its activity feed, both attributed to the agent's address, which the members list shows flagged as an agent.

## Revoking the agent

One onchain call, and the agent's writes stop:

```solidity
function revokeRole(bytes32 oclId, address account) external;   // Owner only, for a Contributor
```

Or let the grant's `expiry` lapse. Independently, the agent's token can be killed with `revokeServiceToken(tokenId)` — a token can only revoke or extend **its own** record, so one agent cannot interfere with another's.

## Complete script

The agent's half of the flow, standalone: report the wallet address, wait for the human's grant to appear, self-issue a token, upload, announce, verify. Steps 1 and 2 involve a human, so the script polls for the role rather than assuming it.

Run it with the agent's own key and the `oclId` of the human's lab — the agent never sees the owner's key.

```javascript
#!/usr/bin/env node
import { readFileSync } from "node:fs";
import { basename } from "node:path";
import { privateKeyToAccount, generatePrivateKey } from "viem/accounts";

const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const LAB_APP_URL = "https://testnet.labs.molecule.xyz";
const AGENT_SERVICE_NAME = "research-agent-1";

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL;
const OCL_ID = process.env.OCL_ID; // the human's lab
// Persist this, or every run is a different agent with no role.
const AGENT_PRIVATE_KEY = process.env.AGENT_PRIVATE_KEY;

let serviceToken;

async function graphql(query, variables) {
  const headers = { "Content-Type": "application/json", Authorization: CONSUMER_CREDENTIAL };
  if (serviceToken) headers["X-Service-Token"] = serviceToken;
  const res = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });
  const { data, errors } = await res.json();
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}

function assertOk(result, op) {
  if (result.error) {
    const { code, message, requestId } = result.error;
    throw new Error(`${op} failed: ${code}: ${message} (requestId ${requestId})`);
  }
  return result;
}

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

// Writes can return UNAUTHORIZED for a few seconds after a role grant confirms
// onchain — the indexer trails the chain. Retry rather than re-issuing the
// token. Same helper as Tutorial 1, which retries NOT_FOUND after a mint.
async function withIndexerLagRetry(fn, { codes = ["NOT_FOUND"], attempts = 5, baseMs = 2000 } = {}) {
  const laggy = new RegExp(codes.join("|"));
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      if (!laggy.test(String(err)) || i === attempts - 1) throw err;
      await sleep(baseMs * 2 ** i); // 2s, 4s, 8s, 16s
    }
  }
}

async function main() {
  const filePath = process.argv[2];
  if (!filePath) throw new Error("Usage: node tutorial-3.js <file-to-upload>");
  if (!OCL_ID) throw new Error("Set OCL_ID to the lab the human owns");

  // ---- Step 1: the agent's identity ----
  if (!AGENT_PRIVATE_KEY) {
    console.log("No AGENT_PRIVATE_KEY set. Generated one for this run only:");
    console.log("  AGENT_PRIVATE_KEY=" + generatePrivateKey());
    throw new Error("Store that key, grant it Contributor, then re-run.");
  }
  const agentAccount = privateKeyToAccount(AGENT_PRIVATE_KEY);
  console.log("1/6 Agent wallet:", agentAccount.address);
  console.log("    Ask the lab owner to add it as Contributor (isAgent = true).");

  // ---- Step 2: wait for the human's grant (public query, no auth needed) ----
  let grant;
  for (let i = 0; i < 60; i++) {
    const members = await graphql(
      `query ListLabMembers($oclId: String!) {
        listLabMembers(oclId: $oclId) {
          members { walletAddress role isAgent expiry }
        }
      }`,
      { oclId: OCL_ID },
    );
    grant = members.listLabMembers.members.find(
      (m) => m.walletAddress.toLowerCase() === agentAccount.address.toLowerCase(),
    );
    if (grant) break;
    await sleep(5000); // poll for up to 5 minutes
  }
  if (!grant) throw new Error("No role grant found for the agent wallet — ask the owner to add it");
  if (grant.role === "VIEWER") throw new Error("Agent holds VIEWER; uploading needs CONTRIBUTOR");
  console.log("2/6 Role:", grant.role, "isAgent:", grant.isAgent, "expiry:", grant.expiry ?? "permanent");

  // ---- Step 3: the agent self-issues a token ----
  // Only now, after the role poll above returned — the sign-in message holds a
  // single-use nonce that expires 10 minutes after issuance.
  const signInMessage = await graphql(
    `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
      getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message expiresAt }
    }`,
    { walletAddress: agentAccount.address, serviceName: AGENT_SERVICE_NAME },
  );
  const messageSignature = await agentAccount.signMessage({
    message: signInMessage.getServiceSignInMessage.message,
  });
  const tokenResult = await graphql(
    `mutation GenerateServiceToken($serviceName: String!, $walletAddress: String!, $messageSignature: String!, $expiresIn: String) {
      generateServiceToken(serviceName: $serviceName, walletAddress: $walletAddress, messageSignature: $messageSignature, expiresIn: $expiresIn) {
        token expiresAt
        error { code message requestId retryable details }
      }
    }`,
    {
      serviceName: AGENT_SERVICE_NAME,
      walletAddress: agentAccount.address,
      messageSignature,
      expiresIn: "30d", // match the role grant rather than taking the 180d default
    },
  );
  assertOk(tokenResult.generateServiceToken, "generateServiceToken");
  serviceToken = tokenResult.generateServiceToken.token;
  console.log("3/6 Token issued, expires", tokenResult.generateServiceToken.expiresAt);

  // ---- Step 4: upload (public; see Tutorial 2 for the encrypted variant) ----
  // Retried on UNAUTHORIZED: the role grant may not be indexed yet. NOT_FOUND
  // is included for the case where the lab itself was minted moments ago.
  const bytes = readFileSync(filePath);
  const { datasetId } = await withIndexerLagRetry(async () => {
    const initiateResult = await graphql(
      `mutation Initiate($oclId: String!, $contentType: String!, $contentLength: Int!) {
        initiateCreateOrUpdateFile(oclId: $oclId, contentType: $contentType, contentLength: $contentLength) {
          uploadToken uploadUrl method headers { key value }
          error { code message requestId retryable details }
        }
      }`,
      { oclId: OCL_ID, contentType: "application/octet-stream", contentLength: bytes.length },
    );
    assertOk(initiateResult.initiateCreateOrUpdateFile, "initiateCreateOrUpdateFile");
    const { uploadToken, uploadUrl, method, headers } = initiateResult.initiateCreateOrUpdateFile;

    const uploadHeaders = {};
    headers.forEach((h) => (uploadHeaders[h.key] = h.value));
    const putResponse = await fetch(uploadUrl, {
      method: method || "PUT",
      headers: uploadHeaders,
      body: bytes,
    });
    if (!putResponse.ok) throw new Error(`Upload failed: ${putResponse.status}`);

    const finishResult = await graphql(
      `mutation Finish($oclId: String!, $uploadToken: String!, $path: String!, $accessLevel: String!, $changeBy: String!) {
        finishCreateOrUpdateFile(oclId: $oclId, uploadToken: $uploadToken, path: $path, accessLevel: $accessLevel, changeBy: $changeBy) {
          datasetId
          error { code message requestId retryable details }
        }
      }`,
      {
        oclId: OCL_ID,
        uploadToken,
        path: basename(filePath),
        accessLevel: "PUBLIC",
        changeBy: agentAccount.address,
      },
    );
    assertOk(finishResult.finishCreateOrUpdateFile, "finishCreateOrUpdateFile");
    return finishResult.finishCreateOrUpdateFile;
  }, { codes: ["UNAUTHORIZED", "NOT_FOUND"] });
  console.log("4/6 Uploaded — datasetId:", datasetId);

  // ---- Step 4b: announce it ----
  const announcement = await graphql(
    `mutation CreateAnnouncement($oclId: String!, $headline: String!, $body: String!, $attachments: [String!]) {
      createAnnouncement(oclId: $oclId, headline: $headline, body: $body, attachments: $attachments) {
        error { code message requestId retryable details }
      }
    }`,
    {
      oclId: OCL_ID,
      headline: `Agent analysis: ${basename(filePath)}`,
      body: "Written by an autonomous agent holding a Contributor role on this lab.",
      attachments: [datasetId],
    },
  );
  assertOk(announcement.createAnnouncement, "createAnnouncement");
  console.log("5/6 Announced");

  // ---- Step 5: verify ----
  const verify = await graphql(
    `query Verify($oclId: String!) {
      labWithDataRoomAndFiles(oclId: $oclId) {
        shortname
        dataRoom { files { path accessLevel version createdBy } }
      }
    }`,
    { oclId: OCL_ID },
  );
  const file = verify.labWithDataRoomAndFiles.dataRoom.files.find(
    (f) => f.path.endsWith(basename(filePath)),
  );
  if (!file) throw new Error("File not found in the data room");
  const attributed =
    file.createdBy?.toLowerCase() === agentAccount.address.toLowerCase();
  console.log(
    "6/6 Verified:", file.path, file.accessLevel,
    attributed ? "— attributed to the agent" : `— createdBy: ${file.createdBy}`,
  );
  if (verify.labWithDataRoomAndFiles.shortname) {
    console.log("Human can see it at:", `${LAB_APP_URL}/projects/${verify.labWithDataRoomAndFiles.shortname}`);
  }
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**Usage:**

```bash
# First run — prints a generated agent key, then stops so the owner can grant the role
CONSUMER_CREDENTIAL="mol_…" OCL_ID="0x0101…" node tutorial-3.js ./findings.csv

# Subsequent runs, once the owner has granted Contributor to that address
AGENT_PRIVATE_KEY="0x..." CONSUMER_CREDENTIAL="mol_…" OCL_ID="0x0101…" \
node tutorial-3.js ./findings.csv
```

***

## Next

| | |
| --- | --- |
| What the agent uploads | [Tutorial 1 — public file](tutorial-1-public-upload.md) · [Tutorial 2 — encrypted file](tutorial-2-encrypted-upload.md) |
| Have the agent announce its work | [Tutorial 4 — Announce the dataset](tutorial-4-announce.md) |
| The role model in full | [Roles & Permissions](../../technical-deep-dive/roles-and-permissions.md) |
| Let the agent run the whole workflow as tool calls | [Molecule Skill](../../ai-tooling/molecule-skill.md) |
