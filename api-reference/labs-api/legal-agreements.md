# Legal Agreements

The legal-agreement flow is three operations: fetch the populated template + `contentHash`, sign it as an EIP-712 typed-data payload, then submit the signature. The backend regenerates and verifies the document server-side and stores the signed artifact in the lab's data room.

> `type` is a `LegalAgreementType` enum. Current value: `ASSIGNMENT_AGREEMENT`.

## EIP-712 Envelope

Every agreement type shares **one** signature schema — `LegalAgreementAcceptance` — forever; adding new agreement types never changes it, since the agreement type is itself a signed field. Sign this typed-data payload with the wallet returned by `legalAgreementTemplate`, then submit the signature to `signLegalAgreement` below.

### Domain

| Field               | Value                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `name`               | `"MoleculeOcl"`                                                                                                                            |
| `version`            | `"1"`                                                                                                                                      |
| `chainId`            | the **deployment's L2 chain** — `84532` (Base Sepolia) on dev/staging, `8453` (Base) in production. This is the chain the LabNFT lives on, not the OCL "canonical chain id" used for CREATE2 address derivation. |
| `verifyingContract`  | the environment's LabNFT proxy, **lowercased** — `0x13ff210695fdb54a7f928eccc28bc3486c05bb28` (dev/staging), `0x9f96027eeafb9ad5f2b5d7043b36ee96b2eebe92` (production) |

### Types

```
LegalAgreementAcceptance(
  bytes32 oclId,           // lowercased 0x… 32-byte lab id
  string  agreementType,   // registry SLUG — e.g. "assignment-agreement" — NOT the GraphQL enum value
  bytes32 contentHash,     // contentHash from legalAgreementTemplate, echoed verbatim
  string  templateVersion, // templateVersion from legalAgreementTemplate, echoed verbatim
  address signer,          // lowercased signer wallet — must equal walletAddress
  uint64  issuedAt         // unix seconds, echoed verbatim from legalAgreementTemplate
)
```

> **`agreementType` is the registry slug, not the `LegalAgreementType` enum value.** The GraphQL enum (`type: LegalAgreementType`) uses `ASSIGNMENT_AGREEMENT`; the signed `agreementType` field uses the human-readable slug the wallet renders to the user. Signing with the enum value instead of the slug produces a digest the backend can't match — `signLegalAgreement` returns `error.code` `UNAUTHENTICATED` with `details.reason` `INVALID_SIGNATURE`.

| `LegalAgreementType` (GraphQL enum) | `agreementType` (signed slug) |
| ------------------------------------ | ------------------------------- |
| `ASSIGNMENT_AGREEMENT`               | `assignment-agreement`          |

Verification is off-chain (viem `verifyTypedData` — EOA and EIP-1271, so Safe/smart-contract wallets work) against the LabNFT's current owner. `verifyingContract` is the LabNFT proxy purely as wallet-rendered scope and for forward compatibility with onchain verification — verification itself does not call the contract.

### Building the Typed Data (viem)

```javascript
const typedData = {
  domain: {
    name: "MoleculeOcl",
    version: "1",
    chainId: 84532, // 8453 in production
    verifyingContract: "0x13ff210695fdb54a7f928eccc28bc3486c05bb28", // per-environment LabNFT proxy, lowercased
  },
  types: {
    LegalAgreementAcceptance: [
      { name: "oclId", type: "bytes32" },
      { name: "agreementType", type: "string" },
      { name: "contentHash", type: "bytes32" },
      { name: "templateVersion", type: "string" },
      { name: "signer", type: "address" },
      { name: "issuedAt", type: "uint64" },
    ],
  },
  primaryType: "LegalAgreementAcceptance",
  message: {
    oclId: oclId.toLowerCase(),
    agreementType: "assignment-agreement", // registry slug for ASSIGNMENT_AGREEMENT
    contentHash, // from legalAgreementTemplate, verbatim
    templateVersion, // from legalAgreementTemplate, verbatim
    signer: walletAddress.toLowerCase(),
    issuedAt: BigInt(issuedAt), // from legalAgreementTemplate, verbatim
  },
};

const signature = await walletClient.signTypedData(typedData);
```

