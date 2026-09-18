Filters for `searchLabs`. Each list restricts hits to entries matching any of its values; lists are combined with AND. Values are forwarded to the search backend as given.

| Name | Type | Description |
| --- | --- | --- |
| `byOclIds` | `[String!]` | Restrict hits to these labs (32-byte 0x-hex OCL ids). |
| `byTags` | `[String!]` | Restrict hits to files and announcements carrying any of these tags. |
| `byCategories` | `[String!]` | Restrict hits to files and announcements in any of these categories. |
| `byAccessLevels` | `[String!]` | Restrict hits to entries with any of these access levels (PUBLIC, HOLDERS, ADMIN). |
| `byKinds` | `[String!]` | Restrict hits to these kinds of entry: "FILE" and/or "ANNOUNCEMENT". Any other value is rejected by the search backend. |
