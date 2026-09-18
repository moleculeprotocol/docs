Patch input for LabNft display metadata. All fields optional: an omitted field is left unchanged and an explicit `null` clears it; `name` is the exception and cannot be cleared.

| Name | Type | Description |
| --- | --- | --- |
| `name` | `String` | New display name, 2-100 characters, from which the lab's shortname is rederived. Clearing it fails with `VALIDATION_FAILED`, and a shortname clash with another lab fails with `CONFLICT`. |
| `description` | `String` | New description of the lab. |
| `image` | `String` | See `generateLabImageUploadUrl` for hosting an image. |
| `externalUrl` | `String` | New external link for the lab. |
| `websiteUrl` | `AWSURL` | Website URL, using absolute HTTPS without credentials, at most 2048 characters. `null` clears it; invalid URLs fail with `VALIDATION_FAILED`. |
| `xUrl` | `AWSURL` | X (Twitter) profile URL; validation and clearing rules match `UpdateLabNftMetadataInput.websiteUrl`. |
| `telegramUrl` | `AWSURL` | Telegram group or channel URL; validation and clearing rules match `UpdateLabNftMetadataInput.websiteUrl`. |
| `governanceUrl` | `AWSURL` | Governance forum or voting URL; validation and clearing rules match `UpdateLabNftMetadataInput.websiteUrl`. |
| `projectFormat` | `String` | Active slug from `LabTaxonomyResult.projectFormats`; `null` clears it. Unknown or retired slugs fail with `VALIDATION_FAILED`, with the rejected slug in error details. |
| `therapeuticFields` | `[String!]` | Active slugs from `LabTaxonomyResult.therapeuticFields`, deduplicated in order; `null` or `[]` clears them. More than 10 unique slugs or unknown or retired slugs fail with `VALIDATION_FAILED`. |
