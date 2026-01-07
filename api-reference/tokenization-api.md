# 🔐 Tokenization API

## Overview

The Tokenization API enables developers to programmatically mint IP-NFTs (Intellectual Property NFTs) and tokenize them into fungible IP Tokens (IPTs). This API combines off-chain GraphQL mutations with on-chain blockchain transactions to create legally-bound, tradeable intellectual property assets.

**Capabilities:**
* Mint new IP-NFTs with legal assignment agreements
* Upload artwork and metadata to IPFS
* Create fungible IP Tokens (IPTs) from existing IP-NFTs
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
   * Expected volume of minting/tokenization
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

## IP-NFT Minting Workflow

Creating an IP-NFT involves 9 steps combining API calls and blockchain transactions.

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    IP-NFT MINTING FLOW                      │
└─────────────────────────────────────────────────────────────┘

Step 1: Reserve Token ID (BLOCKCHAIN)
↓ Smart contract: IPNFT.reserve()
↓ Fee: 0.001 ETH
↓ Returns: reservationId

Step 2: Generate Assignment Agreement (API)
↓ Mutation: generateAssignmentAgreement
↓ Returns: agreementCid, agreementContentHash

Step 3: Generate Image Upload URL (API)
↓ Mutation: generateImageUploadUrl
↓ Returns: presigned S3 URL

Step 4: Upload Image (DIRECT S3)
↓ HTTP PUT to presigned URL
↓ Upload: artwork image file

Step 5: Upload Metadata (API)
↓ Mutation: uploadMetadataWithImageKey
↓ Returns: metadataCid, metadataUrl (IPFS)

Step 6: Get Terms Message (API)
↓ Query: getTermsMessage
↓ Returns: message to sign

Step 7: Sign Terms (CLIENT-SIDE)
↓ Wallet signature using viem/ethers
↓ Returns: signature

Step 8: Signoff Metadata (API)
↓ Mutation: signoffMetadata
↓ Returns: authorization signature

Step 9: Mint IP-NFT (BLOCKCHAIN)
↓ Smart contract: IPNFT.mintReservation()
↓ Fee: 0.001 ETH + gas
↓ Result: IP-NFT minted ✓
```

### Key Mutations

#### 1. generateAssignmentAgreement

Generates a legal IP assignment agreement and uploads it to IPFS.

**Input:**
* `projectData` (AWSJSON): Stringified JSON with project details

**Required Fields in projectData:**
```json
{
  "project": {
    "name": "Project Name",
    "description": "Project description",
    "initialSymbol": "SYMBOL",
    "organization": "Organization Name",
    "research_lead": {
      "name": "Dr. Name",
      "email": "email@example.com"
    },
    "topic": "Research Topic",
    "funding_amount": {
      "value": 50000,
      "currency": "USD",
      "currency_type": "ISO4217",
      "decimals": 2
    }
  },
  "connectedWalletAddress": "0x...",
  "chainId": 1,
  "ipnftId": "reservationId"
}
```

**Response:**
```json
{
  "agreementCid": "QmXnnyufdzAWL...",
  "agreementUri": "ipfs://QmXnnyufdzAWL...",
  "agreementContentHash": "0xabcdef...",
  "isSuccess": true
}
```

#### 2. generateImageUploadUrl

Creates a presigned URL for uploading IP-NFT artwork.

**Input:**
* `filename` (String): Image filename
* `contentType` (String): MIME type (e.g., "image/png")
* `ipnftId` (String): Reservation ID from Step 1
* `expiresIn` (Int, optional): URL expiration in seconds

**Response:**
```json
{
  "uploadUrl": "https://s3.amazonaws.com/...",
  "key": "ipnft-images/...",
  "expiresAt": "2024-01-15T10:45:00.000Z",
  "isSuccess": true
}
```

#### 3. uploadMetadataWithImageKey

Uploads NFT metadata with image reference to IPFS.

**Input:**
* `metadata` (AWSJSON): Stringified metadata object
* `imageKey` (String): S3 key from Step 3
* `ipnftId` (String): Reservation ID

**Response:**
```json
{
  "metadataCid": "QmYnZkpT9X...",
  "metadataUrl": "ipfs://QmYnZkpT9X...",
  "isSuccess": true
}
```

#### 4. getTermsMessage

Generates the terms acceptance message for wallet signing.

**Input:**
* `metadataCid` (String): CID from Step 5
* `minter` (String): Wallet address
* `chainId` (Int): Blockchain network ID

**Response:**
```json
{
  "message": "I accept all terms for IPNFT...",
  "digest": "0xabcdef...",
  "isSuccess": true
}
```

#### 5. signoffMetadata

Backend authorization for minting after terms are signed.

**Input:**
* `ipnftId` (String): Reservation ID
* `tokenURI` (String): Metadata URL from Step 5
* `chainId` (Int): Network ID
* `minter` (String): Wallet address
* `to` (String): Recipient address
* `termsSignature` (String): Signature from Step 7

**Response:**
```json
{
  "authorization": "0x1234567890abcdef...",
  "isSuccess": true
}
```

### Example Request (Step 2)

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "mutation GenerateAssignmentAgreement($projectData: AWSJSON!) { generateAssignmentAgreement(projectData: $projectData) { agreementCid agreementUri agreementContentHash isSuccess error { message } } }",
    "variables": {
      "projectData": "{\"project\":{\"name\":\"Research Project\",\"description\":\"Novel research\",\"initialSymbol\":\"RES001\",\"organization\":\"University Lab\",\"research_lead\":{\"name\":\"Dr. Smith\",\"email\":\"smith@uni.edu\"},\"topic\":\"Biomedical\",\"funding_amount\":{\"value\":50000,\"currency\":\"USD\",\"currency_type\":\"ISO4217\",\"decimals\":2}},\"connectedWalletAddress\":\"0x...\",\"chainId\":1,\"ipnftId\":\"123\"}"
    }
  }'
```

