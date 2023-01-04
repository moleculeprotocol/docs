# 🛠 Technical Components of IP-NFTs

Molecule uses a host of services to enable the transacting of intellectual property in the form of NFTs. Find below a list of third-party projects and their corresponding application in the IP-NFT tech stack:

<figure><img src="https://lh3.googleusercontent.com/2H3rZNjLyZUfCDBOvgNblFIJB8CmEIrqRFDIjE87HGScZcEBZG7z_S-EpUmJAnj1auYpzF0vDZvJKDsp0jKF4xhRmy6WU4tUN6HjLjUWmDg-yA5R4_GJcX_Hrf_AugPgC1AmFwXllAKT5L_DFfbfPSqePLocO077JFD26ETvXoHn2okAYHngnVp9kIjIdg" alt=""><figcaption><p>Diagram showing the process of IP-NFT minting</p></figcaption></figure>

### ****[**Lit Protocol**](https://litprotocol.com/) **(Access Control Module)**

To securely manage access rights to private research data and IP, Molecule uses Lit Protocol's SDK. Lit is a leading technology provider for creating and managing access to encryption keys for different use-cases. In our case, Lit Protocol hosts the access control condition to an encrypted descyption key for the documents or data connected to an IP-NFT

{% embed url="https://developer.litprotocol.com/" %}
Lit Protocol Documentation
{% endembed %}

### ****[**Arweave**](https://www.arweave.org/) **(Data Storage Module)**

Similar to Filecoin, Arweave is a decentralised, permanent data storage network employed by Molecule. Molecule currently hosts the IP-NFT metadata on Arweave

{% embed url="https://docs.arweave.org/" %}
Arweave Documentation
{% endembed %}

### ****[**Filecoin**](https://filecoin.io/) **(Data Storage Module)**

To enable decentralised data storage of public and private research data, Molecule is in the process of implementing Filecoin. Starting with public research descriptions and data points, Molecule aims to also host private research data on the distributed data storage system

{% embed url="https://docs.filecoin.io/" %}
Filecoin Documentation
{% endembed %}
