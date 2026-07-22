# 🔐 Tokenization API

## Overview

The Tokenization API enables developers to generate Lab (OCL) membership agreements and tokenize Labs into IP Tokens (IPTs) on Base. It combines offchain GraphQL mutations with onchain blockchain transactions to create legally-bound, tradeable research assets.

**Capabilities:**

* Generate Lab (OCL) membership agreements
* Tokenize Labs into IP Tokens (IPTs) on Base
* Manage the complete tokenization lifecycle
* Integrate with smart contracts via viem/ethers

***

## Authentication

All Tokenization API mutations require an API key.

### Obtaining an API Key

To request an API key and access to the full technical integration guide:

1. Join our [Discord community](https://t.co/L0VEiy4Bjk)
2. Contact the Molecule team with:
   * Your use case and project details
   * Expected tokenization volume
3. You'll receive:
   * **API Key** for authentication
   * **Technical Integration Guide** with complete code examples and ABI files

### Using Your API Key

Include the API key in all requests using the `x-api-key` header:

```bash
x-api-key: YOUR_API_KEY
```

***

## API Endpoint

```
Production: https://production.graphql.api.molecule.xyz/graphql
Staging:    https://staging.graphql.api.molecule.xyz/graphql
```

***

## OCL Membership Agreement

Generate the membership agreement for an onchain lab (OCL) — the terms document an IP Token holder accepts when the Lab is tokenized. It returns an `agreementKey` (an S3 object key) paired with a SHA-256 `agreementContentHash` that binds the signature to the exact document.

### generateOclMembershipAgreement

**Input:**

* `agreementData` (AWSJSON): Stringified object with the fields below.

**Required Fields in agreementData:**

```json
{
  "oclId": "0x0101000000000000000000000000000000000000000000000000000000000042",
  "symbol": "LAB-SYMBOL",
  "title": "Lab Membership Agreement"
}
```

| Field  | Type   | Required | Description                                          |
| ------ | ------ | -------- | ---------------------------------------------------- |
| oclId  | String | Yes      | Canonical 32-byte oclId of the lab (0x + 64 hex)     |
| symbol | String | Yes      | Lab symbol                                           |
| title  | String | No       | Optional agreement title                             |

**Response:**

```json
{
  "agreementKey": "0x0101...0042/agreements/3f2a....json",
  "agreementUrl": "https://...",
  "agreementContentHash": "0xabcdef...",
  "agreementType": "OCL_MEMBERSHIP",
  "generatedAt": "2026-07-15T10:30:00.000Z",
  "isSuccess": true
}
```

**Example Request:**

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "mutation GenerateOclMembershipAgreement($agreementData: AWSJSON!) { generateOclMembershipAgreement(agreementData: $agreementData) { agreementKey agreementUrl agreementContentHash agreementType generatedAt isSuccess error { message } } }",
    "variables": {
      "agreementData": "{\"oclId\":\"0x0101000000000000000000000000000000000000000000000000000000000042\",\"symbol\":\"LAB-SYM\",\"title\":\"Lab Membership Agreement\"}"
    }
  }'
```

> The returned `agreementKey` and `agreementContentHash` are consumed by the `getOclTermsMessage(agreementKey, contentHash, labId, chainId)` query to produce the message the member signs.

***

## Lab (OCL) Tokenization Workflow

Labs are tokenized on **Base** through the `OclTokenizer` contract. It mints one fractional ERC-20 **IP Token (IPT)** per Lab — deployed as a `LabToken` contract — gated by a signed membership agreement.

### High-Level Flow

```
Step 1: Generate Membership Agreement (API)
↓ Mutation: generateOclMembershipAgreement
↓ Returns: agreementKey (S3 key), agreementContentHash (SHA-256)

Step 2: Get OCL Terms Message (API)
↓ Query: getOclTermsMessage(agreementKey, contentHash, labId, chainId)
↓ Returns: the exact terms text to sign

Step 3: Sign Terms (CLIENT-SIDE)
↓ Plain personal_sign (EIP-191) over the message — not EIP-712
↓ ECDSA signatures and ERC-1271 smart-account signatures both accepted

Step 4: Tokenize (BLOCKCHAIN, Base)
↓ Smart contract: OclTokenizer.tokenize()
↓ Fee: Gas only
↓ Result: ERC-20 IP Token (LabToken) contract deployed ✓
```

The terms message is reconstructed onchain by `OclTermsPermissioner.specificTermsV1()` — the backend's `getOclTermsMessage` returns byte-identical text, so always sign exactly the string the API returns.

### OclTokenizer Contract Functions

**tokenize():**

* Creates the Lab's ERC-20 IP Token
* Parameters:
  * `labId` (uint256): The LabNFT token ID
  * `amount` (uint256): Initial supply issued to the caller (in wei)
  * `symbol` (string): Token ticker symbol — the name is derived automatically as `Lab Tokens of Lab #<labId>`
  * `s3Key` (string): `agreementKey` from Step 1
  * `contentHash` (bytes32): `agreementContentHash` from Step 1
  * `signature` (bytes): Signed terms from Step 3
