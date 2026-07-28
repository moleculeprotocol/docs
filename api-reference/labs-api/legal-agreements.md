# Legal Agreements

The legal-agreement flow is three operations: fetch the populated template + `contentHash`, sign it as an EIP-712 typed-data payload, then submit the signature. The backend regenerates and verifies the document server-side and stores the signed artifact in the lab's data room.

> `type` is a `LegalAgreementType` enum. Current value: `ASSIGNMENT_AGREEMENT`.

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
    isSuccess
    agreement
    contentHash
    templateVersion
    agreementType
    issuedAt
    error {
      message
      code
      retryable
    }
  }
}
```

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
    isSuccess
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
    error {
      message
      code
      retryable
    }
  }
}
```

Check `isSuccess` before reading the other fields. `signed` is true if any version has been signed; `isCurrentVersionSigned` reflects the current template version (the FE routes the signing flow on this). The `signedVersions` enrichment is fetched lazily, only when its sub-fields are selected.

## Sign Legal Agreement (mutation)

Record acceptance of a legal agreement. The backend regenerates the document from `(type, oclId, walletAddress, issuedAt)`, verifies the EIP-712 signature and LabNFT ownership, embeds the signature into a self-verifying envelope, and uploads it to the lab's data room. Rejects if the current template version is already signed.

```graphql
mutation SignLegalAgreement($input: SignLegalAgreementInput!) {
  signLegalAgreement(input: $input) {
    isSuccess
    oclId
    path
    contentHash
    templateVersion
    datasetId
    version
    message
    error {
      message
      code
      retryable
    }
  }
}
```

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

