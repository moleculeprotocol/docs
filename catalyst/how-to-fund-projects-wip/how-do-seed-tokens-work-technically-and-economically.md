---
description: A deep-dive into the mechanics of Seed tokens
---

# How do Seed tokens work technically & economically?

### Token Specification

Seed tokens are built using the ERC-1155 standard on Ethereum, which facilitates managing both fungible and non-fungible tokens under a single contract. This approach offers distinct advantages over the ERC-20 standard, especially in terms of operational efficiency and flexibility:

1. **Single Contract for Multiple Tokens**: Unlike ERC-20, which requires a separate contract for each token instance, ERC-1155 allows for multiple token instances to be managed within a single contract. This simplifies the deployment and management of many Seed token markets existing in parallel.
2. **Upgradeability**: A key advantage of ERC-1155 is that upgrades to the token contract can be applied across all token instances simultaneously. This feature is particularly valuable for Catalyst, as it enables streamlined updates without the need to individually upgrade each token type or instance.
3. **Batch Transactions**: ERC-1155 supports transactions involving multiple tokens at once, reducing the number of transactions required and thereby lowering transaction costs and complexity.
4. **Reduced Gas Costs**: By enabling batched transactions, ERC-1155 can significantly reduce the gas fees typically associated with separate transaction processes for each token action in ERC-20.

### **Token Economics**

**Supply Limitations**&#x20;

Each project on Catalyst is allocated a fixed maximum supply of 10,000 Seed tokens, as defined in the smart contracts. This consistent token supply across projects enables clear and standardized comparisons of funding and valuations, facilitating transparent and comparable assessments for funders.

**Token Distribution**

* **Pre-Minting to the Sourcer**: At the start of each project, .1% of the total token supply, or 100 tokens, is pre-minted and granted to the project creator, known as the Sourcer. This initial allocation is strategic:
  * **Collaborative Engagement**: The pre-minted tokens are primarily used by the Sourcer to engage and incentivize collaborators who can contribute to the project. This might include experts who can enhance the project proposal, community influencers who can help amplify the project’s visibility, or potential backers who might contribute substantial funding.
  * **Incentive for Contribution**: By distributing these tokens, Sourcers can directly reward contributors for adding value to the project, whether by enhancing the proposal's quality, providing critical reviews, or assisting in outreach efforts to attract more funding.
* **Purchasing Mechanism**: Following the pre-mint phase, the remaining tokens are made available for purchase through a bonding curve. This pricing mechanism adjusts the cost of tokens based on their buying and selling activity, ensuring a transparent and fair distribution process for all potential investors.

**Implications for Project Funding and Sourcer Incentives**

* **Project Viability**: The distribution of pre-minted tokens helps to establish a strong initial network around the project, boosting its likelihood of success.
* **Aligned Interests**: Sourcers and contributors are incentivized to actively support the project as their token value is tied to its success.
* **Collaborator Engagement**: This model promotes community involvement, with tokenholders becoming key stakeholders in the project’s outcome.

### **Bonding Curve Dynamics**

**Detailed Pricing Mechanism**&#x20;

Catalyst employs a linear bonding curve to determine the price of Seed tokens as their supply increases. The curve starts at a base price corresponding to the pre-minted amount for the Sourcer and ascends linearly, indicating a direct proportionality between the number of tokens sold and the price per token. The price function `P(s) = m * (s - p)` calculates the cost for any given token 's', where 'm' represents the slope of the curve, 's' the current supply, and 'p' the pre-minted amount. Each new purchase or sale moves the price point along this curve.

Contrary to traditional bonding curves, Catalyst imposes a unique model for token resale. Should a user decide to sell their Seed tokens, they can only withdraw the initial amount of their contribution, rather than selling at the prevailing market price. This mechanism is designed to ensure alignment between funders' interests and the project's success, encouraging funding rather than trading.

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption><p>Illustrative Bonding Curve: This graph represents the price-supply relationship of Seed tokens on Catalyst. The curve starts with a pre-minted amount 'p' at the initial price point. It then linearly increases, reflecting the price 'P(s)' for each subsequent Seed token 's'. The shaded areas represent the funding stages: 'Under' indicates the amount below the funding goal, 'Over' indicates funding beyond the goal, and 'F' marks the funding goal in ETH. The slope 'm' indicates the rate at which the price increases with the supply.</p></figcaption></figure>

**Economic Implications** The linear nature of the bonding curve has several economic implications for project valuation and funding stability:

* **Predictable Valuation**: As the price increases in a known linear fashion, funders can easily calculate the current and future value of tokens. This predictability attracts confident investment as funders can anticipate growth.
* **Funding Stability**: The incremental price increase encourages early funding, as early participants can acquire tokens at a lower price. This helps projects to gain initial capital quickly, contributing to funding momentum and stability.
* **Market Sentiment Reflection**: The linear curve reflects real-time market sentiment and commitment towards the project. As more participants buy in, the rising curve demonstrates increasing collective valuation of the project.
* **Long-Term Incentivization**: The inability to sell tokens above the purchase price, as per Catalyst's unique model, discourages short-term speculation and promotes long-term investment in the project’s success.



1. **Usage and Limitations**:
   * **Roles of Tokens**: Explain the different uses of Seed tokens in the platform, such as governance, funding, and potentially rewards if applicable.
   * **Transfer Restrictions**: Explicitly state the non-transferability of tokens during certain phases, which is crucial to ensure users understand their liquidity conditions.
2. **Exit and Refund Mechanisms**:
   * **Selling Back Tokens**: Go into detail about the conditions under which tokens can be sold back to the platform and the financial return users can expect, emphasizing the designed protection against speculative behavior.
   * **Refund Processes**: Clarify the process for obtaining refunds, particularly in scenarios where a project expires or fails to meet its funding goal.
3. **Integration with Broader Ecosystem**:
   * **Interactions with Other Platform Components**: Describe how Seed tokens interact with other elements of the Catalyst ecosystem, such as the project creation phase, funding mechanisms, and potential integration with other decentralized finance (DeFi) protocols.
   * **Legal and Regulatory Considerations**: Briefly touch on any legal or regulatory considerations relevant to the issuance and management of these tokens, particularly if they might affect token holders.
4. **FAQs and Troubleshooting**:
   * **Common Issues and Questions**: Provide a brief FAQ section addressing common questions and potential issues users might face with Seed tokens.
5. **Glossary of Terms**: Include a glossary for key terms used in the documentation to help both crypto/web3-savvy users and newcomers understand the specialized vocabulary.

\
