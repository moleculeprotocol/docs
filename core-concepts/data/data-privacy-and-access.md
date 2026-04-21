---
description: >-
  How Onchain Labs protect the confidentiality of sensitive research data
  through client-side encryption, decentralised key management, and on-chain
  access verification
icon: fingerprint
---

# Data Privacy & Access

### Why Privacy Matters

Scientific research data is often commercially sensitive, personally identifiable, or competitively valuable. Releasing raw experimental results, proprietary compounds, or patient-derived datasets without control can compromise patent applications, regulatory submissions, and competitive advantage. At the same time, the transparency benefits of on-chain science — provenance, reproducibility, collaboration — require that data _exists_ in a verifiable, shared infrastructure.

Onchain Labs resolve this tension by encrypting data before it enters the public infrastructure. The blockchain records _that_ data exists, _who_ uploaded it, and _who_ can access it — but never the data itself. The underlying content is encrypted client-side and stored in its encrypted form across IPFS and Arweave. Only parties who satisfy on-chain access conditions can reconstruct the decryption keys.

### Encryption Model

Every confidential file uploaded to an Onchain Lab is encrypted inside the researcher's browser before it leaves the device. The protocol uses Lit Protocol's encryption scheme, which generates a symmetric encryption key, encrypts the file content client-side, and then shards that key across Lit Protocol's decentralised node network using threshold cryptography. No single node — and no centralised server, including Molecule's — ever holds the complete key.

The encrypted blob is what gets uploaded, staged, pinned to IPFS, and archived to Arweave. At every point in the storage pipeline, the data is ciphertext. Even if the IPFS CID is publicly known and the content is publicly retrievable, it is unreadable without the decryption key — which can only be reconstructed when on-chain access conditions are satisfied.

Files marked as Public skip encryption entirely. The researcher explicitly chooses to make this data openly accessible. Public files still benefit from content addressing, versioning, and provenance tracking, but they carry no confidentiality guarantees by design.

### Access Conditions

Access level is assigned per-file at upload time by the Lab owner. Who can decrypt a file is determined by on-chain conditions, not by a centralised permission system. When a researcher uploads a confidential file, the Client SDK creates access conditions based on the Lab's IP-NFT contract. These conditions are evaluated by Lit Protocol's node network every time someone requests decryption.

Currently, access is verified through the IPNFT contract's `canRead()` function and the AccessResolver's `isAuthorizedSignerForIpnft()` function. This supports three access levels:&#x20;

* Public (no restriction),&#x20;
* Token-Holder (restricted to authorised Lab signers), and&#x20;
* Admin-Only (restricted to Lab administrators).

Future releases will introduce granular, composable access conditions: credential-gated access requiring a minimum holding of a project's IPTs, access-list gating by specific wallet addresses, payment-gated unlocks requiring a fee in a specified token, license-gated access via time-bound license NFTs (ERC-4907), and time-locked conditions that auto-release data at a specified date or block number. These conditions will be composable — a Lab could require both token ownership and fee payment before granting access.

### Decryption Flow

When an authorised user requests a confidential file, the following sequence occurs. The Client SDK retrieves the file's encryption metadata from Kamu. The encrypted blob is fetched from IPFS (or Arweave) by its CID. The SDK then requests decryption from Lit Protocol, passing the user's wallet signature and the file's access conditions.&#x20;

Each Lit node independently evaluates the on-chain conditions by querying the relevant smart contracts. If the threshold of nodes confirms the conditions are met, they release their key shards to the client. The symmetric key is reconstructed inside the user's browser, the file is decrypted client-side, and the plaintext is displayed. If the conditions are not met, the nodes refuse to release their shards, and decryption fails.

At no point does Molecule's backend, any single Lit node, or any intermediary have access to the decrypted content. The key only exists in its complete form inside the authorised user's browser, for the duration of the session.

### Access Grants

Lab administrators can extend temporary access to specific addresses without adding them as permanent authorised signers. By calling `grantReadAccess` on the IP-NFT contract with a recipient address and an expiration timestamp, administrators create a time-limited grant that is recorded on-chain. The grant automatically expires at the specified time. This enables controlled sharing with collaborators, reviewers, or funders without permanently modifying the Lab's access control list.

### Retrieval & Decryption Pipeline

<figure><img src="../../.gitbook/assets/Mermaid Chart - Create complex, visual diagrams with text.-2026-02-07-094253.png" alt=""><figcaption></figcaption></figure>

### Privacy Summary

The net result of this architecture is that no single party — not Molecule, not Lit Protocol, not any individual storage node — can access confidential research data. The file content is encrypted before it leaves the researcher's browser, transmitted as ciphertext over HTTPS, stored as ciphertext across IPFS and Arweave, and only ever decrypted inside an authorised user's browser after on-chain conditions are independently verified by a threshold of Lit nodes. Every action against the data — uploads, version changes, access events — is recorded with the author's decentralised identifier, creating a tamper-evident provenance trail that exists independently of any centralised service.

<table><thead><tr><th width="177.015625">Layer</th><th>Protection</th><th>Mechanism</th></tr></thead><tbody><tr><td>At rest</td><td>File content encrypted before leaving browser</td><td>Lit Protocol symmetric encryption, threshold key sharding</td></tr><tr><td>In transit</td><td>All communications over HTTPS; payload is ciphertext</td><td>TLS + pre-encryption</td></tr><tr><td>Key storage</td><td>No single party holds the complete decryption key</td><td>Threshold cryptography across Lit's decentralised node network</td></tr><tr><td>Access control</td><td>On-chain conditions determine who can decrypt</td><td>Smart contract evaluation by Lit nodes (IPNFT canRead, AccessResolver)</td></tr><tr><td>During decryption</td><td>Key reconstructed only inside the authorised user's browser</td><td>Client-side key assembly and decryption</td></tr><tr><td>Provenance</td><td>Every file action tracked with author's DID</td><td>Kamu version records with did:ethr:{wallet_address}</td></tr><tr><td>Permanence</td><td>Encrypted content persists even if file record is removed</td><td>IPFS + Arweave store ciphertext; keys are separate</td></tr></tbody></table>
