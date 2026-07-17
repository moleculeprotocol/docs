---
description: >-
  How a Lab's offchain data identity is anchored onchain — every Lab's data
  room DID is bound to its OCL-ID through a dual co-attested registry entry
  that anyone can verify.
icon: binary-circle-check
---

# Data Anchoring (DID Linking)

## Data Anchoring (DID Linking)

Every Lab's research data lives offchain — in its data room, versioned and provenance-tracked by Kamu (ODF). DID linking is the mechanism that anchors that offchain identity onchain: the data room (and the Lab's data-platform account) each carry a [W3C DID](https://www.w3.org/TR/did-core/) (`did:odf:…`), and the `MoleculeOclDidRegistry` contract binds those DIDs to the Lab's canonical `oclId` in a record that anyone can read and verify with nothing but an RPC connection.

This is a deliberately novel piece of infrastructure. Rather than storing data references in a database and asking the world to trust it, Molecule turns the Lab ↔ data binding itself into a first-class onchain object — cryptographically co-attested by two independent parties, versioned, and permanently auditable.

### The Trust Problem

Without an onchain anchor, the mapping between a Lab and its research data exists only inside an offchain service. The Lab's account would hold assets, manage its treasury, and control IP — but its link to the actual scientific work would rest on trusting a database. Anyone claiming "this dataset belongs to that Lab" would have nothing verifiable to point at.

DID linking solves this. Once linked, the association is public onchain state: a third party — an investor doing due diligence, a protocol composing on top of Labs, an indexer building analytics — can resolve a Lab's data identity directly from the chain and check exactly who vouched for the binding.

### How a Link Is Created

Linking happens automatically when a Lab is created:

```
1. createLab → Kamu provisions the Lab's account + data room,
               each identified by a DID (did:odf:ed25519:…)
2. The backend queues the Lab for linking
3. The linking worker builds a signed, deadline-bound LinkDidRequest per DID
4. Kamu signs the request with the dataset's own ed25519 key (the proof)
   and attests to that proof with its registered attester key
5. Molecule co-signs an EIP-712 attestation binding the request to the proof
6. The relayer submits linkDidBatch — both DIDs are linked in one
   atomic transaction
7. The registry emits DidLinked events; the indexer confirms the link
```

Progress is queryable via the Labs API: `getDidLinkStatus(oclId)` returns the linked DIDs and a status of `PENDING`, `SUBMITTED`, `LINKED`, or `FAILED`. The terminal `LINKED` state is driven by the onchain events themselves — the backend doesn't declare success; the chain does.

### Dual Co-Attestation

A link is only accepted if **two independent parties** sign off on it, and the registry verifies both signatures onchain before writing anything:

* **Kamu (the data platform)** attests that the dataset's own ed25519 key genuinely signed the link request — i.e. the data source itself consented to the binding. The raw ed25519 proof is recorded onchain alongside the link.
* **Molecule** attests, via an EIP-712 signature, that it authorized this specific `oclId → DID` binding for this exact request.

Attesters are resolved by recovered signature address — not by array position — and must be distinct; a missing, duplicated, or mismatched attester reverts the transaction. The relayer that submits the transaction is *untrusted for authenticity*: it holds the only role allowed to call `linkDid`, but it cannot forge a link, because it controls neither attester key. Every request is additionally replay-protected (single-use request IDs per Lab/provider/subject), deadline-bound, and EIP-712 domain-bound to the specific registry contract and chain.

Successful links carry the `CO_ATTESTED` verification tier. A stricter `ONCHAIN_VERIFIED` tier — where the data source's proof is verified directly onchain rather than attested to — is defined in the contract and reserved for a future verifier.

### What Anyone Can Verify

From public chain state alone:

* **A Lab's live data identity** — `getActiveDid(oclId, provider, subject)` returns the currently-linked DID string for the Lab's account and data room.
* **Who vouched for it** — every `DidLinked` event carries the full DID, the ed25519 proof, both attestation signatures, the request hash, and the verification tier. Anyone can re-recover both attesters and independently re-verify the dataset's proof offchain.
* **The full lineage** — links are never deleted. Re-linking a new DID deactivates the previous record (`DidDeactivated`) and increments a per-binding version counter, so the complete rotation history stays auditable forever.
* **That the Lab is really the Lab** — the `oclId` deterministically packs the LabNFT tokenId and the Lab's ERC-6551 account address, and the registry validates that packing against the protocol's canonical derivation config before accepting a link.

### Beyond Kamu: Anchoring Any Data Source

The registry itself is **data-source-agnostic** — this is what makes the design powerful. It stores DIDs as opaque strings (no DID method is parsed onchain), namespaces every binding by an arbitrary `(provider, subject)` pair, and delegates verification to a pluggable verifier contract registered per namespace. The ODF/Kamu co-attestation scheme is simply the first verifier.

That means any DID-identified data source — another data platform, an instrument feed, an institutional archive — could in principle be anchored to a Lab by deploying a new verifier and registering it, with **no change to the registry**. The Lab becomes a universal, verifiable onchain index of its offchain data identities, whatever systems those identities live in. Today, `did:odf` (Kamu) is the wired provider; the verifier interface explicitly anticipates additional policies over time.

### Contract Reference

| Item | Value |
| --- | --- |
| Registry (Base mainnet, proxy) | [`0x6cd3Cf3c34a18Bf48F90590c3a57708F175b2eE3`](https://basescan.org/address/0x6cd3Cf3c34a18Bf48F90590c3a57708F175b2eE3) |
| Verifier — OdfCoAttestVerifier (Base mainnet) | [`0x5a7a22bbad7B8B3c91EdB7FC2Af2DE5de9060D8B`](https://basescan.org/address/0x5a7a22bbad7B8B3c91EdB7FC2Af2DE5de9060D8B) |
| Registry (Base Sepolia, proxy) | [`0x8A23622967Cf7e3BB3219217ff685a5E4830C617`](https://sepolia.basescan.org/address/0x8A23622967Cf7e3BB3219217ff685a5E4830C617) |
| Verifier — OdfCoAttestVerifier (Base Sepolia) | [`0x8c1BD0120CDe6102E60F547F5c898b82F6075541`](https://sepolia.basescan.org/address/0x8c1BD0120CDe6102E60F547F5c898b82F6075541) |

Key functions:

```solidity
// Write — relayer-only (RELAYER_ROLE), pausable
function linkDid(LinkDidRequest calldata req, bytes calldata proof, LinkDidAttestation[] calldata attestations) external;
function linkDidBatch(LinkDidRequest[] calldata reqs, bytes[] calldata proofs, LinkDidAttestation[][] calldata attestations) external;

// Read
function getActiveDid(bytes32 oclId, bytes32 provider, bytes32 subject) external view returns (string memory);
```

Events: `DidLinked`, `DidDeactivated`, `VerifierSet`, `RelayerUpdated`, `DerivationConfigSet`. The contract is UUPS-upgradeable with an unusual safety guard: an upgrade that would change the EIP-712 signing domain reverts, so offchain signers can never be silently invalidated. Administration (verifier registration, pausing, attester rotation) sits with Molecule's governance multisigs; the relayer role is held by a dedicated ERC-4337 smart account.

### Current Status

DID linking is **live on Base mainnet** — Labs created through the app are linked automatically, and co-attested `DidLinked` records are accumulating on the registry since the v0.1.0 go-live. On the roadmap: the `ONCHAIN_VERIFIED` tier (direct onchain proof verification), additional verifier policies (attested, community, stake- or ZK-backed), and additional providers and subjects beyond the initial ODF account + data-room pair.

For the offchain half of the story — how files are stored, versioned, and encrypted inside the data room this anchor points at — see [Data Storage](../data/data-storage.md) and [Data Privacy & Access](../data/data-privacy-and-access.md).
