Returns the deterministic message that a service must sign to authenticate via wallet signature. Public query, no auth required. Use cases: autonomous agents, bots, CI/CD pipelines, or any service that needs a token without a browser-based Privy session.

Returns `ServiceSignInMessageResult!`.

| Name | Type | Description |
| --- | --- | --- |
| `walletAddress` | `String!` | Wallet address of the service (e.g. an agent's EOA). |
| `serviceName` | `String!` | Name of the service requesting a token. |
