---
description: A deep-dive into the mechanics of Seed tokens
---

# How do Seed tokens work technically & economically? \[WIP]

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

At the start of each project, .1% of the total token supply, or 100 tokens, is pre-minted and granted to the project creator, known as the Sourcer. This initial allocation is strategic.

The pre-minted tokens are primarily used by the Sourcer to engage and incentivize collaborators who can contribute to the project. This might include experts who can enhance the project proposal, community influencers who can help amplify the project’s visibility, or potential backers who might contribute substantial funding. By distributing these tokens, Sourcers can directly reward contributors for adding value to the project, whether by enhancing the proposal's quality, providing critical reviews, or assisting in outreach efforts to attract more funding.

Following the pre-mint phase, the remaining tokens are made available for purchase through a bonding curve. This pricing mechanism adjusts the cost of tokens based on their buying and selling activity, ensuring a transparent and fair distribution process for all potential investors.

### **Bonding Curve Dynamics**

**Detailed Pricing Mechanism**&#x20;

Catalyst employs a linear bonding curve to determine the price of Seed tokens as their supply increases. The curve starts at a base price of 0, corresponding to the pre-minted amount for the Sourcer and ascends linearly, indicating a direct proportionality between the number of tokens sold and the price per token. The price function `P(s) = m * (s - p)` calculates the cost for any given token 's', where 'm' represents the slope of the curve, 's' the current supply, and 'p' the pre-minted amount. Each new purchase or sale moves the price point along this curve.

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption><p>Illustrative Bonding Curve: This graph represents the price-supply relationship of Seed tokens on Catalyst. The curve starts with a pre-minted amount 'p' at the initial price point. It then linearly increases, reflecting the price 'P(s)' for each subsequent Seed token 's'. The shaded areas represent the funding stages: 'Under' indicates the amount below the funding goal, 'Over' indicates funding beyond the goal, and 'F' marks the funding goal in ETH. The slope 'm' indicates the rate at which the price increases with the supply.</p></figcaption></figure>

**Economic Implications**&#x20;

The nature of the bonding curve has several economic implications:

* **Stable Liquidity Mechanism:** Unlike [automated market maker (AMM)](https://docs.uniswap.org/concepts/uniswap-protocol) systems, which are common in cryptocurrency trading, bonding curves provide a more stable and predictable source of liquidity. Bonding curves eliminate reliance on individual liquidity providers who can influence the market by providing or withdrawing liquidity. This feature is particularly valuable in the early stages of funding when liquidity is typically lower, ensuring that participants can buy and sell Seed tokens without the worry of unpredictable and sudden liquidity shortages.
* **Market Indicator:** The bonding curve also acts as a signal for project momentum. As tokens are purchased and the price increases along the curve, it indicates growing interest and commitment to the project. The change in price acts as an indicator of attention and development progress, providing a measure of engagement into the project.
* **Temporary Market Structure:** The role of the bonding curve is transitory. It is designed to serve as an initial funding mechanism until the project reaches its funding goal. Upon achieving this milestone, the bonding curve market is closed, and the Seed tokens transition to a different market structure suitable for [IP tokens](https://docs.molecule.to/documentation/ip-tokens/what-are-ipts). This shift highlights the bonding curve's purpose: to facilitate early-stage funding and to establish a foundational market for funding, not dictate the long-term market dynamics for the resultant IP tokens. It paves the way for a more mature market structure that reflects the evolving nature of the project and its IP.

**Transfer Restrictions**

The Seed token contracts enforce specific transfer restrictions to maintain a unified market for project funding. During critical funding phases, tokens are non-transferable, ensuring that all funds contributed in exchange for Seed tokens are directed solely towards the project's funding goal, rather than to secondary traders. This policy is crucial to uphold the principle that buying a Seed token is inherently a form of project support, rather than a speculative investment. By restricting transfers, Catalyst ensures that the Seed tokens remain a tool for project fundraising and that the token's liquidity directly correlates to the project's genuine backing, rather than speculative trading activities.

These limitations are lifted once the project moves beyond the initial funding stage, transitioning Seed tokens into a new market structure. At this stage, the tokens may become transferable under certain conditions, which will be outlined in subsequent stages of the project's lifecycle. It's important for users to be fully aware of these liquidity conditions to understand how and when their tokens can be transferred, reflecting the project's transition from fundraising to further development or success realization stages.

**Exiting Seed Tokens & Withdrawing Contributions**

Catalyst has a unique resale model that deviates from traditional bonding curves. When participants choose to sell their Seed tokens, they are eligible to receive only the initial ETH amount they contributed. This approach is deliberately designed to realign the participant's focus from trading to supporting the project's success. By restricting the sale price to the original contribution amount (before a project reaches its funding goal), Catalyst ensures that the primary incentive for purchasing Seed tokens is to fund the project, rather than to engage in trading.

**Refund Processes**

In instances where a project does not reach its funding goal by the specified deadline (referred to as 'Expired'), Seed token holders may claim a refund. The refund amount is equivalent to the initial contribution minus fees incurred during the purchase. The process for claiming refunds is straightforward and is initiated directly on the platform, providing a clear exit strategy for funders in the case of expired projects. This mechanism provides a safety net for participants, allowing them to recover their funds if projects fail to reach their funding goal by the funding deadline. It is important for users to understand these processes thoroughly to manage their participation in the platform effectively.

**Fee Structure and Rationale**

The Catalyst platform incorporates a fee system in its Seed token economy, serving several critical functions:

1. **Fee Implementation**:
   * Upon purchasing Seed tokens, a 5% fee is deducted from the contributed ETH.&#x20;
   * When Seed tokens are sold back to the platform, a 5% exit fee is similarly applied.
2. **Fee Allocation**:
   * If a project is successfully funded, and terms of funding are successfully negotiated, the sourcer claims all project fees.&#x20;
   * If a project does not reach funding goal before funding deadline&#x20;
3. **Rationale**:
   * **Sustainable Platform Growth**: Fees are essential for the ongoing development and maintenance of the Catalyst platform. They ensure that the ecosystem has a steady stream of funding to support continuous improvement, and the introduction of new features.
   * **Risk Mitigation**: The exit fee discourages speculative behavior by ensuring that there is a cost associated with entering and exiting funding positions. This creates a more stable environment for project funding and encourages long-term funding in the project’s success rather than short-term trading.
   * **Resource Allocation**: Fees collected can be redistributed in ways that add value back to the ecosystem, such as providing liquidity.
