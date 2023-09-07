---
description: The process of minting IP Tokens from an IP-NFT.
---

# ✨ How to synthesize IPTs from an IP-NFT?

{% embed url="https://www.loom.com/share/f35771c266dd4159832853c5e93cc117" %}
Synthesizing IPTs from an IP-NFT on testnet.
{% endembed %}

The `Synthesizer` is a smart contract enabling IP-NFT owners to create a tailored IPTs ERC20 contract. Think of it as a cryptographic key unlocking fractional rights associated with an IP-NFT - a significant stride towards a future of decentralized IP management.

The `Synthesizer` is a smart contract that acts as a catalyst, synthesizing IPTs from an IP-NFT. IP-NFTs are processed through the `Synthesizer` to produce IPTs, marking a transformative step towards a decentralized future of IP management. The activation energy to conduct this synthesis is the cost of gas on Ethereum. Beware, as this Synthesis reaction is highly [thermodynamically favorable](https://tenor.com/view/robert-downey-jr-tony-stark-iron-man-behold-explosion-gif-9319158) (ΔG° < 0).&#x20;

IPTs have various functional groups; they can effectuate distributed governance, access to privileged and proprietary data, and IP licensing. Though these reactive capacities aren't necessarily inherent to all IP-NFT contracts, they can be configured via IPT-weighted governance. Much like how researchers experiment with many molecules to discover cures for a particular disease, IPTs are meant to be experimented with to discover the cure for apathy and scientific stagnation.

By executing the `synthesizeIpnft(uint256 ipnftId, uint256 moleculesAmount, string calldata agreementCid)` function, an IP-NFT owner initializes the creation of a new IPT supply. This is akin to sparking a digital forge, producing a minimal clone of the current IPT implementation and appointing the Synthesizer contract as its owner. It's the conductor leading the symphony of code.

The original IP-NFT owner has the authority to instruct the `Synthesizer` to mint an arbitrary quantity of IPTs via the issue function. However, token holders need to be aware of possible dilution at the discretion of the IP-NFT holder - it's a little like adding water to a concentrated solution. New emissions are managed by the governance layer overseeing the multisignature wallet retaining the IP-NFT.

An original owner can designate an IPTs contract as capped, effectively setting a supply limit. This condition is often a requirement for a SalesShareDistributor of that token.

While the initial distribution of IPTs to other accounts falls outside the `Synthesizer`'s remit, IPTs holders retain the freedom to transfer their tokens at will.

For certain privileges, IPT holders might need to agree to a legal document provided during the synthesis transaction. Stored as IPFS CIDs within their Metadata structure, these agreements, while not intrinsic to IPTs, act as legal guardrails enforced by other system components, such as the `TermsAcceptedPermissioner`.

And there you have it, a sophisticated fusion of technology and law, setting the stage for a more democratic and accessible future in IP management. A true coding masterpiece in the symphony of blockchain-based IP rights.
