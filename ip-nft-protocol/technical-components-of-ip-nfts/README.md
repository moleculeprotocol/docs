# 🛠 Technical Components of IP-NFTs

Molecule uses a host of services to enable the transacting of intellectual property in the form of NFTs. Below you can find a list of third-party projects and their corresponding application in the IP-NFT tech stack:

<figure><img src="https://lh3.googleusercontent.com/2H3rZNjLyZUfCDBOvgNblFIJB8CmEIrqRFDIjE87HGScZcEBZG7z_S-EpUmJAnj1auYpzF0vDZvJKDsp0jKF4xhRmy6WU4tUN6HjLjUWmDg-yA5R4_GJcX_Hrf_AugPgC1AmFwXllAKT5L_DFfbfPSqePLocO077JFD26ETvXoHn2okAYHngnVp9kIjIdg" alt=""><figcaption><p>Diagram showing the process of IP-NFT minting</p></figcaption></figure>

### [**Lit Protocol**](https://litprotocol.com/) **(Access Control Module)**

IP-NFTs can be used to securely manage access rights to private research data, legal agreements and IP. Molecule uses Lit Protocol to create and persist encryption keys for various use-cases. Lit Protocol nodes use secure threshold computation to store secrets so that no party besides the allowed ones ever has access to a private key. IP-NFT holders that comply to a token's access control conditions are able to retrieve the decrypted encryption key for all documents or data attached to an IP-NFT.

{% embed url="https://developer.litprotocol.com/" %}
Lit Protocol Documentation
{% endembed %}

### [**IPFS**](https://ipfs.tech/) **(Decentralized Storage & Distribution)**

IPFS is the most well known method to address and distribute data on peer to peer networks. Since the network cannot natively guarantee persistence, we're utilising Protocol Labs' [web3.storage](https://web3.storage/) platform to pin all IPFS related content (e.g. encrypted legal documents) and automatically have Filecoin deals created.&#x20;

{% embed url="https://web3.storage/faq/" %}
web3.storage
{% endembed %}

### [**Arweave**](https://www.arweave.org/) **(Data Storage Module)**

IP-NFTs generally are agnostic about the storage layer of their metadata and attachments. Arweave is a decentralised, permanent data storage network with  different persistence guarantees and economical mechanics than Filecoin.  We consider to additionally offer support for Arweave persistence under full custody of IP-NFT minters. Users of the Molecule IP-NFT protocol are already free to use it.

{% embed url="https://docs.arweave.org/" %}
Arweave Documentation
{% endembed %}

### [**Filecoin**](https://filecoin.io/) **(Persistence / Storage Module)**

To keep public and private research data persistent, Molecule implicitly makes use of Filecoin when uploading  data to web3.storage. Besides attached legal documents and IP-NFT metadata containing research project detail information, Molecule aims to also backup private research data on this distributed data storage blockchain.

{% embed url="https://docs.filecoin.io/" %}
Filecoin Documentation
{% endembed %}