* Caller must be the Lab's controller (the current LabNFT owner); reverts `MustControlLab()` otherwise
* One token per Lab — a second call reverts `AlreadyTokenized()`

**attachToken():**

* "Bring your own token": wraps a pre-existing ERC-20 (≤ 18 decimals) in a read-only `WrappedLabToken` carrying the Lab's metadata instead of minting a new one
* Parameters: `labId`, `s3Key`, `contentHash`, `signature`, `tokenContract`

**issue() / cap():**

* `issue(labToken, amount, receiver)` — mint additional supply; controller-only
* `cap(labToken)` — permanently freeze issuance; controller-only, irreversible

### Networks (Lab Tokenization)

| Network      | Chain ID | OclTokenizer (proxy)                         |
| ------------ | -------- | -------------------------------------------- |
| Base Mainnet | 8453     | `0x62F532C3f563D974deEc103AAb8cC597f4f9c84E` |
| Base Sepolia | 84532    | `0xEe19e0Db8a7e59538710FAF6ed3ab655BCfCdB24` |

***

## Requirements

### For Lab Tokenization

* **API Key**: Obtained from Molecule team
* **Lab Control**: The caller must be the Lab's controller (the current LabNFT owner)
* **Base ETH Balance**: Sufficient for gas fees on Base
* **Token Details**: Symbol and initial supply amount (the token name is derived automatically)

***

## Error Handling

All mutations follow a consistent error response format:

```json
{
  "isSuccess": false,
  "error": {
    "message": "Error description",
    "code": "ERROR_CODE",
    "retryable": true
  }
}
```

### Common Errors

| Error Code                | Description                                  | Solution                               |
| ------------------------- | -------------------------------------------- | -------------------------------------- |
| 401 Unauthorized          | Missing or invalid API key                   | Check `x-api-key` header               |
| 400 Bad Request           | Invalid parameters or malformed JSON         | Verify input data format               |
| `INVALID_INPUT`           | Required fields missing or malformed         | Verify the input object shape          |

Separately, the smart contracts revert with their own errors — most commonly `AlreadyTokenized()` (each Lab can only be tokenized once) and `MustControlLab()` (only the Lab's controller can tokenize) from the `OclTokenizer`.

### Troubleshooting

**Signature rejected onchain:**

* Sign exactly the string returned by `getOclTermsMessage` — the permissioner reconstructs the terms onchain and the texts must match byte-for-byte
* Use a plain `personal_sign` (EIP-191) signature, and sign with the Lab controller's wallet (or a smart account that validates via ERC-1271)

**Blockchain Transaction Failures:**

* Verify sufficient ETH balance for gas on Base
* Ensure using correct contract address for your network
* Wait for transaction confirmation before proceeding

***

## Complete End-to-End Examples

For complete code examples including:

* The full Lab tokenization flow with code
* Smart contract ABIs and interfaces
* Error handling and retry logic
* Safe multisig integration

**Contact the Molecule team** to receive the full **Technical Integration Guide**.

### Basic Example Structure

```javascript
// 1. Generate the membership agreement
const agreement = await generateOclMembershipAgreement(agreementData);

// 2. Get the exact terms text to sign
const terms = await getOclTermsMessage(
  agreement.agreementKey, agreement.agreementContentHash, labId, chainId
);

// 3. Sign the terms (plain personal_sign)
const signature = await walletClient.signMessage({ message: terms.message });

// 4. Tokenize the Lab onchain (Base)
const txHash = await tokenize(labId, initialSupply, symbol,
  agreement.agreementKey, agreement.agreementContentHash, signature);

console.log('Lab tokenized!', txHash);
```

***

## Smart Contract Details

**Contract ABIs:**

* Available from the verified [OclTokenizer on BaseScan](https://basescan.org/address/0x62F532C3f563D974deEc103AAb8cC597f4f9c84E), or request them from the Molecule team as part of the Technical Integration Guide

***

## Best Practices

### Workflow Management

* Store intermediate results (agreement keys, signatures) securely
* Implement retry logic for failed API calls
* Wait for blockchain transaction confirmations
* Validate each step before proceeding to next

### Security

* Never expose API keys in client-side code
* Use environment variables for credentials
* Implement proper wallet key management
* Verify all signatures and authorizations

### Testing

* Use Base Sepolia for development
* Test the complete workflow end-to-end
* Verify the agreement document and terms text before signing

***

## Getting Support

For assistance with the Tokenization API:

* **Technical Integration Guide**: Contact Molecule team to receive complete documentation
* **Discord**: Join our [community](https://t.co/L0VEiy4Bjk) for support
* **Smart Contracts**: See [contract addresses](../references/contracts/README.md)

***

_Last updated: July 2026_
