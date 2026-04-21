---
description: >-
  How legal IP rights are registered, minted, and transferred as non-fungible
  tokens onchain
icon: hexagon-vertical-nft
---

# IP-NFTs

### Why IP-NFTs Exist

Scientific intellectual property has historically been locked inside institutional filing cabinets — governed by paper agreements, tracked in spreadsheets, and transferred through months-long legal negotiations. The result is an illiquid, opaque system where promising research sits unfunded because there is no efficient way to represent, transfer, or fractionalize ownership of the underlying IP rights.

IP-NFTs solve this by creating a digital, onchain representation of IP ownership that is legally enforceable, programmable, and transferable. By attaching signed legal agreements to an ERC-721 token, IP-NFTs turn intellectual property into a composable onchain asset — one that can be held inside an Onchain Lab, tokenized into fractional ownership, licensed to third parties, and used to gate access to the underlying research data.

The core insight is that the NFT doesn't _replace_ the legal agreement — it _wraps_ it. The legal rights still exist offchain, governed by real contracts. The IP-NFT creates a verifiable onchain record of who holds those rights, and ensures that when ownership transfers, the legal rights transfer with it.

### What Makes Up an IP-NFT

An IP-NFT is composed of three layers working together:

**Onchain Layer — The Token** The IP-NFT itself is an ERC-721 token on Ethereum. It carries a unique token ID, an owner address, a metadata URI pointing to IPFS, and a custom symbol identifier. The smart contract enforces ownership, transfer logic, access control, and minting authorization. This is the programmable layer that makes IP composable with the rest of the protocol.

**Legal Layer — The Agreements** Attached to every IP-NFT are one or more signed legal agreements stored on IPFS. These are real, enforceable contracts — not just metadata. The legal layer is what gives the token its substance: without the agreements, the NFT is just a pointer. With them, it represents actual IP rights. Every IP-NFT requires at minimum an Assignment Agreement that binds the legal rights to the token's current holder.

**Data Layer — The Research** IP-NFTs connect to the Lab's data room, where the actual research data lives. The token's access control functions determine who can decrypt and view the underlying datasets, manuscripts, and experimental results. This connection means that owning or holding tokens in an IP-NFT doesn't just give you a legal claim — it gives you access to the science itself.

### Where IP-NFTs Live

In the V3 architecture, IP-NFTs are held inside Onchain Labs — specifically within the Lab's Token Bound Account (TBA). This means the IP-NFT's effective owner is whoever controls the parent Lab (the LabNFT holder). When a LabNFT is transferred, the new holder gains control of the Lab and all IP-NFTs inside it in a single transaction.

This creates a clear ownership hierarchy: wallet holds LabNFT → LabNFT controls Lab TBA → Lab TBA holds IP-NFTs → IP-NFTs can be tokenized into IPTs.

IP-NFTs can also exist outside of Labs (held directly in wallets), but the V3 architecture is designed around Lab-based custody, where the Lab's modular account provides governance, treasury management, and data room integration around the IP asset.

### Legal Framework

IP-NFTs bridge offchain legal rights with onchain ownership through a two-agreement structure: a Research Agreement that defines the IP rights, and an Assignment Agreement that binds those rights to the token.

#### Research Agreements

Research Agreements define the underlying IP and data rights. The protocol supports multiple agreement types depending on the source and nature of the IP:

| Agreement Type                     | Use Case                           | Typical Source                      |
| ---------------------------------- | ---------------------------------- | ----------------------------------- |
| Sponsored Research Agreement (SRA) | Funded academic research           | Universities, Research Institutions |
| Joint Development Agreement (JDA)  | Collaborative R\&D between parties | Industry-Academic Partnerships      |
| Patent License                     | Existing patented IP               | Patent Holders, TTOs                |
| SAFIP                              | Early-stage IP with future rights  | Startups, Independent Researchers   |
| SAFE                               | Equity-linked IP arrangements      | Companies, Biotech Startups         |

**Sponsored Research Agreement (SRA)** — the most common agreement type for academic IP. Entered between a researcher/institution and a sponsor, granting IP and R\&D data rights in exchange for funding. Required elements include scope of work, deliverables, timeline, budget, IP rights, data rights, and confidentiality provisions.

