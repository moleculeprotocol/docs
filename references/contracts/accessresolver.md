---
icon: gavel
---

# AccessResolver

## AccessResolver Overview

The `AccessResolver` contract facilitates on-chain authorization checks for [Lit Protocol](https://litprotocol.com/) access control. It verifies if a wallet address is authorized to decrypt files associated with an IP-NFT. This contract acts as a bridge between Molecule's IP-NFT ownership system and Lit Protocol's decentralized encryption network, ensuring secure file sharing within DataRooms.

### Contract Details

| Property | Value                  |
| -------- | ---------------------- |
| Contract | AccessResolver         |
| Type     | UUPS Upgradeable Proxy |
| Solidity | ^0.8.x                 |
| License  | MIT                    |

#### Deployments

* **Ethereum Mainnet**
  * Address: `0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0`
  * [Verified on Etherscan](https://etherscan.io/address/0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0)
* **Sepolia Testnet**
  * Address: `0xd9b492fd34b1579c052b2ea25970178b3011ce6b`
  * [Verified on Etherscan](https://sepolia.etherscan.io/address/0xd9b492fd34b1579c052b2ea25970178b3011ce6b)

### How It Works

1. **Lit Protocol Network** interacts with the `AccessResolver`.
2. `AccessResolver` determines if the user owns or can read the IPNFT contract.
3. Grants or denies decryption keys based on the authorization.

#### Functions

*   **isAuthorizedSignerForIpnft**: Checks if an address is authorized for a specific IP-NFT.

    ```solidity
    function isAuthorizedSignerForIpnft(address signer, uint256 ipnftId) 
    external view returns (bool)
    ```
*   **isAuthorizedSignerForTba**: Determines if an address can act on behalf of a Token Bound Account.

    ```solidity
    function isAuthorizedSignerForTba(address signer, address account) 
    external view returns (bool)
    ```
* **ipnftContractAddress**: Returns the IPNFT contract address used for checks.

#### Events

* **Initialized(uint64 version)**: Emitted when the contract is initialized.
* **OwnershipTransferred(address previousOwner, address newOwner)**: Emitted on ownership change.
* **Upgraded(address implementation)**: Emitted when the implementation is upgraded.

#### Errors

| Error                               | Description                                        |
| ----------------------------------- | -------------------------------------------------- |
| OwnableUnauthorizedAccount(address) | Caller is not authorized for owner-only functions. |
| OwnableInvalidOwner(address)        | Invalid owner address provided.                    |
| UUPSUnauthorizedCallContext()       | Upgrade called in an incorrect context.            |

### Integration Guide

#### With Lit Protocol

Utilize `AccessResolver` as an access control condition in file encryption.

```js
import { createAccessCondition } from '@moleculexyz/storage';

const accessCondition = createAccessCondition(
  { type: 'authorized_ipnft_signer', ipnftId: '42' },
  'ethereum',
  {
    accessResolver: '0xc130e0b49840b266A49F62C0Cc77e353E0C99cD0',
    ipnft: '0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1'
  }
);

// Encryption with Lit Protocol
const encrypted = await litClient.encrypt({
  accessControlConditions: [accessCondition],
  dataToEncrypt: fileBuffer
});
```

#### With React Hooks

```js
import { useLitEncryption } from '@moleculexyz/storage';

function DataRoom({ ipnftId }) {
  const { encryptFile, decryptFile } = useLitEncryption();

  const handleUpload = async (file) => {
    const encrypted = await encryptFile(file, {
      type: 'authorized_ipnft_signer',
      ipnftId
    });
  }

  const handleDownload = async (ciphertext, hash) => {
    const decrypted = await decryptFile(ciphertext, hash, {
      type: 'authorized_ipnft_signer',
      ipnftId
    });
  }
}
```

#### Direct Contract Call

```js
import { useAuthorizedIpnftSigner } from '@moleculexyz/onchain';

function AccessCheck({ ipnftId, userAddress }) {
  const { data: isAuthorized, isLoading } = useAuthorizedIpnftSigner(ipnftId, userAddress);

  return isLoading ? <span>Checking access...</span> : <span>{isAuthorized ? 'Authorized' : 'Not authorized'}</span>;
}
```

### Access Control Conditions

* **ipnft\_read**: Grants time-limited read access.
* **authorized\_ipnft\_signer**: Provides full authorization.

### Security Considerations

* **Read-only nature**: The contract performs authorization checks but does not modify ownership or access.
* **Failure handling**: Access is denied if contract call fails.
* **Network verification**: Ensure querying on the correct network.

### Related Contracts

* IP-NFT: The contract queried for ownership details.
* Tokenizer: Converts IP-NFTs into IPTokens.

### Resources

* **ABI**: Available within `@moleculexyz/common/abis`
* **React Hooks**: `@moleculexyz/onchain` and `@moleculexyz/storage`
* **Lit Protocol Documentation**: [Lit Protocol Developer Docs](https://developer.litprotocol.com/)
