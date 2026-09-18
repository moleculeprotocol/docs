States of the background DID-linking state machine.

| Value | Description |
| --- | --- |
| `PENDING` | Linking is queued or being prepared. |
| `SUBMITTED` | User operation has been submitted onchain and awaits confirmation. |
| `LINKED` | Both DIDs are linked onchain; terminal. |
| `FAILED` | Most recent attempt failed; the worker may retry. |