Pass `signature` — plus the same `issuedAt`, and any `signerName` / `entity` / `title` you got from the template call — into [Sign Legal Agreement](#sign-legal-agreement-mutation) below.

### Test Vector

Use this to verify your EIP-712 implementation produces byte-identical output before wiring it up against a live wallet. Signed with the well-known Anvil/Hardhat default test account #0 — **never use this key outside local testing.**

| Field         | Value                                                                       |
| ------------- | ---------------------------------------------------------------------------- |
| private key   | `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`         |
| signer        | `0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266`                                 |

Domain:

```json
{
  "name": "MoleculeOcl",
  "version": "1",
  "chainId": 84532,
  "verifyingContract": "0x13ff210695fdb54a7f928eccc28bc3486c05bb28"
}
```

Message:

```json
{
  "oclId": "0x0101000000000000000000007777777777777777777777777777777777777777",
  "agreementType": "assignment-agreement",
  "contentHash": "0xc0ffee0000000000000000000000000000000000000000000000000000000042",
  "templateVersion": "1.0.0",
  "signer": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
  "issuedAt": 1781136000
}
```

Expected output:

| Name                              | Value                                                                                                                               |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| EIP-712 digest (`hashTypedData`)   | `0x054f310942709f9aabf643fec035b1b9a726356b4e78f0f5c1f93ae9ad659ba8`                                                                |
| Signature (`signTypedData`)        | `0x327ecaa2da9382370bda32176794a231154e25be5d0e37f0fd4011ac067d94d752d816db038764f8a4a251457a42d2df3bbbc710b3909af421e5c7984c4464e61b` |

If your implementation doesn't reproduce these, check first for mixed-case addresses (EIP-712 hashes raw bytes so case doesn't affect the digest, but some libraries reject invalid-checksum mixed case before they get that far — use lowercase throughout) and for `issuedAt` sent as a `number` instead of the `uint64`/`bigint` the type expects.

## Get Legal Agreement Template (query)

Return the populated agreement the given wallet is expected to sign, plus the `contentHash` for the EIP-712 envelope. Read-only.

```graphql
query LegalAgreementTemplate(
  $oclId: String!
  $type: LegalAgreementType!
  $walletAddress: String!
  $signerName: String
  $entity: String
  $title: String
) {
  legalAgreementTemplate(
    oclId: $oclId
    type: $type
    walletAddress: $walletAddress
    signerName: $signerName
    entity: $entity
    title: $title
  ) {
    agreement
    contentHash
    templateVersion
    agreementType
    issuedAt
  }
}
```

This is a query: failures arrive as top-level GraphQL `errors[]` entries with `errorType` set to a catalogue code (e.g. `UNAUTHENTICATED`, `UNAUTHORIZED`, `NOT_FOUND`, `VALIDATION_FAILED`), and `data` comes back `null` (the field is non-nullable, so the error propagates to the root). The result type has no `error` field.

**Parameters:**

| Parameter     | Type               | Required | Description                                                                                           |
| ------------- | ------------------ | -------- | ----------------------------------------------------------------------------------------------------- |
| oclId         | String             | Yes      | Canonical 32-byte oclId of the lab                                                                    |
| type          | LegalAgreementType | Yes      | Agreement type (`ASSIGNMENT_AGREEMENT`)                                                               |
| walletAddress | String             | Yes      | Intended signer. On the user-auth path this must equal the authenticated wallet                       |
| signerName    | String             | No       | Signer identity (natural person). Echoed VERBATIM into `signLegalAgreement`; covered by `contentHash` |
| entity        | String             | No       | Signing entity, if the signer represents an organization                                              |
| title         | String             | No       | Signer title                                                                                          |

> `agreement` is for display only — the client must NOT re-serialize or re-hash it; sign over `contentHash` as given, and echo `issuedAt`, `signerName`, `entity`, and `title` back unchanged at sign time or the regenerated hash won't match.

