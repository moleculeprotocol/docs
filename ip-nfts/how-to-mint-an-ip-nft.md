# ✨ How to mint an IP-NFT?

{% hint style="info" %}
The public, production ready frontends to mint IPNFTs are:

Ethereum Mainnet: [https://mint.molecule.to/](https://mint.molecule.to/)

Testnet (Görli): [https://testnet.mint.molecule.to/](https://testnet.mint.molecule.to/)&#x20;
{% endhint %}



{% embed url="https://www.loom.com/share/eafffeb92b7d423daf43609d2b737be1" %}
Minting an IP-NFT on testnet.
{% endembed %}

* **Prepare your Research Agreement:** Two parties negotiate and sign a legal contract (Research Agreement). The **Research Agreement** states the rights of the asset holder (dataset, future IP, etc.) and the price paid for the asset. In our case, these agreements are often lined out as Sponsored Research Agreements (SRAs) but they could be any other kind of legal agreement.
* **Access the** [**Molecule IP-NFT Minting Front-End**](https://mint.molecule.to)**:** Visit the minting front-end and connect your wallet to get started with the process. Before starting, make sure you have ether on your wallet for gas fees, which vary based on traffic. If you are looking to mint an IP-NFT on testnet, get some testnet ETH from the [Goerli Faucet](https://goerlifaucet.com/). &#x20;
* **Connect your wallet:** The first thing you will need to do to mint an IP-NFT is connect a wallet. It is important that this wallet is an [externally-owned account](https://ethereum.org/en/developers/docs/accounts/) (EOA) and not a smart contract wallet, such as a Gnosis Multisig. This is to ensure your Research Agreement can be encrypted with a private key prior to pinning on decentralized storage.
* **Reserve an IP-NFT:** The process starts by reserving a TokenID on the IP-NFT contract. This TokenID will be used to let our access control infrastructure know that only the owner of the TokenID should have access to the data behind the NFT.&#x20;
* **Project Details:** Add information about your IP-NFT. All information in this step will be saved in the metadata of the IP-NFT and will be publicly available. Importantly, all information specified is immutable after an IP-NFT has been minted, so make sure that the information you provide is correct.
* **Research Agreement:** Upload the Research Agreement mentioned in Step 1, and add Contract Title, Signature Date, Contract Owner (Assignor), and Contract Type. It is important that these data elements match what is represented in the Research Agreement, and are referenced in the Assignment Agreement in the next step. The Research Agreement will be encrypted, so don't worry if it has confidential information.&#x20;
* **Assignment Agreement:** This agreement assigns the rights of the Research Agreement (Step 6) to the holder of the IP-NFT. Sign your legal name, specify the organization you are assigning the rights to (DAO name), and specify the target wallet address (default is the connected wallet). If you are minting the IP-NFT to a different wallet, make sure to change that wallet address here. **PLEASE DOUBLE CHECK THE WALLET ADDRESS**. After minting, this target address will be the IP-NFT holder, and is the legal assignee of the Research Agreement. The Assignment Agreement will be publicly viewable to ensure rights are properly assigned.&#x20;
* **Artwork:** Upload an image to commemorate the research. This image will be public, so please follow the content guidelines and upload appropriate images.
* **Review your IP-NFT data:** Review the data submitted for the IP-NFT. Once ready, continue the process. After submitting your data in this step, the Research Agreement will be encrypted and uploaded to the decentralized data storage. The decryption key will be stored with Lit Protocol to manage access control based on the NFT ownership. You will be able to download this key at any time to access your files as long as you control the IP-NFT.
* **Sign IP-NFT minting transaction:** Your wallet will pop-up and ask you to sign your agreement to the minting terms and to sign the minting transaction that creates the IP-NFT. A symbolic minting fee (0.001 ETH / \~2 USD) required to counter spam attacks on the protocol.
* **View your IP-NFT:** Once the transaction is finalized on chain you will find an IP-NFT in your wallet. Congrats! You just created an IP-NFT and brought legal rights on-chain!
