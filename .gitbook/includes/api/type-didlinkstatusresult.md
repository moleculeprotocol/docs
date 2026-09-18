Payload of the public getDidLinkStatus query. Failures are thrown as GraphQL errors, never encoded in this payload.

| Name | Type | Description |
| --- | --- | --- |
| `message` | `String` | Human-readable status message. |
| `didLinkStatus` | `DidLinkStatus` | DID-linking snapshot for the lab. Always present for a well-formed `oclId`, with an unknown lab or one having no linking record reporting null status, hashes and DIDs and zero counts. |