## Check Legal Agreement Status (query)

Whether (and at which template versions) the agreement has been signed for the lab. **Public** — same surface as `labs`. Also available inline as `Lab` / `LabRef.legalAgreementStatus`.

```graphql
query LegalAgreementStatus($oclId: String!, $type: LegalAgreementType!) {
  legalAgreementStatus(oclId: $oclId, type: $type) {
    signed
    isCurrentVersionSigned
    currentTemplateVersion
    signedVersions {
      templateVersion
      path
      signer
      signedAt
      contentHash
      issuedAt
      signature
    }
  }
}
```

> **Two surfaces, two failure modes.** As the top-level `legalAgreementStatus(oclId, type)` query above, failures throw like every other query — a top-level GraphQL `errors[]` entry with `errorType` set to a catalogue code — and the payload's `error` field is always `null`. As the `Lab.legalAgreementStatus` / `LabRef.legalAgreementStatus` field, an upstream failure does not null the whole lab: the field returns the payload with `error` set (typically `UPSTREAM_UNAVAILABLE`). There, `error != null` means the status could not be determined and `signed` / `isCurrentVersionSigned` must not be trusted; only when `error == null` does `signed: false` mean "not signed". Select `error { code message requestId retryable details }` on that field and check it before routing a signing flow — for example, inline on `labs`:

```graphql
query LabsWithAgreementStatus {
  labs(perPage: 5) {
    nodes {
      oclId
      legalAgreementStatus(type: ASSIGNMENT_AGREEMENT) {
        signed
        isCurrentVersionSigned
        error {
          code
          message
          requestId
          retryable
          details
        }
      }
    }
  }
}
```

`signed` is true if any version has been signed; `isCurrentVersionSigned` reflects the current template version (the FE routes the signing flow on this). The `signedVersions` enrichment is fetched lazily, only when its sub-fields are selected.

## Sign Legal Agreement (mutation)

Record acceptance of a legal agreement. The backend regenerates the document from `(type, oclId, walletAddress, issuedAt)`, verifies the EIP-712 signature and LabNFT ownership, embeds the signature into a self-verifying envelope, and uploads it to the lab's data room. Rejects if the current template version is already signed.

```graphql
mutation SignLegalAgreement($input: SignLegalAgreementInput!) {
  signLegalAgreement(input: $input) {
    oclId
    path
    contentHash
    templateVersion
    datasetId
    version
    message
    error {
      code
      message
      requestId
      retryable
      details
    }
  }
}
```

Success ⇔ `error == null`. On failure `message` mirrors `error.message`; branch on `error.code` (and `details.reason` where documented — e.g. `CONFLICT` with reason `ALREADY_SIGNED` when the current template version is already signed, or `FAILED_PRECONDITION` with reason `TEMPLATE_EXPIRED`). `details` is a JSON-encoded string: `JSON.parse(error.details ?? "{}").reason`.

**`SignLegalAgreementInput` fields:**

| Field         | Type               | Required | Description                                                                                                              |
| ------------- | ------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------ |
| oclId         | String             | Yes      | Canonical 32-byte oclId of the lab                                                                                       |
| type          | LegalAgreementType | Yes      | Agreement type (`ASSIGNMENT_AGREEMENT`)                                                                                  |
| walletAddress | String             | Yes      | Signer's wallet. Must equal the EIP-712 signer, the lab's current LabNFT owner, and (user path) the authenticated wallet |
| signature     | String             | Yes      | EIP-712 signature (0x…) over the `LegalAgreementAcceptance` typed data                                                   |
| issuedAt      | AWSTimestamp       | Yes      | Echoed VERBATIM from `legalAgreementTemplate` — used to regenerate the document                                          |
| signerName    | String             | No       | Echoed VERBATIM from the template call (covered by `contentHash`)                                                        |
| entity        | String             | No       | Echoed verbatim; see `signerName`                                                                                        |
| title         | String             | No       | Echoed verbatim; see `signerName`                                                                                        |

---

