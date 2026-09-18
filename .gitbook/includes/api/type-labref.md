Lightweight lab reference, returned wherever a lab is referenced rather than fully expanded. How much is hydrated depends on the query, so each field documents where it is null. Beyond the identifier fields a null can mean the source was briefly unavailable, or a value served from a mirror that lags by a few hours.

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | Canonical 32-byte oclId (lowercase 0x-hex); primary identifier. |
| `mintedAt` | `AWSDateTime` | LabNft mint block timestamp from the OclIdentityCreated event. |
| `shortname` | `String` | Human-readable slug (see `Lab.shortname`), seeded as "lab-<tokenId>" at index time and re-derived on rename. Null on `searchLabs` and for unbackfilled legacy labs; replaces the legacy token `symbol`. |
| `ipnftId` | `String` | Linked legacy IPNFT tokenId. Null on `searchLabs` and when no IPNFT is linked; `ipnft` carries the full object on the queries that hydrate it. |
| `hasDataRoom` | `Boolean` | Whether the lab's data room exists; it must be created before files or announcements can be added. Null on `searchLabs`, which does not mean false. |
| `ipnft` | `IPNFT` | Legacy IPNFT this lab was migrated from, hydrated when selected. Null when no IPNFT is linked, and on the activity feeds and `searchLabs`, which expose `ipnftId` for a follow-up lookup. |
| `labAccountAddress` | `String!` | ERC-6551 token-bound smart-account address for this lab; deterministic from the LabNft tokenId via the OnChainLabFactory. Replaces the misleadingly-named `ipnftAddress`. |
| `labNftTokenId` | `String!` | LabNft tokenId as a decimal string. Replaces the legacy `ipnftTokenId`. |
| `latestContributionAt` | `AWSDateTime` | Timestamp of the lab's latest data room activity, lagging a few hours on `labsConnection` and `lab`. Null on `labs` unless selected, on other queries, when there is none, or on lookup failure. |
| `name` | `String` | LabNft display name (defaults to "Lab #<tokenId>"). Null on `searchLabs`. |
| `description` | `String` | Owner-editable description (see `updateLabNftMetadata`). Null when unset and on `searchLabs`. |
| `image` | `String` | LabNft image URL, null until the owner sets a custom image. Null on `searchLabs`. |
| `externalUrl` | `String` | External link surfaced by the LabNft tokenURI metadata. Null on `searchLabs`. |
| `members` | `[LabMember!]` | Active members of the lab, mirroring the public `listLabMembers` query. Null only on lookup failure. |
| `legalAgreementStatus` | `LegalAgreementStatusResult` | Signed status of a legal agreement for this lab, inlined so a client can fetch it per lab instead of calling `legalAgreementStatus` N times. Public, and null only when the lab cannot be resolved. |
| `trlValue` | `String` | Technology Readiness Level from Molecule's AI-assisted assessment rather than onchain. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `trlRationale` | `String` | Human-readable explanation for the assigned `trlValue`. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `trlLastUpdated` | `AWSDateTime` | When `trlValue` last changed. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `isVerified` | `Boolean` | Whether Molecule has verified this lab. Null when no verification decision has been recorded, and on the activity feeds and `searchLabs`. |
| `weightedScore` | `Float` | Weighted score from Molecule's assessment, summing all weighted criterion scores, where 5.0 is best and 1.0 worst. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `scoreInterpretation` | `String` | Human-readable summary of the overall assessment behind `weightedScore`. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `criterionScores` | `[AWSJSON!]` | Per-criterion breakdown behind `weightedScore`, each entry shaped like `{ "criterion": String, "score": Number }` with further keys possible over time. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `scoredAt` | `AWSDateTime` | When the scoring behind `weightedScore` was last computed. Null when no assessment exists, and on the activity feeds and `searchLabs`. |
| `todos` | `[AWSJSON!]` | Action items for the lab from Molecule's assessment (AI-assisted), each a JSON object shaped like `{ "todo": String, "completed": Boolean }`; further keys may be added over time. Null when none exist, and on the activity feeds and `searchLabs`. |
| `websiteUrl` | `AWSURL` | Lab website URL; null when unset or not hydrated, including on `searchLabs`. |
| `xUrl` | `AWSURL` | Lab X (Twitter) profile URL; null when unset or not hydrated, including on `searchLabs`. |
| `telegramUrl` | `AWSURL` | Lab Telegram group or channel URL; null when unset or not hydrated, including on `searchLabs`. |
| `governanceUrl` | `AWSURL` | Lab governance forum or voting URL; null when unset or not hydrated, including on `searchLabs`. |
| `projectFormat` | `String` | Project format slug, including stored retired slugs; active choices are in `LabTaxonomyResult.projectFormats`. Null when unset or not hydrated, including on `searchLabs`. |
| `therapeuticFields` | `[String!]` | Therapeutic-field slugs in their saved order; active choices are in `LabTaxonomyResult.therapeuticFields`. Empty when unset, null when not hydrated, including on `searchLabs`. |
| `isRwaEnabled` | `Boolean` | Whether the lab is RWA-enabled, as defined by `Lab.isRwaEnabled`. Null when metadata is not hydrated, including on `searchLabs`. |

**`legalAgreementStatus` arguments**

| Name | Type | Description |
| --- | --- | --- |
| `type` | `LegalAgreementType!` | Which legal agreement to report on. |