**SAFIP (Simple Agreement for Future IP)** — a streamlined agreement for early-stage research where IP has not yet been generated. The SAFIP grants rights to future IP arising from specified research activities. Use cases include pre-clinical research funding, early discovery programs, and computational research with uncertain outputs.

**SAFE (Simple Agreement for Future Equity)** — used when IP rights are bundled with equity considerations, typically in corporate spinouts or startup contexts.

#### Assignment Agreement

The Assignment Agreement is the critical legal bridge between the Research Agreement and the IP-NFT. It transfers the contractual position from the assignor to the IP-NFT holder and includes an automatic transfer clause: when the IP-NFT changes hands — whether through direct transfer, Lab acquisition (LabNFT sale), or marketplace transaction — the associated legal rights transfer to the new owner.

Key provisions include transfer of contractual position from assignor to assignee, automatic rights transfer following NFT ownership, representations and warranties, and digital signature binding. Signatures are verified onchain using EIP-191 (for externally owned accounts) and EIP-1271 (for smart contract wallets like Lab TBAs), creating an auditable, tamper-evident record of consent.

### Minting Pathways

There are two paths to minting an IP-NFT, depending on whether the IP originates from an institution or from an independent researcher.

{% tabs %}
{% tab title="Research Agreement Flow" %}
**Path 1: Research Agreement Flow**

For IP originating from institutions, universities, or corporate research:

1. Execute Research Agreement (SRA, JDA, SAFIP, SAFE, or Patent License)
2. Negotiate with TTO/Legal (if applicable)
3. Execute Assignment Agreement
4. Upload agreements to IPFS
5. Mint IP-NFT with attached agreements

