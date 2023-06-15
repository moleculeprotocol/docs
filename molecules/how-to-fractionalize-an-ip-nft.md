# ✨ How to synthesize Molecules from an IP-NFT?

{% embed url="https://www.loom.com/share/f35771c266dd4159832853c5e93cc117" %}
Synthesizing Molecules from an IP-NFT on testnet.
{% endembed %}

The `Synthesizer` is a smart contract enabling IP-NFT owners to create a tailored Molecules ERC20 contract. Think of it as a cryptographic key unlocking fractional rights associated with an IP-NFT - a significant stride towards a future of decentralized IP management.

But Molecules are more than mere ERC20 tokens. They can potentially facilitate distributed governance, access to privileged documents, proprietary data, and licensing. These capabilities aren't inherent in all IP-NFT contracts, but can be configured by third-party developers leveraging Molecules ownership. A bit like designing your own therapeutic, you design the optimal molecule for a particular target.

By executing the `synthesizeIpnft(uint256 ipnftId, uint256 moleculesAmount, string calldata agreementCid)` function, an IP-NFT owner initializes the creation of a new Molecules supply. This is akin to sparking a digital forge, producing a minimal clone of the current Molecules implementation and appointing the Synthesizer contract as its owner. It's the conductor leading the symphony of code.

The original IP-NFT owner has the authority to instruct the Molecules to mint an arbitrary quantity of molecules via the issue function. However, token holders need to be aware of possible dilution at the discretion of the IP-NFT holder - it's a little like adding water to a concentrated solution. New emissions are managed by the governance layer overseeing the multisignature wallet retaining the IP-NFT.

An original owner can designate a Molecules contract as capped, effectively setting a supply limit. This condition is often a requirement for a SalesShareDistributor of that token.

While the initial distribution of Molecules to other accounts falls outside the Synthesizer's remit, Molecule holders retain the freedom to transfer their tokens at will.

For certain privileges, Molecule holders might need to agree to a legal document provided during the synthesis transaction. Stored as IPFS CIDs within their Metadata structure, these agreements, while not intrinsic to the Molecules, act as legal guardrails enforced by other system components, such as the `TermsAcceptedPermissioner`.

And there you have it, a sophisticated fusion of technology and law, setting the stage for a more democratic and accessible future in IP management. A true coding masterpiece in the symphony of blockchain-based IP rights.