***

## IPT Tokenization Workflow

Creating fungible IP Tokens from an IP-NFT involves 5 steps.

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│                IPT TOKENIZATION FLOW                         │
└─────────────────────────────────────────────────────────────┘

Step 1: Verify Ownership (BLOCKCHAIN READ)
↓ Smart contract: IPNFT.ownerOf(tokenId)
↓ Verify: You own the IP-NFT

Step 2: Generate Membership Agreement (API)
↓ Mutation: generateIptMembershipAgreement
↓ Returns: agreementCid

Step 3: Get IPT Terms Message (API)
↓ Query: getIptTermsMessage
↓ Returns: message to sign

Step 4: Sign IPT Terms (CLIENT-SIDE)
↓ Wallet signature using viem/ethers
↓ Returns: signature

Step 5: Tokenize (BLOCKCHAIN)
↓ Smart contract: Tokenizer.tokenizeIpnft()
↓ Fee: Gas only
↓ Result: ERC-20 IPT contract deployed ✓
```

### Key Mutations

#### 1. generateIptMembershipAgreement

Generates IPT membership agreement and uploads to IPFS.

**Input:**
* `agreementData` (AWSJSON): Stringified object

**Required Fields:**
```json
{
  "ipnftId": "123",
  "symbol": "IPT-SYMBOL",
  "title": "IP Token Title"
}
```

**Response:**
```json
{
  "agreementCid": "QmAbc123...",
  "agreementUri": "ipfs://QmAbc123...",
  "agreementContentHash": "0xdef456...",
  "isSuccess": true
}
```

#### 2. getIptTermsMessage

Generates the IPT terms acceptance message for signing.

**Input:**
* `agreementCid` (String): CID from Step 2
* `ipnftId` (String): IP-NFT identifier
* `chainId` (Int): Blockchain network ID

**Response:**
```json
{
  "message": "I accept IPT terms...",
  "digest": "0x789ghi...",
  "isSuccess": true
}
```

### Example Request (Step 2)

```bash
curl -X POST https://production.graphql.api.molecule.xyz/graphql \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY' \
  -d '{
    "query": "mutation GenerateIptAgreement($agreementData: AWSJSON!) { generateIptMembershipAgreement(agreementData: $agreementData) { agreementCid agreementUri isSuccess error { message } } }",
    "variables": {
      "agreementData": "{\"ipnftId\":\"123\",\"symbol\":\"CART-IPT\",\"title\":\"Cancer Research IP Token\"}"
    }
  }'
