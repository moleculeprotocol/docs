---
description: >-
  Crowdsales of dIPs enable researchers to raise funding for the development of
  an IP-NFT.
---

# 👨👩👧👦 What is a crowdsale of dIPs?

## Introduction

In an era of decentralized finance and blockchain technology, new ways of funding projects are rapidly emerging. Among these, the `CrowdSale.sol`, `LockingCrowdsale.sol`, and `StakedLockingCrowdsale.sol` are Ethereum smart contracts that provide unique fundraising mechanisms, particularly suitable for funding scientific research.

### Understanding Ethereum Smart Contracts

Before we delve into specifics, let's briefly define an Ethereum smart contract. These are self-executing contracts with the terms of the agreement between buyer and seller being directly written into lines of code. They automate transactions and eliminate the need for intermediaries, reducing cost and increasing transparency.

### CrowdSale.sol

[`CrowdSale.sol`](https://github.com/moleculeprotocol/IPNFT/blob/main/src/crowdsale/CrowdSale.sol) is the fundamental contract that enables the creation of a crowdsale, which is a fundraising mechanism where individuals contribute [ERC20 tokens](https://eips.ethereum.org/EIPS/eip-20) in return for dIPs of a project.

Think of this as a digital variant of traditional fundraising, where instead of getting a thank-you note or a freebie, contributors receive tokens that might have utility in the project's ecosystem, or even represent shares in an organization.

#### Use Case in Scientific Research:

Scientific research can propose a project, define the number of tokens up for sale, the price per token, and the duration of the sale. Supporters of the project can purchase these tokens, effectively funding the research.

### LockingCrowdsale.sol

[`LockingCrowdsale.sol`](https://github.com/moleculeprotocol/IPNFT/blob/main/src/crowdsale/LockingCrowdSale.sol) extends the capabilities of `CrowdSale.sol` by adding a locking mechanism to the tokens purchased during the crowdsale. This means that the tokens contributors receive are "locked" for a certain period, and cannot be transferred or sold until the lock expires.

#### Use Case in Scientific Research:

This can be particularly useful in scientific research as it ensures that contributors (token holders) are committed to the project for a certain duration. It can help to avoid quick sell-offs and stabilize the project's token economy.

### StakedLockingCrowdsale.sol

[`StakedLockingCrowdsale.sol`](https://github.com/moleculeprotocol/IPNFT/blob/main/src/crowdsale/StakedLockingCrowdSale.sol) adds another layer of complexity and control. Here, in addition to the purchase, contributors also need to stake (lock up as collateral) a certain amount of another token to participate in the crowdsale. These staked tokens may also be vested over time, providing an added incentive for contributors to remain engaged with the project over a longer period.

#### Use Case in Scientific Research:

For instance, a research project could require that contributors not only purchase project tokens but also stake a certain amount of a specific token (like a governance token of the research platform). This would ensure the contributors have a sustained interest in the project's success. The staked tokens could be returned over time, potentially with interest or rewards, further incentivizing long-term involvement.

## VITA-FAST Example

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>Alice and Bob place bids in a crowdsale of VITA-FAST dIPs. The crowdsale has a 2-day Sale Period. The total bids (20 ETH) exceeded the Fundraise Goal (16 ETH), so the refunds (4 ETH + 4,000 VITA) are returned after the Sale Period ends and the crowdsale is settled. Because VITA is the Staked Token for this crowdsale, only the amount of VITA locked can be bid in the crowdsale. Refunds are given <em>pro rata at the conclusion of the Sale Period;</em> Alice's bid (15 ETH) was three times Bob's bid (5 ETH), so her refund (3 ETH + 3,000 VITA) is three times Bob's (1 ETH + 1,000 VITA). After settlement, allocations of dIPs can be claimed, but are locked (non-transferrable) for the duration of the 60-day Locking Period. After the Locking Period, Locked dIPs and vested VITA (vVITA) can be unlocked, making them transferrable.</p></figcaption></figure>

## Conclusion

By employing such contracts, scientific research projects can create a community of contributors and, through staking and vesting mechanisms, encouraged to stay involved over time. This can provide a robust and committed source of funding, contributing significantly to the advancement of a research project.
