---
description: >-
  Background information and code examples for implementers who want to interact
  with the IP-NFT protocol in code.
---

# ⚙ Technical Details for Developers

You can find all relevant IP-NFT contract sources on our official [Github repo](https://github.com/moleculeprotocol/IPNFT). Runnable versions of the code samples depicted in this article can be found in the accompanying [samples repo](https://github.com/moleculeprotocol/ipnft-samples).

## Non Fungible Tokens in a Nutshell

Non fungible tokens ([NFTs](https://ethereum.org/en/nft/)) are smart contract based assets that associate a unique token identifier with the blockchain address of its respective owner. The underlying smart contract defines rules on how they are minted (brought into existence), transferred, or burnt (destroyed). It also can restrict their ability to be transferred or offer features that are unlocked for individual token holders.

IP-NFTs are based on [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) which allows users to mint and distribute several tokens of the same "kind", not unlike several instances of a rare - but not unique - gem. An ERC-1155 token kind with only one instance effectively represents assets in the same way as [ERC-721](https://eips.ethereum.org/EIPS/eip-721) does.

The IP-NFT collection contract is deployed on Mainnet and on Görli Testnet, you can find the addresses [here](smart-contract-addresses.md). Each token's metadata is stored as a file descriptor URI (e.g. `ar://HxXKCIE0skR4siRNYeLKI61Vwg_TJ5PJTbxQmtO0EPo`) that must be resolved client side, e.g. by using decentralized storage network gateways ([Arweave](https://arweave.news/introduction-to-the-most-important-infrastructure-in-the-ar-ecosystem-ar-gateway/) or [IPFS](https://ipfs.github.io/public-gateway-checker/)). The contract is non-enumerable, i.e. users can't simply query their owned assets on-chain but instead must rely on reading the respective event logs to build their own off-chain state. We've deployed TheGraph subgraphs [on mainnet](https://api.thegraph.com/subgraphs/name/moleculeprotocol/ip-nft-mainnet) and [on Görli](https://thegraph.com/explorer/subgraph/dorianwilhelm/ip-nft-subgraph-goerli) that can be queried for asset ownership and other IP-NFT related information.

{% hint style="info" %}
Read more about [TheGraph, subgraphs](https://thegraph.com/docs/en/about/#what-the-graph-is) and [how to query them](https://thegraph.com/docs/en/querying/querying-the-graph/) from your application.
{% endhint %}

Here's a GraphQL example of how to query IP-NFT data from the subgraph:

```graphql
query IPNFTsOf($owner: ID!)
{
  ipnfts(where: {owner: $owner}) {
    id
    owner
    createdAt
    tokenURI
  }
}
```

Variables:

```json
{
  "owner": "0xd1f5B9Dc9F5d55523aB25839f8785aaC74EDE98F"
}
```

Result:

```json
{
  "data": {
    "ipnfts": [
      ...
      {
        "id": "21",
        "owner": "0xd1f5b9dc9f5d55523ab25839f8785aac74ede98f",
        "createdAt": "1671818892",
        "tokenURI": "ar://HxXKCIE0skR4siRNYeLKI61Vwg_TJ5PJTbxQmtO0EPo"
      }
    ]
  }
}
```

We've deployed the contracts as [UUPS proxies](https://docs.openzeppelin.com/contracts/4.x/api/proxy#UUPSUpgradeable) owned by the Molecule developer team, thus the contract you're interacting with and the contract that contains the current logic are different. Make sure to always invoke functions on the UUPS proxy - its official addresses can be found [here](smart-contract-addresses.md). The implementation contracts have been verified on Etherscan, so you can [easily retrieve their ABIs](https://etherscan.io/address/0xa4ed1346ba6a4231e35677e73eb7866c05d61725#code).

## Reserving Token IDs & Acquiring Mintpasses

IP-NFTs can generally be minted by arbitrary accounts. However, to keep the collection's items as clean and valid as possible, we added a guard in front of the `mintReservation` method that requires a minter to hold a valid mintpass. To retrieve a mintpass for Mainnet usage, users can submit [this self service form](https://airtable.com/shr9QN0tPPeK4GGjA) that notifies the Molecule team to drop a new mintpass for the requestor. On testnets users can issue mintpasses for themselves, either by following the interactive flow [on the official IP-NFT minting UI](https://ip-nft.molecule.to) or by calling one of our [dedicated mintpass dispenser contract](https://goerli.etherscan.io/address/0x0F1Bd197c5dCC6bC7E8025037a7780010E2Cd22A#writeContract)'s `dispense` methods on-chain.

Once holding a mintpass, the next step on the minting journey is to reserve an IP-NFT token id by calling the IP-NFT contract's `reserve()` method. This capturing step is necessary because the legal documents attached to the final IP-NFT are referring to the NFT's token id that only becomes available after the mint has occurred. Minters will use the token id to craft the legal documents that outline the rights and obligations of owning that NFT in the real world. A selection of premade contract templates for IP-NFTs can be found [on Github](https://github.com/moleculeprotocol/Legal-Contracts).

## Assemble and Upload Metadata

The JSON metadata documents behind IP-NFTs are required to strictly validate against a [well defined JSON schema](https://w3s.link/ipfs/bafybeifyciqacag3wev63lspuvxs3e6fe3n6mt4rwharttxgwpkcq6f5z4/ipnft.schema.json) that's flexible enough to cover many relevant use cases. [Here's a visual tool](https://jsonschema.dev/s/BkeKG) to investigate a valid IP-NFT's metadata interactively. Note that the generic fields `name`, `image` and `description` are located on the document's root level [as required by ERC-1155](https://eips.ethereum.org/EIPS/eip-1155#metadata), whereas the `agreements` and `project_details` structures are modeled as rich property definitions.

```json
{
  "schema_version": "0.0.1",
  "name": "Our awesome test IP-NFT",
  "description": "Lorem ipsum dolor sit amet, ...",
  "image": "ar://7De6dRLDaMhMeC6Utm9bB9PRbcvKdi-rw_sDM8pJSMU",
  "external_url": "https://ip-nft.com/1",
  "properties": {
    "type": "IP-NFT",
    "agreements": [
      {
        "type": "License Agreement",
        "url": "ar://4FG3GR8qCdLo923tVBC85NTYHaaAc3TvCsF3aZwum_o",
        "mime_type": "application/pdf",
        "content_hash": "bagaaiera7ftqs3jmnoph3zgq67bgjrszlqtxkk5ygadgjvnihukrqioipndq",
      }
    ],
    "project_details": {
      "industry": "Pharmaceutical R&D",
      "organization": "Newcastle University, UK",
      "topic": "Aging",
      "research_lead": {
        "first_name": "Chuck",
        "last_name": "Norris",
        "email": "chuck@norris.com"
      }
    }
  }
}

```

### Validate Metadata Correctness

To validate IP-NFT metatadata documents against that schema, you can use arbitrary JSON schema tools. [Ajv](https://ajv.js.org/guide/getting-started.html) is one of the most powerful ones in Javascript. We're omitting [the code](https://github.com/moleculeprotocol/ipnft-samples/blob/main/verifyMetadata.mts) to retrieve schema or documents for brevity's sake but in a nutshell validation boils down to:

```typescript
import Ajv from "ajv";
import addFormats from "ajv-formats";

(async () => {
  const ipnftSchema = JSON.parse(ipnftSchemaJson);
  const document = JSON.parse(ipnftMetadataJson);

  const ajv = new Ajv();
  addFormats(ajv);
  const validateIpnft = ajv.compile(ipnftSchema);
  const validationResult = validateIpnft(document);  
})();
```

### Link to External Resources

IP-NFT metadata documents require you to refer to external resources, e.g. the `image` or `agreements[].url` fields. While you can choose to go with the well known `https://` protocol, it's advisable to use web3-native decentralized storage pointers like `ar://` or `ipfs://` instead. Clients are supposed to resolve them to their respective http gateway counterparts and most NFT related services and frontends can handle them. You don't need to run an Arweave or IPFS node yourself - [Ardrive](https://app.ardrive.io) or web3.storage are excellent helper services that get the job done and our official IP-NFT minting UI uses [Bundlr](https://bundlr.network/) to efficiently upload files to the Arweave network with [near instant finalization](https://docs.bundlr.network/docs/FAQs/Dev-FAQ#what-is-optimistic-finalization).

Here's a nodejs example on how to upload a JSON document using web3.storage (it's simpler to do this [within a browser context](https://web3.storage/docs/how-tos/store/#preparing-files-for-upload), though):

```typescript
import dotenv from "dotenv";
import { Blob } from "node:buffer";
import { Web3Storage } from "web3.storage";
dotenv.config();

const w3sClient = new Web3Storage({ token: process.env.W3S_TOKEN });

(async () => {
  const content = {
    uploaded_at: new Date().toISOString(),
  };

  const binaryContent = new Blob([JSON.stringify(content)], {
    type: "application/json",
  });

  const file = {
    name: "filename.json",
    stream: () => binaryContent.stream(),
  };

  //@ts-ignore
  const cid = await w3sClient.put([file], {
    name: "some.json",
    wrapWithDirectory: false,
  });
  console.log(cid);
})();
```

results in an IPFS CIDv1:

```
bafkreicrhuxfzrydht6tmd4kyy6pkbhspqthswg6xbiqlaztmf774ojxhq
```

To resolve the published content, you can request it from any public IPFS http gateway. You'll experience the lowest latency when querying a gateway close to the node that you used for uploading; web3.storage offers `https://w3s.link` for that purpose. Requesting [https://w3s.link/ipfs/bafkreicrhuxfzrydht6tmd4kyy6pkbhspqthswg6xbiqlaztmf774ojxhq](https://w3s.link/ipfs/bafkreicrhuxfzrydht6tmd4kyy6pkbhspqthswg6xbiqlaztmf774ojxhq) yields

```json
{"uploaded_at":"2023-01-02T22:45:17.788Z"}
```

## Decentralized Encryption and IP-NFT Token Gating

An IP-NFT's most important metadata property are its `agreements`, another term for legal documents attached to it. These documents refer to the IP-NFT smart contract's ("collection") address and the IP-NFT's token id. The agreements' content might contain confidential information about the involved parties and hence should be encrypted before being uploaded. Decentralized and permissionless encryption is a non trivial requirement that we solve by relying on [Lit Protocol](https://developer.litprotocol.com/SDK/Explanation/encryption).

Lit runs a network of nodes that derive signing and encryption keys by multiparty computation / threshold cryptography on trusted computing enclaves. The nodes themselves only know parts of the private key that effectively is fully assembled on the client side after all conditions for key retrieval have been met. Lit protocol allows gating any content behind access control conditions that are backed by blockchain state, therefore it's disclosing decryption keys only to holders of an NFT or users that meet a certain condition on chain.

[Lit's documentation](https://developer.litprotocol.com/SDK/Explanation/encryption#encrypting) lays out the encryption process in detail. To instantiate a Lit SDK instance that's capable of encrypting or decrypting content, [it needs](https://developer.litprotocol.com/sdk/explanation/walletsigs/authsig/#obtaining-the-authsig) an [EIP-4361](https://login.xyz) compatible signature that proves control over the current user's account. Once authenticated we can request a new symmetric key to encrypt our content and ultimately ask the Lit network nodes to store its key shares along with an access control condition. That request yields an encrypted decryption key (😵‍💫) that has been created by the network nodes.

```typescript
  import LitJsSdk from '@lit-protocol/sdk-browser'
  
  const litClient = new LitJsSdk.LitNodeClient()
  await client.connect() //connects to Lit network nodes

  const file: Blob | File = //some file
  const { encryptedFile, symmetricKey } = await LitJsSdk.encryptFile({ file })

  //you can reuse a Siwe / EIP-4361 compatible signed message here, see https://login.xyz/
  const authSig = await LitJsSdk.checkAndSignAuthMessage({ chain: "ethereum" });

  accessControlConditions = {
      conditionType: 'evmBasic',
      contractAddress: tokenContractAddress,
      standardContractType: 'ERC1155',
      chain: "ethereum",
      method: 'balanceOf',
      parameters: [':userAddress', tokenId],
      returnValueTest: {
        comparator: '>',
        value: '0' // user owns more than 0 of this token id
      }
    }

  const u8encryptedSymmetricKey = await litClient.saveEncryptionKey({
    unifiedAccessControlConditions: accessControlConditions,
    symmetricKey,
    authSig,
    chain,
    permanent: false
  })
  
  
```

A user who wants to decrypt the content must again initialize the SDK using a signed message that proves control over their address. Next, they ask several network nodes to present their key shares of the encrypted key by presenting the access control conditions and the encrypted symmetric key to the network. If the network nodes find that the account matches the given conditions, each one yields its key share for the encrypted decryption key. With that, the SDK decrypts the key the content has initially been encrypted with.

```typescript
const authSig = await LitJsSdk.checkAndSignAuthMessage({ chain: "ethereum" });
const symmetricKey = await litClient.getEncryptionKey({
    unifiedAccessControlConditions: accessControlConditions,
    toDecrypt: encryptedSymmetricKey,
    chain: "ethereum",
    authSig
})
```

An IP-NFT metadata's `agreements` item can store the encrypted symmetric key and its access control conditions inside its `encryption` field. Note that the IP-NFT JSON schema of `access_control_conditions` is externally [defined by Lit protocol](https://github.com/LIT-Protocol/lit-accs-validator-sdk/tree/main/src/schemas).

```json
"agreements": [
  {
    "type": "License Agreement",
    "url": "ar://4FG3GR8qCdLo923tVBC85NTYHaaAc3TvCsF3aZwum_o",
    "mime_type": "application/pdf",
    "content_hash": "bagaaiera7ftqs3jmnoph3zgq67bgjrszlqtxkk5ygadgjvnihukrqioipndq",
    "encryption": {
      "protocol": "lit",
      "encrypted_sym_key": "fcbeaf3af31c7104d1d1f7099a6c6f6fda5803a4f7a0bef93256f3377450291872ad07bed3e9402cb47cc932c8f48219e56b84c06becd5ec0ee83ef2c0c93b932fb675c7932fa8df0ad164f17642b32415e382081577a403c19da2eff22c9083fa134ad1f370c2bec449adcdea790498637c7238b7d67cf2d69507a962656d3200000000000000205e4ad4e6323e06862babc934f740bc2e566d175a5da23bb4f1b35635e5cc3768cd040e8776307ea038484ff42033c18f",
      "access_control_conditions": [
        {
          "conditionType": "evmBasic",
          "contractAddress": "0xa1c301d77f701037f491c074e1bd9d4ac24cf5e5",
          "standardContractType": "ERC1155",
          "chain": "goerli",
          "method": "balanceOf",
          "parameters": [
            ":userAddress",
            "6"
          ],
          "returnValueTest": {
            "comparator": ">",
            "value": "0"
          }
        }
      ]
    }
  }
]
```

### Using Multisig Wallet Signers

Due to the high value nature of IP-NFTs you might feel tempted to use a multisig wallet for the minting process, maybe because you'd like to prove that the IP-NFT has been created by a dedicated group of individuals. This works fine for all contract function invocations but is not supported by Lit protocol. Multisig wallets (or contract wallets to be precise) cannot sign messages in the way it's required to authenticate against Lit nodes because they're not based on a private key. This might once be possible by utilizing [EIP-1271](https://eips.ethereum.org/EIPS/eip-1271) compatible wallet signatures but [is not supported](https://discord.com/channels/896185694857343026/916383445784096839/1058018912010248232) at the time of writing.

The recommended workaround is to denote a dedicated trusted member of the multisig that's supposed to intially own the minted IP-NFT. This could be the researcher, a core contributor or maintainer of the project. The IP-NFT contract's `mintReservation` function takes a recipient parameter (`to`) that defines the NFT's initial owner. Note, that the account that _invokes_ the mint function is required to hold a mint pass, not the receiver.

### Granting Read Access to Third Parties

Another shortcoming related to Lit's requirement of private key based authentication signatures is that multisig token holders cannot prove their address to the protocol. To allow multisig members to decrypt the accompanying agreement documents, the IP-NFT contract contains a [`grantReadAccess`](https://goerli.etherscan.io/address/0x36444254795ce6E748cf0317EEE4c4271325D92A#writeProxyContract#F3). It can be invoked by the current token holder (e.g. a multisig) to grant certain accounts (e.g. some of their members or potential buyers) read access to the underlying content for a limited amount of time. Its counterpart [`canRead`](https://goerli.etherscan.io/address/0x36444254795ce6E748cf0317EEE4c4271325D92A#readProxyContract#F3) yields a boolean whether the `reader` is currently allowed access. For the current owner of an IP-NFT this method always returns `true`.

To make read grants work inside Lit protocol, one can craft a [custom contract access control condition](https://developer.litprotocol.com/coreConcepts/accessControl/EVM/customContractCalls) that not only takes the current IP-NFT ownership into account but also lets users pass that currently are granted read access:

```json
"encryption": {
  "protocol": "lit",
  "encrypted_sym_key": "...",
  "access_control_conditions": [
    {
      "conditionType": "evmContract",
      //the IP-NFT UUPS proxy contract address
      "contractAddress": "0x36444254795ce6E748cf0317EEE4c4271325D92A",
      "chain": "goerli",
      "functionName": "canRead",
      "functionParams": [
        ":userAddress",
        "10"
      ],
      "functionAbi": {
        "name": "canRead",
        "inputs": [
          {
            "internalType": "address",
            "name": "reader",
            "type": "address"
          },
          {
            "internalType": "uint256",
            "name": "tokenId",
            "type": "uint256"
          }
        ],
        "outputs": [
          {
            "internalType": "bool",
            "name": "",
            "type": "bool"
          }
        ],
        "stateMutability": "view",
        "type": "function"
      },
      "returnValueTest": {
        "key": "",
        "comparator": "=",
        "value": "true"
      }
    }
  ]
}
```

## Proving Content Integrity

Since agreement documents are encrypted before being stored, each agreement item may contain a `content_hash` that downloaders can use to prove the legal documents' content integrity after they've decrypted it. When using IPFS as storage layer this hash is not adding much value since the network's content ids already provide an untamperable way of guaranteeing content integrity, however [they're not derived from the original content](https://docs.ipfs.tech/concepts/hashing/#content-identifiers-are-not-file-hashes) and hard to prove without an IPFS node at hand.

The `content_hash` field shall contain the sha-256 digest of the attachment's binary content, encoded as a multihash compatible to CIDv1. This allows clients to decode the content hash and verify a document's content without being aware of the hashing algorithm used. This is how it looks like in Typescript using the [multiformats NPM package](https://www.npmjs.com/package/multiformats):

```typescript
import { CID } from "multiformats/cid";
import * as json from "multiformats/codecs/json";
import { sha256 } from "multiformats/hashes/sha2";

const checksum = async (u8: Uint8Array) => {
  //https://multiformats.io/multihash/
  const digest = await sha256.digest(u8);
  return CID.create(1, json.code, digest);
};

const verifyChecksum = async (
  u8: Uint8Array,
  _cid: string
): Promise<boolean> => {
  const cid = CID.parse(_cid);
  //https://github.com/multiformats/multicodec/blob/master/table.csv#L9
  console.log("hash algo used: 0x%s", cid.multihash.code.toString(16));

  const digest = await sha256.digest(u8);
  return cid.multihash.bytes.every((el, i) => el === digest.bytes[i]);
};

(async () => {
  const binaryContent = new TextEncoder().encode("This is the content");
  const cid = await checksum(binaryContent);
  const valid = await verifyChecksum(binaryContent, cid.toString());
  console.log(cid, valid);
})();

```

## Putting it all together: The IP-NFT Minting Flow for Developers

To sum up, minting an IP-NFT technically requires the following steps:

* Get a Mintpass for the account that's supposed to finally invoke the `mintReservation` function.
* invoke the IP-NFT contract's `reserve()` function to reserve a token id
  * it will revert if the caller is not holding a valid mintpass
  * get the reserved token id by parsing the method's event log
* upload an image to a (de)centralised network of your choice
* use the token id and the IP-NFT contract's address to create legal documents
* compute a checksum over the original documents
* optionally encrypt the documents with a Lit access control condition
* assemble a metadata structure containing the file pointers, access control conditions, encrypted symmetric key and checksum
* verify that this metadata structure is valid
* upload the metadata to a (de)centralised network of your choice
* invoke `mintReservation` on the IP-NFT contract with
  * `address to`: the recipient of the minted NFT (e.g. a multisig wallet)
  * `uint256 reservationId`: the token id you've reserved
  * `uint256 mintPassId`: the mintpass id that's going to be redeemed during the mint
  * `string memory tokenURI`: the URI that resolves to the metadata