```

***

## Blockchain Integration

### Smart Contract Addresses

**Mainnet (Ethereum):**
* IPNFT Contract: `0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1`
* Tokenizer Contract: Contact team for address

**Testnet (Sepolia):**
* IPNFT Contract: `0x152B444e60C526fe4434C721561a077269FcF61a`
* Tokenizer Contract: Contact team for address

For complete contract addresses and ABIs, see [Smart Contract Addresses](../ip-nfts/technical-components-of-ip-nfts/smart-contract-addresses.md).

### On-Chain Transactions

The minting and tokenization workflows include blockchain transactions that require:

**Requirements:**
* Ethereum wallet with private key access
* ETH for gas fees
* 0.001 ETH for IP-NFT minting fee
* viem or ethers.js library for transaction signing

**Example (using viem):**

```javascript
import { createWalletClient, http } from 'viem';
import { mainnet } from 'viem/chains';

const walletClient = createWalletClient({
  chain: mainnet,
  transport: http()
});

// Step 1: Reserve token ID
const reserveTx = await walletClient.writeContract({
  address: '0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1',
  abi: IPNFT_ABI, // Request from Molecule team
  functionName: 'reserve',
  value: parseEther('0.001')
});

// Step 9: Mint IP-NFT
const mintTx = await walletClient.writeContract({
  address: '0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1',
  abi: IPNFT_ABI,
  functionName: 'mintReservation',
  args: [
    recipientAddress,
    reservationId,
    metadataUrl,
    symbol,
    authorizationSignature
  ],
  value: parseEther('0.001')
});
```

***

## Requirements

### For IP-NFT Minting

* **API Key**: Obtained from Molecule team
* **Wallet**: Ethereum wallet with private key
* **ETH Balance**: Minimum 0.002 ETH (0.001 reserve + 0.001 mint + gas)
* **Project Data**: Name, description, organization, research lead info
* **Artwork**: Image file (PNG, JPEG, WebP, SVG - max 5MB recommended)

### For IPT Tokenization

* **API Key**: Obtained from Molecule team
* **IP-NFT Ownership**: Must own the IP-NFT you want to tokenize
* **ETH Balance**: Sufficient for gas fees
* **Token Details**: Symbol, name, initial supply amount

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

| Error Code                  | Description                                              | Solution                          |
| --------------------------- | -------------------------------------------------------- | --------------------------------- |
| 401 Unauthorized            | Missing or invalid API key                               | Check `x-api-key` header          |
| 400 Bad Request             | Invalid parameters or malformed JSON                     | Verify input data format          |
| `INVALID_TERMS_SIGNATURE`   | Terms signature doesn't match signer address             | Ensure same wallet signs terms    |
| `INVALID_METADATA`          | Metadata schema validation failed                        | Check required metadata fields    |
| `SYMBOL_ALREADY_TAKEN`      | IP-NFT symbol already exists                             | Use a unique symbol               |
| `ALREADY_TOKENIZED`         | IP-NFT has already been tokenized                        | Each IP-NFT can only be tokenized once |
| `MUST_CONTROL_IPNFT`        | Caller doesn't own the IP-NFT                           | Only IP-NFT owner can tokenize    |

### Troubleshooting

**"INVALID_TERMS_SIGNATURE" Error:**
* Ensure the same wallet address is used throughout the flow
* Verify signature is generated from the exact message returned by `getTermsMessage`
* Check that `minter` parameter matches the signing wallet

**"SYMBOL_ALREADY_TAKEN" Error:**
* Choose a unique symbol for your IP-NFT
* Symbols must be alphanumeric (no special characters)
* Check existing IP-NFTs to avoid conflicts

**Blockchain Transaction Failures:**
* Verify sufficient ETH balance (0.001 ETH + gas)
* Check that wallet has approved spending
* Ensure using correct contract address for your network
* Wait for transaction confirmation before proceeding

***

## Complete End-to-End Examples

For complete code examples including:
* All 9 IP-NFT minting steps with code
* All 5 IPT tokenization steps with code
* Smart contract ABIs and interfaces
* Error handling and retry logic
* Safe multisig integration

**Contact the Molecule team** to receive the full **Technical Integration Guide**.

### Basic Example Structure

```javascript
// 1. Reserve token ID on-chain
const reservationId = await reserveIpnft();

