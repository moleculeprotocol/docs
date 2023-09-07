---
description: The process of minting IP Tokens from an IP-NFT.
---

# ✨ How to tokenize IPTs from an IP-NFT?

{% embed url="https://www.loom.com/share/f35771c266dd4159832853c5e93cc117" %}
Creating IPTs from an IP-NFT on testnet.
{% endembed %}

The `Tokenizer` is a smart contract enabling IP-NFT owners to create a tailored IPT ERC20 contract. Think of it as a cryptographic key that unlocks fractional rights associated with an IP-NFT - a significant stride towards a future of decentralized IP management.&#x20;

The `Tokenizer` is a smart contract that acts as a catalyst, synthesizing IPTs from an IP-NFT. IP-NFTs are processed through the `Tokenizer` to produce IPTs, marking a transformative step towards a decentralized future of IP management. The activation energy to conduct this synthesis is the cost of gas on Ethereum. Beware, as this Synthesis reaction is highly [thermodynamically favorable](https://tenor.com/view/robert-downey-jr-tony-stark-iron-man-behold-explosion-gif-9319158) (ΔG° < 0).&#x20;

IPTs have various functional groups; they can effectuate distributed governance, grant access to privileged and proprietary data, and IP licensing. Though these reactive capacities aren't necessarily inherent to all IP-NFT contracts, they can be configured via IPT-weighted governance. Much like how researchers experiment with many molecules to discover cures for a particular disease, IPTs are meant to be experimented with to discover the cure for apathy and scientific stagnation.

By executing the `Tokenizer:tokenizeIpnft` function, an IP-NFT owner initializes the creation of a new `IPToken` ERC20 contract with an initial supply. This is akin to sparking a digital forge, producing a minimal clone of the current `IPToken` implementation and appointing the Tokenizer contract as its owner. It's the conductor leading the symphony of code.

An `IPToken` contract's owner has the authority to mint an arbitrary supply of IPTs via its `issue` function. However, token holders need to be aware of possible dilution at the discretion of the IP-NFT holder - it's a little like adding water to a concentrated solution. New emissions are managed by the governance layer overseeing the multisignature wallet retaining the IP-NFT.

An original owner can designate a IPTs contract as capped, effectively disabling an further increase of supply. This condition can be queried by other clients, like a potential sales share distribution contract for that token.

While the initial distribution of IPTs to other accounts falls outside the `Tokenizer`'s remit, IPT holders retain the freedom to transfer their tokens at will and without any charge.

For certain privileges, IPT holders might need to agree to a legal document provided during the tokenization transaction. Stored as IPFS CIDs within their `Metadata` structure, these agreements, while not intrinsic to IPTs, act as legal guardrails enforced by other system components, such as the `TermsAcceptedPermissioner`.

And there you have it, a sophisticated fusion of technology and law, setting the stage for a more democratic and accessible future in IP management.&#x20;

{% hint style="info" %}
The Tokenizer contract is deployed on

mainnet: `0x58EB89C69CB389DBef0c130C6296ee271b82f436`

görli: `0xb12494eeA6B992d0A1Db3C5423BE7a2d2337F58c`
{% endhint %}
