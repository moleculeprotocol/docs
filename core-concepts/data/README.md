---
description: How Labs store, protect, and control access to scientific research data
icon: database
---

# Data

### **Data in Onchain Labs**

Every Onchain Lab is a data-centric primitive. When a researcher uploads a dataset, publishes results, or records experimental observations, that data becomes part of the Lab — held alongside its treasury and other assets. The Lab does not just _reference_ data; its onchain identity is cryptographically bound to its data room, so the link between Lab and scientific content is itself verifiable.

### Onchain and Offchain

Scientific research data — datasets, images, lab notebooks, analysis scripts — is too large and too sensitive to store directly on a blockchain. Onchain Labs solve this with a hybrid architecture that separates _data storage_ from _data control_.

The actual research files are stored offchain using a combination of decentralised storage networks that provide persistence, content-addressing, and provenance tracking. Every file version receives a content identifier (CID) — derived from the content itself, so tamper-evident — and is tracked in the data room's provenance log. What lives onchain is the **anchor**: the data room's decentralized identifier (DID) is cryptographically bound to the Lab's `oclId` in the DID registry, in a dual co-attested record anyone can verify. See [Data Anchoring (DID Linking)](../module-registry/data-module.md).

This design gives Labs the best of both worlds. The blockchain provides an immutable, publicly verifiable anchor for the Lab's data identity, while the data room's provenance log records what data exists, when it was uploaded, and by whom. The offchain layer provides the storage capacity, encryption, and performance that scientific data requires.

### The Data Stack

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-152541.png" alt=""><figcaption></figcaption></figure>

The data layer is built from a set of specialised technologies, each handling a different responsibility in the pipeline.

<table><thead><tr><th width="228.42578125">Layer</th><th width="133.84765625">Technology</th><th>Role</th></tr></thead><tbody><tr><td>Upload Gateway</td><td>Filebase</td><td>S3-compatible upload interface. Generates pre-signed URLs for secure browser uploads and automatically pins files to IPFS.</td></tr><tr><td>Decentralised Storage</td><td>IPFS</td><td>Content-addressed storage. Every file receives a unique CID derived from its contents, enabling verifiable retrieval from any IPFS node.</td></tr><tr><td>Permanent Persistence</td><td>Arweave</td><td>Immutable, permanent storage. Ensures research data remains available indefinitely, independent of any single service provider.</td></tr><tr><td>Provenance &#x26; Versioning</td><td>Kamu</td><td>Tracks the complete history of every dataset: versions, transformations, metadata changes, and activity events. Provides a verifiable provenance chain from raw data to published result.</td></tr><tr><td>Encryption &#x26; Access Control</td><td>Onchain-Verified Envelope Encryption + AccessResolver</td><td>Per-file AES-256 DEK wrapped by a protocol-operated key custodian (BLS threshold operator network on roadmap). Access conditions live on Kamu (ODF) and are re-verified against live onchain state (<code>AccessResolver</code>) before the plaintext DEK is released. Legacy files continue to resolve through Lit Protocol.</td></tr></tbody></table>

These components form a layered pipeline: data is encrypted client-side with a per-file wrapped DEK, uploaded through Filebase, pinned to IPFS, persisted on Arweave, and versioned through Kamu — with the data room's identity anchored onchain to the Lab via [DID linking](../module-registry/data-module.md).

### Data Lifecycle

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-06-152749.png" alt=""><figcaption></figcaption></figure>

A typical data flow through an Onchain Lab follows this sequence:

**Upload** — A researcher selects a file and chooses an access level. The backend issues a fresh wrapped data encryption key; the file is AES-256-GCM encrypted client-side before it leaves the browser. The wrapped DEK and the file's onchain access conditions are stored as encryption metadata on Kamu (ODF).

**Store** — The encrypted file is uploaded through Filebase, which pins it to IPFS (producing a CID) and persists it to Arweave for permanent availability. Kamu records the provenance metadata: who uploaded the file, when, the file's version, and its relationship to other datasets in the Lab.

**Anchor** — The file's CID and metadata are committed to the data room's provenance log, and the data room's DID is bound onchain to the Lab's `oclId` in the DID registry (this happens automatically at Lab creation, with dual co-attestation). Together these create a permanent, tamper-evident link between the Lab's onchain identity and its offchain data: the anchored DID identifies the data room, and the content-addressed log inside it proves what the data room contains.

**Access** — When someone requests a file, the backend re-verifies the stored access conditions against live chain state. If the requester meets the conditions (holds the right tokens, holds an active role grant on the Lab, holds a valid license), the protocol key custodian unwraps the DEK and returns it to the client. The file is retrieved from IPFS or Arweave and decrypted client-side. Legacy Lit-encrypted files continue to decrypt through the Lit SDK until migrated.

**Audit** — Every data-room action — uploads, version updates, metadata and access-level changes — is captured in the data room's tamper-evident provenance log, while onchain events (Lab creation, DID links, role grants and revocations) record the control-plane history. Together they form a verifiable chain of custody for the entire research lifecycle.

### Verifiable Research Record

Because the Lab's data room is anchored to its onchain identity and every version inside it is content-addressed, the combination of the Lab's transaction history and its provenance log forms a comprehensive, verifiable record of its scientific activity.

<table><thead><tr><th width="277.28515625">Event</th><th>What is recorded</th></tr></thead><tbody><tr><td>Dataset upload</td><td>Timestamp, uploader address, CID, file version</td></tr><tr><td>Data access</td><td>Requester address, file accessed, timestamp</td></tr><tr><td>Version update</td><td>Previous CID, new CID, change metadata</td></tr><tr><td>Access policy change</td><td>File, old conditions, new conditions, timestamp</td></tr></tbody></table>

This record exists independently of any institution, journal, or platform. It is cryptographic, publicly auditable, and permanently tied to the Lab's identity. When collaborators evaluate a project, when funders assess progress, or when reviewers verify provenance, the evidence is onchain.

### Design Principles

**Confidentiality by default.** Data is encrypted before it leaves the researcher's browser. The protocol assumes scientific data is sensitive unless explicitly made public.

**Per-file granularity.** Access conditions are configured at the individual file level, not at the Lab level. A single Lab can contain public datasets, token-gated research files, and time-locked results simultaneously.

**Onchain access control.** Who can access data is determined by smart contract conditions, not by a centralised permission list. The blockchain (via `AccessResolver`) is the access control layer; the backend only releases the unwrapped DEK after those conditions are satisfied.

**Permanent availability.** Research data is persisted to Arweave, ensuring it remains retrievable regardless of whether any individual service continues operating.

**Provenance from origin.** Every dataset is versioned and tracked from the moment of upload. Kamu records the full lineage: when data was created, how it was transformed, and what it produced.
