---
description: >-
  How Onchain Labs store, protect, and control access to scientific research
  data
icon: database
---

# Data

### **Data in Onchain Labs**

Every Onchain Lab is a data-centric primitive. When a researcher uploads a dataset, publishes results, or records experimental observations, that data becomes part of the Lab — owned by the Lab's Token Bound Account alongside its IP-NFTs, treasury, and other assets. The Lab does not just _reference_ data; it _owns_ the references that cryptographically link its onchain identity to the underlying scientific content.

This means data follows the same ownership model as every other Lab asset. Transferring the LabNFT transfers control of the data. Funding a Lab's treasury funds the project that produced the data. Licensing an IP-NFT can gate access to the data behind it. Data is not a sidecar to the protocol — it is the scientific substance that gives every other asset in the Lab its meaning.

### On-chain and Off-chain

Scientific research data — datasets, images, lab notebooks, analysis scripts — is too large and too sensitive to store directly on a blockchain. Onchain Labs solve this with a hybrid architecture that separates _data storage_ from _data control_.

The actual research files are stored off-chain using a combination of decentralised storage networks that provide persistence, content-addressing, and provenance tracking. What lives on-chain, inside the Lab's Token Bound Account, are the data references: content identifiers (CIDs) that point to the off-chain files, along with the metadata that describes them. Because CIDs are derived from the content itself, they are tamper-evident — if the underlying file changes, the CID changes, and the onchain reference becomes invalid.

This design gives Labs the best of both worlds. The blockchain provides an immutable, auditable record of what data exists, when it was uploaded, who uploaded it, and who has accessed it. The off-chain layer provides the storage capacity, encryption, and performance that scientific data requires.

### The Data Stack

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-152541.png" alt=""><figcaption></figcaption></figure>

The data layer is built from a set of specialised technologies, each handling a different responsibility in the pipeline.

<table><thead><tr><th width="228.42578125">Layer</th><th width="133.84765625">Technology</th><th>Role</th></tr></thead><tbody><tr><td>Upload Gateway</td><td>Filebase</td><td>S3-compatible upload interface. Generates pre-signed URLs for secure browser uploads and automatically pins files to IPFS.</td></tr><tr><td>Decentralised Storage</td><td>IPFS</td><td>Content-addressed storage. Every file receives a unique CID derived from its contents, enabling verifiable retrieval from any IPFS node.</td></tr><tr><td>Permanent Persistence</td><td>Arweave</td><td>Immutable, permanent storage. Ensures research data remains available indefinitely, independent of any single service provider.</td></tr><tr><td>Provenance &#x26; Versioning</td><td>Kamu</td><td>Tracks the complete history of every dataset: versions, transformations, metadata changes, and activity events. Provides a verifiable provenance chain from raw data to published result.</td></tr><tr><td>Encryption &#x26; Access Control</td><td>Onchain-Verified Envelope Encryption + AccessResolver</td><td>Per-file AES-256 DEK wrapped by a protocol-operated key custodian (BLS threshold operator network on roadmap). Access conditions live on Kamu (ODF) and are re-verified against live on-chain state (<code>AccessResolver</code>) before the plaintext DEK is released. Legacy files continue to resolve through Lit Protocol.</td></tr></tbody></table>

These components form a layered pipeline: data is encrypted client-side with a per-file wrapped DEK, uploaded through Filebase, pinned to IPFS, persisted on Arweave, versioned through Kamu, and referenced on-chain in the Lab's Token Bound Account.

### Data Lifecycle

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-152749.png" alt=""><figcaption></figcaption></figure>

A typical data flow through an Onchain Lab follows this sequence:

**Upload** — A researcher selects a file and chooses an access level. The backend issues a fresh wrapped data encryption key; the file is AES-256-GCM encrypted client-side before it leaves the browser. The wrapped DEK and the file's on-chain access conditions are stored as encryption metadata on Kamu (ODF).

**Store** — The encrypted file is uploaded through Filebase, which pins it to IPFS (producing a CID) and persists it to Arweave for permanent availability. Kamu records the provenance metadata: who uploaded the file, when, the file's version, and its relationship to other datasets in the Lab.

**Reference** — The CID and associated metadata are recorded on-chain in the Lab's Token Bound Account. This creates a permanent, tamper-evident link between the Lab's onchain identity and the off-chain data. The transaction itself is timestamped and signed, contributing to the Lab's audit trail.

**Access** — When someone requests a file, the backend re-verifies the stored access conditions against live chain state. If the requester meets the conditions (holds the right tokens, holds an active role grant on the Lab, holds a valid license), the protocol key custodian unwraps the DEK and returns it to the client. The file is retrieved from IPFS or Arweave and decrypted client-side. Legacy Lit-encrypted files continue to decrypt through the Lit SDK until migrated.

**Audit** — Every access event is logged on-chain through the Lab's Token Bound Account. This creates a complete, publicly auditable record of who accessed what data and when — a verifiable chain of custody for the entire research lifecycle.

### Verifiable Research Record

Because every data action flows through the Lab's Token Bound Account, the Lab's transaction history forms a comprehensive, immutable record of its scientific activity.

<table><thead><tr><th width="277.28515625">Event</th><th>What is recorded</th></tr></thead><tbody><tr><td>Dataset upload</td><td>Timestamp, uploader address, CID, file version</td></tr><tr><td>Data access</td><td>Requester address, file accessed, timestamp</td></tr><tr><td>Version update</td><td>Previous CID, new CID, change metadata</td></tr><tr><td>Access policy change</td><td>File, old conditions, new conditions, timestamp</td></tr></tbody></table>

This record exists independently of any institution, journal, or platform. It is cryptographic, publicly auditable, and permanently tied to the Lab's identity. When collaborators evaluate a project, when funders assess progress, or when reviewers verify provenance, the evidence is on-chain.

### Design Principles

**Confidentiality by default.** Data is encrypted before it leaves the researcher's browser. The protocol assumes scientific data is sensitive unless explicitly made public.

**Per-file granularity.** Access conditions are configured at the individual file level, not at the Lab level. A single Lab can contain public datasets, token-gated research files, and time-locked results simultaneously.

**Onchain access control.** Who can access data is determined by smart contract conditions, not by a centralised permission list. The blockchain (via `AccessResolver`) is the access control layer; the backend only releases the unwrapped DEK after those conditions are satisfied.

**Permanent availability.** Research data is persisted to Arweave, ensuring it remains retrievable regardless of whether any individual service continues operating.

**Provenance from origin.** Every dataset is versioned and tracked from the moment of upload. Kamu records the full lineage: when data was created, how it was transformed, and what it produced.