// 2. Generate legal agreement
const agreement = await generateAssignmentAgreement(projectData);

// 3-4. Upload artwork
const imageKey = await uploadArtwork(imageFile);

// 5. Upload metadata to IPFS
const metadata = await uploadMetadata(imageKey, agreement);

// 6-7. Get and sign terms
const termsMessage = await getTermsMessage(metadata.cid);
const termsSignature = await signTerms(termsMessage);

// 8. Get backend authorization
const authorization = await signoffMetadata(termsSignature);

// 9. Mint IP-NFT on-chain
const txHash = await mintReservation(reservationId, metadata.url, authorization);

console.log('IP-NFT minted!', txHash);
```

***

## Smart Contract Details

### IPNFT Contract Functions

**reserve():**
* Reserves a token ID for minting
* Requires: 0.001 ETH payment
* Returns: Reservation ID (emitted in event)

**mintReservation():**
* Mints the reserved IP-NFT
* Parameters:
  * `to` (address): Recipient
  * `reservationId` (uint256): From reserve()
  * `tokenURI` (string): IPFS metadata URL
  * `symbol` (string): IP-NFT symbol
  * `authorization` (bytes): Backend signature
* Requires: 0.001 ETH payment
* Returns: Token ID (emitted in Transfer event)

### Tokenizer Contract Functions

**tokenizeIpnft():**
* Creates ERC-20 IPT from IP-NFT
* Parameters:
  * `ipnftId` (uint256): IP-NFT token ID
  * `tokenAmount` (uint256): Initial IPT supply (in wei)
  * `tokenSymbol` (string): IPT ticker symbol
  * `agreementCid` (string): Membership agreement CID
  * `termsSignature` (bytes): Signed terms acceptance
* Returns: IPT contract address (emitted in event)

**Contract ABIs:**
* Request full ABIs from Molecule team as part of the Technical Integration Guide

***

## Network Information

### Supported Networks

| Network          | Chain ID | IPNFT Contract                               |
| ---------------- | -------- | -------------------------------------------- |
| Ethereum Mainnet | 1        | `0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1` |
| Sepolia Testnet  | 11155111 | `0x152B444e60C526fe4434C721561a077269FcF61a` |

### Gas Fee Estimates

* **Reserve**: ~50,000 gas + 0.001 ETH fee
* **Mint Reservation**: ~150,000 gas + 0.001 ETH fee
* **Tokenize IPNFT**: ~2,000,000 gas (deploys new ERC-20 contract)

_Gas estimates are approximate and vary by network conditions_

***

## Best Practices

### Workflow Management

* Store intermediate results (CIDs, signatures) securely
* Implement retry logic for failed API calls
* Wait for blockchain transaction confirmations
* Validate each step before proceeding to next

### Security

* Never expose API keys in client-side code
* Use environment variables for credentials
* Implement proper wallet key management
* Verify all signatures and authorizations

### Metadata Quality

* Use high-resolution artwork (minimum 512x512px)
* Provide complete project descriptions
* Include all relevant research lead information
* Ensure legal agreement accuracy

### Testing

* Use testnet (Sepolia) for development
* Test complete workflow end-to-end
* Verify metadata renders correctly
* Check IPFS pinning is successful

***

## Getting Support

For assistance with the Tokenization API:

* **Technical Integration Guide**: Contact Molecule team to receive complete documentation
* **Discord**: Join our [community](https://t.co/L0VEiy4Bjk) for support
* **Smart Contracts**: See [contract addresses](../ip-nfts/technical-components-of-ip-nfts/smart-contract-addresses.md)

***

_Last updated: December 2024_