Required inputs: signed Research Agreement (PDF), contract title, owner, type, and signature date, and Assignment Agreement with target wallet address (typically the Lab's TBA).
{% endtab %}

{% tab title="POI Flow" %}
**Path 2: Proof of Invention (POI) Flow**

For independent researchers with original inventions:

1. Create Proof of Invention (generates merkle root hash)
2. Execute Assignment Agreement
3. Upload POI and agreement to IPFS
4. Mint IP-NFT with attached documents

Required inputs: invention disclosure documents, POI merkle root hash (cryptographic proof of invention date), and Assignment Agreement with target wallet address.

The POI flow bypasses the Research Agreement step when there is no external funding or institutional involvement.
{% endtab %}
{% endtabs %}

#### Minting Process

**Step 1: Reserve Token ID**

```
reserve() → reservationId
```

Creates a reservation for a future token ID. Reservations are valid until used.

**Step 2: Prepare Metadata**

Upload artwork and agreements to IPFS, generate metadata JSON with agreement references, and optionally encrypt sensitive documents via Lit Protocol.

**Step 3: Sign Agreements**

The minter signs the Assignment Agreement (legally binding) and an onchain signature linking their wallet to their legal identity.

**Step 4: Mint**

```
mintReservation(to, reservationId, tokenURI, symbol)
```

Mints the IP-NFT to the specified address with attached metadata. In the V3 flow, the `to` address is typically the Lab's Token Bound Account, placing the newly minted IP-NFT directly inside the Lab.

### What You Can Do with an IP-NFT

Once minted inside a Lab, IP-NFTs become the anchor for a range of protocol interactions:

<table><thead><tr><th width="231.1328125">Action</th><th>Description</th></tr></thead><tbody><tr><td><strong>Tokenized</strong></td><td>Create IPTs (governance tokens) via the Tokenizer contract, enabling fractional ownership and fundraising</td></tr><tr><td><strong>Funded</strong></td><td>Launch fundraising campaigns to raise capital into the Lab's treasury</td></tr><tr><td><strong>Licensed</strong></td><td>Grant temporary usage rights via ERC-4907 rentable NFT mechanism — a licensee pays a fee in stablecoins for time-bound access to the IP</td></tr><tr><td><strong>Data-gated</strong></td><td>Gate access to the Lab's data room based on IP-NFT read permissions, enabling token-holders to access premium research data</td></tr><tr><td><strong>Governed</strong></td><td>IPT holders vote on decisions affecting the IP: milestone approvals, licensing terms, treasury allocation, clawback proceedings</td></tr><tr><td><strong>Transferred</strong></td><td>Transfer the IP-NFT directly, or transfer the entire Lab (including all IP-NFTs) by selling the LabNFT</td></tr><tr><td><strong>Revenue-generating</strong></td><td>Licensing fees, dataset access payments, and milestone payouts flow into the Lab's treasury, distributable to IPT holders</td></tr></tbody></table>

#### IP Licensing via ERC-4907

The V3 architecture supports onchain IP licensing through the ERC-4907 rentable NFT standard. A pharmaceutical company or research institution can license an IP-NFT for a defined period by executing a single transaction. The licensor (Lab) sets the rental terms — duration, fee amount, and payment token. The licensee pays the fee, which is automatically routed to the Lab's treasury. For the duration of the rental period, the licensee has a verifiable onchain record of their license grant, while the Lab retains ownership of the IP-NFT.

Royalties from licensing revenue can be distributed to the original scientist, the institution, and other stakeholders via ERC-2981 royalty standard. Complex licensing arrangements that require custom legal terms may still involve off-chain agreements, but the standard licensing transaction is fully executable onchain.

### Technical Reference

#### Data Model

solidity

```solidity
struct IPNFT {
    uint256 tokenId;
    address owner;
    string tokenURI;   // IPFS metadata URI
    string symbol;     // Custom identifier
}
```

Metadata (IPFS):

json

````json
{
  "name": "Project Name",
  "description": "Research description",
  "image": "ipfs://...",
  "agreements": [
    {
      "type": "Sponsored Research Agreement",
      "url": "ipfs://...",
      "content_hash": "...",
      "encryption": { ... }
    },
    {
      "type": "Assignment Agreement",
      "url": "ipfs://...",
      "content_hash": "..."
    }
  ]
}
```

#### Agreement Schema

| Field | Type | Description |
|---|---|---|
| type | string | Agreement type (SRA, JDA, SAFIP, SAFE, Patent License, Assignment) |
| url | string | IPFS or Arweave storage location |
| content_hash | string | CID for integrity verification |
| encryption | object | Optional Lit Protocol encryption metadata |
| amends | string | CID of previous agreement if this is an amendment |

### Access Control

IP-NFT holders can grant time-limited read access to specific addresses:
```
grantReadAccess(address reader, uint256 tokenId, uint256 until)
````

Access is verified via the `canRead` mapping. This is the mechanism that integrates with the data layer: when a user requests access to an encrypted file in a Lab's data room, Lit Protocol queries the IP-NFT's `canRead` function to determine whether the requester has permission to decrypt. Access levels (Public, Token-Holder, Admin-Only) are enforced at the file level through this onchain verification. For details on how encryption and access gating work, see the **Data Privacy & Access** page.

#### Agreement Amendments

Agreements can be amended post-mint by attaching new versions. The process involves creating an amended agreement referencing the original CID, uploading to IPFS with an `amends` field pointing to the previous version, and updating the IP-NFT metadata to include the new agreement. Amendment history is preserved through the amends chain, providing a full audit trail.

### Contract Addresses

**Mainnet**

<table><thead><tr><th width="253.37890625">Contract</th><th>Address</th></tr></thead><tbody><tr><td>IP-NFT</td><td><code>0xcaD88677CA87a7815728C72D74B4ff4982d54Fc1</code></td></tr><tr><td>SignedMintAuthorizer</td><td><code>0xBc5FbB45A2bbB64d9B2EeBFa327284a35d5C5865</code></td></tr></tbody></table>

**Sepolia**

<table><thead><tr><th width="256.890625">Contract</th><th>Address</th></tr></thead><tbody><tr><td>IP-NFT</td><td><code>0x152B444e60C526fe4434C721561a077269FcF61a</code></td></tr><tr><td>Authorizer</td><td><code>0x7a9F3773352e4ee0Da6307Cd32C45fE89602129A</code></td></tr></tbody></table>

### Security Considerations

The IP-NFT contract uses the UUPS proxy pattern (upgradeable). The owner can pause minting and transfers. Mint authorization is configurable via the `IAuthorizeMints` interface — the current deployment uses a SignedMintAuthorizer that validates signatures from an authorized minting authority. Agreements should be encrypted for sensitive IP using Lit Protocol before upload. Assignment Agreements should include dispute resolution provisions to handle conflicts between onchain ownership and offchain legal interpretation.
