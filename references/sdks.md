---
icon: book-spine
---

# SDKs

## Molecule Protocol Client SDK Documentation

### Overview

The `@moleculexyz/client-sdk` provides type-safe access to Molecule Protocol's GraphQL APIs and subgraphs, specifically for IP-NFTs, IPTokens, crowdsales, and data rooms.

#### Features:

* Query IPTs, markets, and IP-NFTs
* Manage data room files (upload, encrypt, retrieve)
* Interact with crowdsale data
* Handle authentication with Privy tokens

#### Limitations:

* Does not execute on-chain transactions. For this, use `@moleculexyz/onchain` + `wagmi`.
* Does not custody keys or manage wallets.
* Does not replace contract interactions for writes.

#### Supported Environments:

* Node.js 18+
* Browser (ESM)
* React/Next.js

#### Relationship to Contracts:

Contracts are the definitive source. This SDK queries indexed data from APIs and subgraphs for convenience. For on-chain writes, use the contract ABIs directly with `viem/wagmi`.

### Installation

```bash
npm install @moleculexyz/client-sdk
```

#### Peer Dependencies:

```bash
npm install viem wagmi
```

### Quickstart

This example demonstrates how to upload a file to a data room, a real write operation you can complete in under 10 minutes:

```javascript
import { createDesciSdk } from '@moleculexyz/client-sdk';

// 1. Create authenticated SDK instance
const sdk = createDesciSdk({
  identityToken: 'your-privy-jwt-token',
  walletAddress: '0xYourWallet...',
});

// 2. Initiate file upload
const ipnftUid = '0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1_42';
const file = new File(['Hello World'], 'readme.txt', { type: 'text/plain' });

const { uploadUrl, uploadToken } = await sdk.labs.initiateFileUploadV2(
  ipnftUid,
  file.type,
  file.size
);

// 3. Upload to signed URL
await fetch(uploadUrl, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type },
});

// 4. Finalize upload
const result = await sdk.labs.finishFileUploadV2(
  ipnftUid,
  uploadToken,
  '/documents/readme.txt',
  'PRIVATE',
  '0xYourWallet...',
  { description: 'Project readme' }
);

console.log('File uploaded:', result.path);
```

#### Prerequisites:

* Privy authentication token (from `@privy-io/react-auth`)
* Wallet address
* An IP-NFT you have access to

### Core Abstractions

Understanding these concepts helps you use the SDK effectively:

| SDK Concept | On-chain Equivalent                           |
| ----------- | --------------------------------------------- |
| `IPT`       | IPToken ERC-20 contract                       |
| `IP-NFT`    | IP-NFT ERC-721 token                          |
| `Market`    | Uniswap/DEX pool for an IPT                   |
| `CrowdSale` | CrowdSale contract instance                   |
| `DataRoom`  | Off-chain encrypted storage tied to an IP-NFT |
| `ProjectV2` | Lab/project metadata in the API               |

### Client

The DesciSdk client wraps all domain APIs:

```javascript
const sdk = createDesciSdk(config);
```

#### Modules:

* **tokens**: IPT queries with market data
* **ipnfts**: IP-NFT queries
* **labs**: Data room & project management
* **crowdsales**: Crowdsale data from subgraph
* **minting**: IP-NFT minting flow
* **tokenizer**: IPT creation flow

### Authentication Modes

#### Unauthenticated (public data):

* `sdk.tokens.getAllWithMarkets()`
* `sdk.ipnfts.getById(id)`
* `sdk.crowdsales.getById(id)`

#### Authenticated (requires `identityToken` + `walletAddress`):

* `sdk.labs.getPassphrase(ipnftUid)`
* `sdk.labs.initiateFileUploadV2(...)`
* `sdk.minting.reserve()`

### Common Workflows

#### Display IPT Market Data

```javascript
import { sdk } from '@moleculexyz/client-sdk';

// Fetch all IPTs with prices
const tokens = await sdk.tokens.getAllWithMarkets({
  sortBy: 'MARKET_CAP',
  sortOrder: 'desc',
  limit: 20,
});

// Display
tokens.forEach(token => {
  const market = token.markets[0];
  console.log(`${token.metadata.symbol}: $${market?.priceUsd || 'N/A'}`);
});
```

#### Upload Encrypted File to DataRoom

```javascript
import { createDesciSdk } from '@moleculexyz/client-sdk';
import { useLitEncryption } from '@moleculexyz/storage';

// 1. Encrypt file with Lit Protocol
const { encryptFile } = useLitEncryption();
const encrypted = await encryptFile(file, {
  type: 'authorized_ipnft_signer',
  ipnftId: '42',
});

// 2. Upload encrypted content
const sdk = createDesciSdk({ identityToken, walletAddress });

const { uploadUrl, uploadToken } = await sdk.labs.initiateFileUploadV2(
  ipnftUid,
  'application/octet-stream',
  encrypted.ciphertext.length
);

await fetch(uploadUrl, { method: 'PUT', body: encrypted.ciphertext });

// 3. Finalize with encryption metadata
await sdk.labs.finishFileUploadV2(
  ipnftUid,
  uploadToken,
  '/private/research.pdf',
  'PRIVATE',
  walletAddress,
  {
    encryptionMetadata: {
      dataToEncryptHash: encrypted.dataToEncryptHash,
      accessControlConditions: JSON.stringify(encrypted.accessCondition),
      chain: 'ethereum',
      litNetwork: 'datil-dev',
      // ... other Lit metadata
    },
  }
);
```

