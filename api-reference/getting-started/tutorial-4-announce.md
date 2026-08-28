---
description: >-
  Publish a lab update that attaches the dataset you uploaded, and read it back
  off the activity feed.
icon: bullhorn
---

# Tutorial 4: Announce the dataset

An announcement is the lab's public update stream: a headline, a body, and optionally the datasets it is about. Attaching the file makes the announcement the discoverable surface for it — announcements are indexed by `searchLabs` alongside files, and they appear on the lab's activity feed and public page.

Requires Owner or Contributor (a Viewer cannot announce). Pick up with `oclId`, `serviceToken` and the `datasetId` returned by `finishCreateOrUpdateFile`.

{% hint style="info" %}
**Before you start:** you need the [two prerequisites](README.md#prerequisites) — a `mol_` consumer credential and a funded Base Sepolia wallet — plus the [shared setup block](README.md#shared-setup), which defines the config constants and the `graphql()` / `assertOk()` helpers every snippet below uses. The [complete script](#complete-script) at the end of this page carries all of it inline and runs standalone.
{% endhint %}

```javascript
const announcementResult = await graphql(
  `mutation CreateAnnouncement(
    $oclId: String!
    $headline: String!
    $body: String!
    $attachments: [String!]
  ) {
    createAnnouncement(oclId: $oclId, headline: $headline, body: $body, attachments: $attachments) {
      message
      error { code message requestId retryable details }
    }
  }`,
  {
    oclId,
    headline: "Baseline assay results published",
    body: "First replicate set for the ApoB series. 240 samples, three conditions. Raw CSV attached.",
    attachments: [datasetId], // the datasetId from finishCreateOrUpdateFile
  },
);
assertOk(announcementResult.createAnnouncement, "createAnnouncement");
```

**Expected response:**

```json
{
  "data": {
    "createAnnouncement": {
      "message": "…",
      "error": null
    }
  }
}
```

`createAnnouncement` returns no announcement object, and its `message` is passed through from the storage layer rather than being a fixed string — so success is `error == null` and nothing else. Verify by reading the announcement back.

**If it fails:**

| `error.code` | What happened | Fix |
| ------------ | ------------- | --- |
| `UNAUTHORIZED` | Caller is a Viewer, or has no role | Contributor or Owner required. Fresh grant? See the [retry note](tutorial-3-agent-access.md#step-4-the-agent-uploads-and-announces) |
| `NOT_FOUND` | An `attachments` entry isn't a dataset in this lab | Pass the exact `datasetId` strings from `finishCreateOrUpdateFile`, from this lab |
| `VALIDATION_FAILED` | Empty `headline` or `body` | Both are required and non-empty |

## Verify it worked

```javascript
const feed = await graphql(
  `query LabActivity($oclId: String!) {
    labActivity(oclId: $oclId, page: 0, perPage: 10, filter: ANNOUNCEMENT) {
      nodes {
        __typename
        ... on LabEventAnnouncement {
          announcement {
            id
            headline
            body
            changeBy
            eventTime
            attachments { path contentType accessLevel }
          }
        }
      }
      pageInfo { totalPages currentPage }
    }
  }`,
  { oclId },
);
console.log(JSON.stringify(feed.labActivity.nodes[0], null, 2));
```

Your announcement is the newest node, with `attachments` resolved to the full file objects — not just ids — and `changeBy` set to the wallet that posted it. It is also on the lab's public page at `${LAB_APP_URL}/projects/<shortname>`.

`labActivity` is a **public** query: anyone with a consumer credential can read the feed, which is the point of an announcement.

## Complete script

Announce and verify, standalone. Takes the `oclId` and the `datasetId` you got from `finishCreateOrUpdateFile` in Tutorial 1 or 2.

```javascript
#!/usr/bin/env node
import { privateKeyToAccount } from "viem/accounts";

const GRAPHQL_URL = "https://staging.graphql.api.molecule.xyz/graphql";
const LAB_APP_URL = "https://testnet.labs.molecule.xyz";
const SERVICE_NAME = "tutorial-agent";

const CONSUMER_CREDENTIAL = process.env.CONSUMER_CREDENTIAL;
const WALLET_PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY;
const OCL_ID = process.env.OCL_ID;
// Optional: omit to announce without attaching anything.
const DATASET_ID = process.env.DATASET_ID;

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

async function main() {
  const headline = process.argv[2];
  const body = process.argv[3];
  if (!headline || !body) throw new Error('Usage: node tutorial-4.js "<headline>" "<body>"');
  if (!OCL_ID) throw new Error("Set OCL_ID");

  const account = privateKeyToAccount(WALLET_PRIVATE_KEY);

  // ---- Service token (Tutorial 1 Step 1) ----
  const signInMessage = await graphql(
    `query GetServiceSignInMessage($walletAddress: String!, $serviceName: String!) {
      getServiceSignInMessage(walletAddress: $walletAddress, serviceName: $serviceName) { message }
    }`,
    { walletAddress: account.address, serviceName: SERVICE_NAME },
  );
  const messageSignature = await account.signMessage({
    message: signInMessage.getServiceSignInMessage.message,
  });
  const tokenResult = await graphql(
    `mutation GenerateServiceToken($serviceName: String!, $walletAddress: String!, $messageSignature: String!) {
      generateServiceToken(serviceName: $serviceName, walletAddress: $walletAddress, messageSignature: $messageSignature) {
        token
        error { code message requestId retryable details }
      }
    }`,
    { serviceName: SERVICE_NAME, walletAddress: account.address, messageSignature },
  );
  assertOk(tokenResult.generateServiceToken, "generateServiceToken");
  serviceToken = tokenResult.generateServiceToken.token;
  console.log("1/3 Got service token");

  // ---- Announce ----
  const announcementResult = await graphql(
    `mutation CreateAnnouncement($oclId: String!, $headline: String!, $body: String!, $attachments: [String!]) {
      createAnnouncement(oclId: $oclId, headline: $headline, body: $body, attachments: $attachments) {
        message
        error { code message requestId retryable details }
      }
    }`,
    {
      oclId: OCL_ID,
      headline,
      body,
      attachments: DATASET_ID ? [DATASET_ID] : undefined,
    },
  );
  assertOk(announcementResult.createAnnouncement, "createAnnouncement");
  console.log("2/3 Announced");

  // ---- Verify off the activity feed ----
  const feed = await graphql(
    `query LabActivity($oclId: String!) {
      labActivity(oclId: $oclId, page: 0, perPage: 10, filter: ANNOUNCEMENT) {
        nodes {
          __typename
          ... on LabEventAnnouncement {
            announcement {
              id headline changeBy eventTime
              attachments { path contentType accessLevel }
            }
          }
        }
      }
      labWithDataRoomAndFiles(oclId: $oclId) { shortname }
    }`,
    { oclId: OCL_ID },
  );
  const newest = feed.labActivity.nodes.find(
    (n) => n.announcement?.headline === headline,
  );
  if (!newest) throw new Error("Announcement not found on the activity feed");
  console.log(
    "3/3 Verified:", newest.announcement.headline,
    "—", newest.announcement.attachments.length, "attachment(s)",
    "by", newest.announcement.changeBy,
  );
  if (feed.labWithDataRoomAndFiles?.shortname) {
    console.log("Lab page:", `${LAB_APP_URL}/projects/${feed.labWithDataRoomAndFiles.shortname}`);
  }
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**Usage:**

```bash
WALLET_PRIVATE_KEY="0x..." CONSUMER_CREDENTIAL="mol_…" \
OCL_ID="0x0101…" DATASET_ID="did:odf:fed01…" \
node tutorial-4.js "Baseline assay results published" "First replicate set for the ApoB series."
```

***

## Next

| | |
| --- | --- |
| Upload another file first | [Tutorial 1 — public](tutorial-1-public-upload.md) · [Tutorial 2 — encrypted](tutorial-2-encrypted-upload.md) |
| Run it against mainnet | [Running in Production](README.md#running-in-production) |
| Search and feed queries | [Browse & Search](../labs-api/browse-and-search.md) |
