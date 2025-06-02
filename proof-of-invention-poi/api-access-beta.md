# ⚙️ API Access (Beta)

## Overview

The PoI API allows developers to programmatically generate cryptographic proofs for their invention files. This API is currently in **beta** and requires an API key for access.

> **Important**: This API is currently in beta. To request API access, please join our [Discord community](https://t.co/L0VEiy4Bjk) and reach out to our team to obtain an API key.

### Authentication

All API requests require authentication using an API key. Use the Bearer token authentication method:

```
Authorization: Bearer YOUR_API_KEY
```

### API Endpoints

#### Generate Proof of Invention

Creates a merkle tree from uploaded files and returns the merkle root along with transaction data.

**Endpoint:** `POST /api/v1/inventions`

**Content-Type:** `multipart/form-data`

**Request Parameters:**

| Parameter | Type    | Required | Description                                         |
| --------- | ------- | -------- | --------------------------------------------------- |
| files     | File\[] | Yes      | Array of files to include in the proof of invention |

**Example Request:**

```bash
curl -X POST \
  https://poi.molecule.xyz/api/v1/inventions \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -H 'Content-Type: multipart/form-data' \
  -F 'files=@document1.pdf' \
  -F 'files=@document2.pdf'
```

**Success Response:**

```json
{
  "success": true,
  "data": {
    "proof": {
      "format": "simple-v1",
      "tree": [
        "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
        "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
        "0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba"
      ],
      "values": [
        {
          "value": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
          "treeIndex": 1
        },
        {
          "value": "0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba",
          "treeIndex": 2
        }
      ]
    },
    "transaction": {
      "data": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
      "to": "0x1DEA29b04a59000b877979339a457d5aBE315b52"
    }
  },
  "metadata": {
    "supportedEvmChainIds": [1, 8453],
    "apiVersion": "1.0",
    "timestamp": "2025-05-22T15:30:45.123Z"
  }
}
```

### Error Handling

The API returns appropriate HTTP status codes along with error messages in the response body:

#### Error Response Format

```json
{
  "success": false,
  "error": {
    "message": "Error message describing what went wrong",
    "code": 400
  },
  "metadata": {
    "supportedEvmChainIds": [1, 8453],
    "apiVersion": "1.0",
    "timestamp": "2025-05-22T15:31:12.456Z"
  }
}
```

#### Common Error Codes

| Status Code | Description                                                |
| ----------- | ---------------------------------------------------------- |
| 400         | Bad Request - Invalid input parameters                     |
| 401         | Unauthorized - Invalid or missing API token                |
| 415         | Unsupported Media Type - Invalid Content-Type header       |
| 500         | Internal Server Error - Something went wrong on the server |

### Next Steps After API Response

After receiving a successful response from the API, you'll need to submit the transaction to the blockchain to store your proof on-chain:

1. Use the `transaction` object from the API response
2. Submit a transaction to any supported EVM blockchain (listed in `supportedEvmChainIds`)
3. Use the `payload` as transaction data and `recipient` as the recipient address

#### Example (using viem)

```javascript
const tx = await walletClient.sendTransaction({
  data: result.transaction.data,  // Merkle root from API response
  to: result.transaction.to,  // Contract address from API response
  from: yourWalletAddress,
  ...
})

// Wait for transaction confirmation
const receipt = await waitForTransactionReceipt(walletClient, { hash: tx })
```

This transaction creates a permanent, timestamped record of your proof of invention on the blockchain.

### File Requirements

* Maximum file size: 100MB (total for all files)
* Supported file types: All file types are supported

### Usage Guidelines

1. **API Rate Limits**: During the beta period, the API is limited to 100 requests per day per API key.
2. **File Storage**: Files are processed to create the merkle tree but are not stored on our servers.
3. **Transaction Handling**: The API returns the transaction data, but you are responsible for submitting the transaction to the blockchain.

### Getting Support

If you encounter any issues or have questions about the API, please join our [Discord community](https://t.co/L0VEiy4Bjk) for support.

### Beta Program

As part of our beta program, we're actively seeking feedback to improve the API. If you have suggestions or feature requests, please share them with us on Discord.

***

_Last updated: May 22, 2025_