#### Check Crowdsale Contribution

```javascript
const contribution = await sdk.crowdsales.getContribution(
  saleId,
  userAddress
);

if (contribution && !contribution.claimed) {
  console.log(`You can claim ${contribution.amount} tokens`);
}
```

#### Batch Read: IPTs Owned by Address

```javascript
const balances = await sdk.ipnfts.getAllWithBalances(userAddress);

balances?.forEach(ipt => {
  console.log(`${ipt.symbol}: ${ipt.balance}`);
});
```

### Transactions & Signing

#### Read vs Write

<table><thead><tr><th width="334.8359375">Operation</th><th width="109.578125">Type</th><th>Auth Required</th><th>On-chain</th></tr></thead><tbody><tr><td><code>sdk.tokens.getAllWithMarkets()</code></td><td>Read</td><td>No</td><td>No (API)</td></tr><tr><td><code>sdk.crowdsales.getById()</code></td><td>Read</td><td>No</td><td>No (Subgraph)</td></tr><tr><td><code>sdk.labs.initiateFileUploadV2()</code></td><td>Write</td><td>Yes</td><td>No (API)</td></tr><tr><td><code>sdk.labs.finishFileUploadV2()</code></td><td>Write</td><td>Yes</td><td>No (API)</td></tr></tbody></table>

#### Who Signs What

The SDK:

* Handles API authentication (Privy JWT tokens).
* Utilizes presigned upload URLs (S3).

For on-chain transactions (minting, tokenizing, bidding), use `viem/wagmi` with the contract ABIs from `@moleculexyz/common/abis`.

### Confirmation Model

SDK API calls are confirmed when the Promise resolves - these are off-chain operations. For on-chain confirmations, use `wagmi's waitForTransactionReceipt`.

### Error Handling

#### Error Types

```javascript
import {
  SdkError,
  AuthorizationError,
  AuthenticationRequiredError,
  NotFoundError,
  ValidationError,
  NetworkError,
  isAuthorizationError,
  isNotFoundError
} from '@moleculexyz/client-sdk';
```

#### Common Errors and Fixes

| Error                       | Cause                 | Fix                                              |
| --------------------------- | --------------------- | ------------------------------------------------ |
| AuthenticationRequiredError | Missing identityToken | Pass auth config to `createDesciSdk()`           |
| AuthorizationError          | No access to resource | Check wallet owns/has access to IP-NFT           |
| NotFoundError               | Invalid ID            | Verify `ipnftUid` format: `{contract}_{tokenId}` |
| NetworkError                | API unreachable       | Retry with exponential backoff                   |

#### Retry Guidance

```javascript
import { isNetworkError } from '@moleculexyz/client-sdk';

async function fetchWithRetry(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (isNetworkError(error) && i < retries - 1) {
        await new Promise(r => setTimeout(r, 1000 * (i + 1)));
        continue;
      }
      throw error;
    }
  }
}
```

### API Reference

#### DesciSdk

Main client class.

```typescript
class DesciSdk {
  tokens: TokensApi;
  ipnfts: IpnftsApi;
  labs: LabsApi;
  crowdsales: CrowdsalesApi;
  minting: MintingApi;
  tokenizer: TokenizerApi;

  updateAuth(auth: AuthConfig): void;
  isAuthenticated: boolean;
}
```

#### Key Methods

| API        | Method                        | Description                      |
| ---------- | ----------------------------- | -------------------------------- |
| tokens     | getAllWithMarkets()           | All IPTs with market data        |
| tokens     | getById(id)                   | Single IPT by contract address   |
| ipnfts     | getById(uid)                  | IP-NFT by `{contract}_{tokenId}` |
| ipnfts     | getByOwner(address)           | All IP-NFTs for wallet           |
| labs       | getAll()                      | All projects                     |
| labs       | getPassphrase(uid)            | DataRoom encryption key (auth)   |
| labs       | initiateFileUploadV2()        | Start file upload (auth)         |
| labs       | finishFileUploadV2()          | Complete file upload (auth)      |
| crowdsales | getById(id)                   | Crowdsale details                |
| crowdsales | getContribution(saleId, user) | User's contribution              |

Full TypeScript types are included in the package.

### Versioning & Compatibility

#### SemVer Policy

* **Major**: Breaking API changes
* **Minor**: New features, backward compatible
* **Patch**: Bug fixes

#### SDK ↔ Contract Compatibility

| SDK Version | Contract Compatibility                     |
| ----------- | ------------------------------------------ |
| 0.x         | IPNFT v2.5, Tokenizer v1.4, CrowdSale v1.x |

The SDK queries indexed data. Contract upgrades may require SDK updates for new fields/events.

#### Breaking Change Policy

Breaking changes are announced in release notes with migration guides. We maintain backward compatibility for at least one minor version when possible.

### Links & Resources

* **Contracts**: `./contracts-overview.md`
* **GitHub**: [Molecule Protocol](https://github.com/moleculeprotocol)
* **NPM**: [Client SDK](https://www.npmjs.com/package/@moleculexyz/client-sdk)
* **Changelog**: See GitHub releases
* **Related SDKs**:
  * `./onchain.md` — Wallet integration.
  * `./storage.md` — File encryption.
  * `./core.md` — Utilities & schemas
* **Support**: [Discord](https://discord.gg/molecule)
