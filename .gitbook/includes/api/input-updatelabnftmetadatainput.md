Patch input for LabNft display metadata. All fields optional: an omitted field is left unchanged and an explicit `null` clears it; `name` is the exception and cannot be cleared.

| Name | Type | Description |
| --- | --- | --- |
| `name` | `String` | New display name, 2-100 characters, from which the lab's shortname is rederived. Clearing it fails with `VALIDATION_FAILED`, and a shortname clash with another lab fails with `CONFLICT`. |
| `description` | `String` | New description of the lab. |
| `image` | `String` | See `generateLabImageUploadUrl` for hosting an image. |
| `externalUrl` | `String` | New external link for the lab. |
