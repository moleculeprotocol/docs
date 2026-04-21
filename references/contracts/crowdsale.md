---
icon: people-group
---

# CrowdSale

## CrowdSale Contract Documentation

### Overview

The CrowdSale contracts enable fixed-price token sales for IPTokens, allowing researchers and Lab owners to raise funding from their community. The system includes three contract variants that form an inheritance chain, each adding capabilities on top of the base.

CrowdSale is the base contract: a simple fixed-price sale where contributors bid with an ERC-20 payment token and receive IPTokens proportionally if the funding goal is met. LockingCrowdSale extends the base by locking claimed tokens in a TimelockedToken contract for a configurable duration after the sale closes. StakedLockingCrowdSale further extends locking sales by requiring bidders to stake a secondary token alongside their bid, which is returned upon claiming.



### Contract Details



Contract: CrowdSale (base), LockingCrowdSale, StakedLockingCrowdSale

Type: Standard (not upgradeable)

Solidity: 0.8.18

License: MIT



### Deployments



Ethereum Mainnet: 0xF0A8D23F38E9CbBe01C4Ed37f23BD519b65BC6C2 (Etherscan)



Sepolia Testnet: 0x8cA737E2cdaE1Ceb332bEf7ba9eA711a3a2f8037 (Etherscan)



### How It Works



#### Sale Lifecycle



Every crowdsale follows a four-stage lifecycle: RUNNING, SETTLED or FAILED, and then claim.



An issuer calls startSale with the sale configuration — the auction token (IPToken to sell), bidding token (payment token to accept), funding goal, sales amount, closing time, and an optional permissioner for gating participation. The issuer must approve the contract to transfer the salesAmount of auction tokens before calling. A unique saleId is derived from the keccak256 hash of the sale parameters.



While the sale is RUNNING (before closingTime), participants call placeBid with the amount of bidding tokens they wish to contribute. If a permissioner is configured, the caller must provide a valid permission proof.



After closingTime passes, anyone can call settle. If total bids meet or exceed the funding goal, the sale moves to SETTLED and contributors can claim their proportional share of auction tokens (with any surplus bidding tokens refunded). If total bids fall short, the sale moves to FAILED and contributors can reclaim their full bid amounts.



The beneficiary (or anyone on their behalf) calls claimResults to withdraw the funding goal in bidding tokens (minus any protocol fee) after a successful sale, or recover the unsold auction tokens after a failed sale.



#### Proportional Distribution



When a sale is oversubscribed (total bids exceed the funding goal), auction tokens are distributed proportionally based on each contributor's share of total bids. The surplus bidding tokens are refunded. For example, if a sale has a 100 USDC goal and receives 200 USDC total, a contributor who bid 50 USDC receives 25% of the auction tokens and is refunded 25 USDC.



#### Protocol Fees



The contract owner can configure a fee in basis points (up to 50%) that is deducted from the funding goal amount when the beneficiary claims. Fees are sent to the contract owner. Fee configuration only affects future sales, not existing ones.



### Contract Variants



#### CrowdSale (Base)



The base contract handles the core sale lifecycle: starting, bidding, settling, claiming, and fee management. Claimed auction tokens are transferred directly to the claimer with no locking.



#### LockingCrowdSale



Extends CrowdSale by wrapping claimed auction tokens in a TimelockedToken contract. When claiming from a locking sale, tokens are locked until the sale's closingTime plus the configured lockingDuration (maximum 366 days). If the locking period has already expired by the time a user claims, tokens are transferred directly without locking. Each unique auction token gets a single reusable TimelockedToken contract, deployed via EIP-1167 minimal proxy.



#### StakedLockingCrowdSale



Extends LockingCrowdSale by requiring bidders to stake a secondary token alongside their bid. The staking token and amount ratio are configured at sale start. Staked tokens are held for the same locking duration as the auction tokens and are returned to the bidder when they claim. If the sale fails, both the bid and the staked tokens are returned.



### Functions



#### Write Functions



startSale — Starts a new crowdsale. Caller must approve salesAmount of auctionToken first.



placeBid — Places a bid with biddingTokenAmount of the sale's bidding token. Requires approval. Accepts an optional permission parameter for gated sales.



settle — Concludes a sale after closingTime. Sets state to SETTLED if funding goal is met, FAILED otherwise.



claim — Claims auction tokens (if SETTLED) or refunds (if FAILED). Accepts permission bytes for gated sales.



claimResults — Allows the beneficiary to withdraw proceeds (SETTLED) or unsold tokens (FAILED). Callable once per sale.



setCurrentFeesBp — Owner-only. Sets the protocol fee for future sales in basis points (max 5000 = 50%).



#### Read Functions



getSaleInfo — Returns the SaleInfo struct for a given saleId: state, total bids, surplus, claimed status, and fee.



contribution — Returns the bidding token amount a specific address has contributed to a sale.



getClaimableAmounts — Computes the auction tokens and refund amounts a bidder can claim from a settled sale.



### Events



Started: Emitted when a new sale begins. Includes the full Sale struct and fee configuration.

Bid: Emitted for each bid placed.

Settled: Emitted when a sale succeeds. Includes total bids and surplus amount.

Failed: Emitted when a sale does not reach its funding goal.

Claimed: Emitted when a participant claims tokens or refunds.

ClaimedFundingGoal: Emitted when the beneficiary withdraws proceeds.

ClaimedAuctionTokens: Emitted when the beneficiary recovers unsold tokens after a failed sale.



### Errors



BadDecimals: Auction token must have 18 decimals.

BadSalesAmount: Funding goal or sales amount is too low.

BadSaleDuration: Closing time must be between now and 180 days from now.

SaleAlreadyActive: A sale with the same parameters already exists.

SaleClosedForBids: Sale has passed its closing time or is no longer RUNNING.

BidTooLow: Bid amount is zero.

SaleNotFund: Sale does not exist.

SaleNotConcluded: Attempting to settle before closingTime.

BadSaleState: Operation requires a different sale state than the current one.

AlreadyClaimed: Beneficiary has already claimed results for this sale.

FeesTooHigh: Fee exceeds 50% (5000 basis points).



### Security Considerations



Not upgradeable: CrowdSale contracts are standard (non-proxy) deployments. Address is permanent and code cannot change.

Reentrancy protection: All claim operations use OpenZeppelin's ReentrancyGuard.

Permissioner gating: Sales can optionally enforce participation rules via an external Permissioner contract.

Decimal enforcement: Auction tokens must be 18 decimals. Bidding tokens can have arbitrary decimals.

Fee cap: Protocol fee is capped at 50% and only applies to future sales.

Audited: Covered by the Pashov security review.



### Related Contracts



IPNFT — The IP-NFT that backs the IPToken being sold.

Tokenizer — Creates the IPToken that serves as the sale's auction token.

TimelockedToken — Used by LockingCrowdSale to lock claimed tokens.

SchmackoSwap — OTC marketplace for trading IP-NFTs directly.



### Resources



Source Code: GitHub (CrowdSale.sol, LockingCrowdSale.sol, StakedLockingCrowdSale.sol)

ABI: Available in @moleculexyz/common/abis

Audit: Audit Report
