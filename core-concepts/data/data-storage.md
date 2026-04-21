---
description: >-
  How research files are stored, content-addressed, versioned, and permanently
  persisted across the decentralised storage stack
icon: hashtag-lock
---

# Data Storage

### How Storage Works

Every file uploaded to an Onchain Lab passes through a layered storage pipeline that encrypts, distributes, versions, and references data across multiple decentralised systems. The result is a file that is encrypted before it leaves the researcher's browser, pinned to a content-addressed network for retrieval, persisted permanently independent of any single provider, versioned with full provenance history, and referenced on-chain in the Lab's Token Bound Account.

### Upload Flow

When a researcher uploads a file to a Lab, the following sequence occurs:

**Authenticate.** The researcher signs a SIWE (Sign-In with Ethereum) message with their wallet, establishing a session with Lit Protocol. Session signatures are returned and remain valid for approximately 24 hours, authorising the researcher to encrypt files against the Lab's access conditions.

**Prepare.** The Client SDK computes a content hash checksum of the raw file. If the file's access level is set to Token-Holder or Admin-Only, the SDK creates access conditions based on the Lab's IP-NFT (using the IPNFT canRead permission), encrypts the file client-side using a symmetric encryption key, and shards that key across Lit Protocol's node network using threshold cryptography. The encrypted blob replaces the original file in memory. If the access level is Public, no encryption is required and the raw file proceeds directly.

**Upload.** The Client SDK initiates the upload through the Molecule API, which reserves an upload slot and generates a pre-signed URL via Filebase (the S3-compatible upload gateway). The encrypted file is uploaded directly from the browser to the pre-signed URL with progress tracking.

**Commit.** Once the upload completes, Kamu fetches the file from staging, creates a new version record (recording the timestamp, content hash, author, and data room path), and commits the file to IPFS via the pinning service. IPFS returns a content identifier (CID) derived from the file's contents. The file is also persisted to Arweave for permanent availability. The temporary staging file is deleted.

**Reference.** The CID and associated metadata are written on-chain to the Lab's Token Bound Account. This creates a tamper-evident, publicly auditable link between the Lab's onchain identity and the off-chain file. The transaction is timestamped and signed, becoming part of the Lab's activity log.

**Record.** Kamu stores the full file record — including the file DID, the Lab identifier, encryption metadata, access level, content hash, and version information — in its provenance database. The Molecule API stores a corresponding application record linking the file to the Lab's data room.

### Content Addressing

Every file stored through the pipeline receives a content identifier (CID) — a cryptographic hash derived from the file's contents using IPFS's multihash format. The CID serves as both the file's address and its integrity proof: requesting a CID from any IPFS node guarantees that the returned content is exactly what was originally stored. If even a single byte of the underlying file changes, the CID changes, and the on-chain reference in the Lab's TBA becomes a mismatch — making tampering immediately detectable.

Because CIDs are deterministic, the same file uploaded by different researchers at different times will always produce the same identifier. This property enables deduplication across the network and allows independent verification of data integrity without trusting any specific storage provider.

### Versioning and Provenance

Kamu maintains a complete, append-only history of every dataset in every Lab. When a file is uploaded, updated, or modified, Kamu creates a new version record that preserves the previous version's CID alongside the new one. No version is ever overwritten or deleted — the full lineage is permanently retrievable.

Each version record includes the content hash, the timestamp, the author's decentralised identifier (DID) linked to their wallet address, the data room path, and a reference to the previous version. This creates a verifiable provenance chain from the current state of any dataset back to its original upload. When a collaborator, funder, or reviewer needs to verify when data was created, who created it, or how it evolved over time, the evidence is in Kamu's version graph.

Kamu also records activity events — file access, metadata changes, announcements, and other Lab actions — providing a broader context for the dataset's history beyond just version changes.

### Permanent Persistence

IPFS provides content-addressed retrieval, but it does not guarantee permanent availability on its own. If every node that pins a file goes offline, the file becomes unreachable — the CID still exists as an address, but nothing answers the request.

Onchain Labs solve this by persisting files to Arweave in addition to IPFS. Arweave is a permanent, pay-once storage network — once a file is written, it remains available indefinitely regardless of whether any individual node or service continues operating. This dual-storage approach means files are retrievable from IPFS for fast, everyday access, and backed by Arweave for permanent, censorship-resistant availability.

Even if a file record is removed from a Lab's data room index, the underlying content persists on both IPFS (as long as it remains pinned) and Arweave (permanently). Because all files are encrypted before upload, this persistence does not compromise confidentiality — the content is publicly available but unreadable without the decryption keys controlled by Lit Protocol.

### E2E Upload Flow

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-07-071554.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-07-082524.png" alt=""><figcaption></figcaption></figure>

### Storage Summary

| Data Type                  | Where Stored                  | Managed By                           | Purpose                                       |
| -------------------------- | ----------------------------- | ------------------------------------ | --------------------------------------------- |
| Research files (encrypted) | IPFS + Arweave                | Filebase (upload), Kamu (versioning) | Decentralised, permanent, content-addressed   |
| On-chain data references   | Lab's Token Bound Account     | Smart contract                       | Tamper-evident CID pointers and metadata      |
| tokenURI pointer           | On-chain (Lab TBA)            | Lab smart contract                   | Permanent reference to IP-NFT metadata        |
| File versions              | Kamu provenance DB            | Kamu                                 | Append-only version history and audit trail   |
| Encryption keys            | Lit Protocol node network     | Lit Protocol                         | Threshold-sharded, condition-gated decryption |
| Access conditions          | On-chain + Lit Protocol       | Lab smart contract + Lit             | Defines who can decrypt each file             |
| File provenance            | Kamu provenance DB            | Kamu                                 | DID-based authorship, timestamps, lineage     |
| Activity events            | Kamu provenance DB + on-chain | Kamu + Lab TBA                       | Access logs, metadata changes, announcements  |
