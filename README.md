# ✨ So you want to sponsor a contest

This `README.md` contains a set of checklists for our contest collaboration.

Your contest will use two repos: 
- **a _contest_ repo** (this one), which is used for scoping your contest and for providing information to contestants (wardens)
- **a _findings_ repo**, where issues are submitted (shared with you after the contest) 

Ultimately, when we launch the contest, this contest repo will be made public and will contain the smart contracts to be reviewed and all the information needed for contest participants. The findings repo will be made public after the contest report is published and your team has mitigated the identified issues.

Some of the checklists in this doc are for **C4 (🐺)** and some of them are for **you as the contest sponsor (⭐️)**.

---

# Contest setup

## 🐺 C4: Set up repos
- [ ] Add sponsor to this private repo with 'maintain' level access.
- [ ] Send the sponsor contact the url for this repo to follow the instructions below and add contracts here. 

## ⭐️ Sponsor: Provide contest details

Under "SPONSORS ADD INFO HERE" heading below, include the following:

- [ ] Name of each contract and:
  - [ ] source lines of code (excluding blank lines and comments) in each
  - [ ] external contracts called in each
  - [ ] libraries used in each
- [ ] Describe any novel or unique curve logic or mathematical models implemented in the contracts
- [ ] Does the token conform to the ERC-20 standard? In what specific ways does it differ?
- [ ] Describe anything else that adds any special logic that makes your approach unique
- [ ] Identify any areas of specific concern in reviewing the code
- [ ] Add all of the code to this repo that you want reviewed
- [ ] Create a PR to this repo with the above changes.

---

# Contest prep

## ⭐️ Sponsor: Contest prep
- [ ] Make sure your code is thoroughly commented using the [NatSpec format](https://docs.soliditylang.org/en/v0.5.10/natspec-format.html#natspec-format).
- [ ] Modify the bottom of this `README.md` file to describe how your code is supposed to work with links to any relevent documentation and any other criteria/details that the C4 Wardens should keep in mind when reviewing. ([Here's a well-constructed example.](https://github.com/code-423n4/2021-06-gro/blob/main/README.md))
- [ ] Please have final versions of contracts and documentation added/updated in this repo **no less than 24 hours prior to contest start time.**
- [ ] Be prepared for a 🚨code freeze🚨 for the duration of the contest — important because it establishes a level playing field. We want to ensure everyone's looking at the same code, no matter when they look during the contest. (Note: this includes your own repo, since a PR can leak alpha to our wardens!)
- [ ] Promote the contest on Twitter (optional: tag in relevant protocols, etc.)
- [ ] Share it with your own communities (blog, Discord, Telegram, email newsletters, etc.)
- [ ] Optional: pre-record a high-level overview of your protocol (not just specific smart contract functions). This saves wardens a lot of time wading through documentation.
- [ ] Delete this checklist and all text above the line below when you're ready.

---

# Yield Witch v2 contest details
- $23,750 USDC main award pot
- $1,250 USDC gas optimization award pot
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2022-07-yield-witch-v2-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts July 14, 2022 20:00 UTC
- Ends July 17, 2022 20:00 UTC

[ ⭐️ SPONSORS ADD INFO HERE ]

# Auditing the Witch v2
This contest is for the first major refactor on liquidations engine for the Yield Protocol. The existing engine has been heavily used since its launch in October 2021 and in that time we have found that liquidations have not been as fair as we would have liked. More often than not, liquidated users have lost all their collateral as we have failed to make liquidations competitive.

This second version of the Witch aims to:
1. Give us more flexibility in exploring different liquidation models.
2. Make liquidations more profitable for liquidators by allowing payments in fyToken.
3. Introduce a mechanism to reward starting an auction.
4. Allow fine-tuning of all parameters for any collateral/underlying pair.
5. Correct bugs.

For context on the rest of the Yield Protocol, please refer to the [Yield Protocol Docs](https://docs.yieldprotocol.com/#/).

## Liquidations Process
The liquidations process can be divided in three sequential steps: Auction, Payment, Closure.

### Auction

![image](https://user-images.githubusercontent.com/38806121/178305551-15d80e93-e7ef-490a-8a09-c7340b6eb58e.png)

Any vault that is undercollateralized in the Cauldron can be set up for an auction. The `cauldron.level` function returns the value of the collateral minus the value of the debt in a given vault, adjusted for collateralization ratio and, after maturity, borrowing rate increase. If the return value is negative, it means that the value of the collateral is below the value of the debt and the vault can be sequestered by the Witch for liquidation.

There are exceptions to this rule.
1. It is possible to set any given collateral/underlying pair as out of scope for the Witch. Some pairs should not be liquidated (such as DAI/fyDAI).
2. It is possible to set any owner to be protected from liquidations. The goal for this is to be able to run parallel liquidation engines.

Governance can choose whether liquidations for any pair should be of the whole vault, or of only a part of the vault.

Auctions follow a reverse Dutch auction, in which liquidators offer to pay part or all of the debt in the vault, and in exchange they get an amount of the vault collateral that increases with time. At the end of an auction a liquidator gets all the collateral under auction in exchange for paying all the debt under auction. It is possible for governance to set the amount of collateral that would be given for paying all the debt at the start of the auction, and the length of the auction. By modifying these two parameters governance can choose any linear progression for the auction.

Auctions can be configured to be profitable at the beginning of an auction by setting an `initialOffer` parameter above the LTV ratio of any given pair. Combined with auctioning only part of a vault this allows for a similar behaviour to Compound liquidations, if desired.

All the parameters for each individual auction are calculated in `_calcAuction`.

There is a soft limit on how much collateral can be set for auction concurrently for a given collateral/underlying pair. When the limit is passed, no new auctions for that pair are accepted. Note that the first auction to reach the limit is allowed to pass it, so that there is never the situation where a vault would be too big to ever be auctioned.

Once the auction is ready to start, the Witch takes the vault from the user with `cauldron.give`.

### Payment

![image](https://user-images.githubusercontent.com/38806121/178741273-ac070fa5-7254-4192-9c89-d43fd89f0725.png)

![image](https://user-images.githubusercontent.com/38806121/178740875-323a06e4-3997-4e57-95c8-a16d44172316.png)

In the Yield Protocol, all debt is denominated in fyToken. FYToken can be bought at a discount to their underlying under the appropraite market conditions and will often be most profitable mode of payment for liquidators.

It is also possible for liquidators to pay the debt directly with underlying, with a mechanism analogous to `Ladle.close`. If paying with underlying the Witch applies an exchange rate of 1:1, and if after maturity a borrowing fee would have accrued to the debt and would have to be paid, only if the payment is done with underlying.

Liquidators specify the maximum they want to pay (in fyToken or in underlying) and the minimum collateral they wish to obtain. The maximum works so that no more is taken from the liquidator than would be necessary to pay the whole debt.

The collateral obtained for the payment will be calculated in `calcPayout` according to the configured formula, and will be split into two cuts, one for the liquidator and one for the auctioneer.

Once the payment is calculated, accounting is adjusted in the Cauldron, collateral is sent to liquidator and auctioneer, and payment is taken from liquidator.

### Closure

![image](https://user-images.githubusercontent.com/38806121/178305843-96f7f140-9647-4e52-b923-8a91440c181a.png)

An auction can finish when there is no debt (with `_auctionEnded` called inside `_updateAccounting`), or anytime if the vault is collateralized (with an external call to `cancel`).

An auction finishing erases the auction data structure, and returns the vault to its original owner.

## Orchestration
The Witch has the following permissions on other contracts in the Yield Protocol:
 - `cauldron.give` - Allows to change the owner of any vault.
 - `cauldron.slurp` - Allows to change the balances of any vault, skipping collateralization checks.
 - `join.join` - Allows to make a join recognize unaccounted tokens it holds, or pull them from any user that has given approval.
 - `join.exit` - Allows to take tokens from a join and send them to any address.

Any of these permissions will have a catastrophic impact if abused.

## Governance Errors or Attacks
While governance actions will be carefully tested, it is in everyone's interest to disclose what's the worst that could happen.
 - `point`: Would disable liquidations as all payments fail.
 - `setLine`: Proportion being set to zero or at a very small number would disable liquidations. Setting a `line` for a non-existing pair might mean that the real pair is not set to be liquidable.
 - `setLimit`: Setting a `limit` for a non-existing pair might mean that the real pair is not set to be liquidable. Setting `max` to zero would only allow one concurrent auction. Setting `max` to 2^128-1 would allow any amount of colalteral to be auctioned concurrently.
 - `setAnotherWitch`: It can be misused to protect specific users from liquidation.
 - `setIgnoredPair`: It can be misused to disable liquidations for any pair.
 - `setAuctioneerReward`: It can be misused to direct all profit for auctioneers, effectively disincentivizing liquidators to liquidate.

## Design Decisions

### Dependencies
The Witch only depends on contracts from yield-utils-v2, which have been previously audited and are out of scope.

### Token transfers
The Witch uses the same batch pattern as the rest of the Yield Protocol, and never uses `transferFrom`. Instead, it expects the liquidator to have sent payment to the appropriate contract before calling any payment function. If the liquidator doesn't use some pattern for batching transactions, it is bound to lose the payment to front-running bots, and we are fine with that.

### Point
The `point` function is a standard function to change orchestration throughout the Yield Protocol. While it could be simplified for this contract, we prefer to keep it as is for consistency.

### OtherWitches
The name is a bit misleading, as any address can be entered. Better naming suggestions are welcome.

### Reentrancy
We trust other contracts in the Yield Protocol, including tokens accepted as collateral or underlying, as safe against reentrancy. They are only added through governance.

### AuctionEnded
Calling `_auctionEnded` from within `_updateAccounting` is a bit wonky, but saves gas. A better naming or code structure that keeps things cheap and readable would be welcome.

### Rounding
With rounding we have paid special attention to being able to finish all auctions, not so much to rounding-based attacks which we think unlikely.

## Audit Scope

### Users Getting Hurt
We expect every user interacting with the Witch to know what it is that they are doing, and as such we are not concerned about them getting hurt if they are not careful.

### Governance Process
We implement all of our governance changes through a multi-layered process to eliminate chances of a botched governance change where the wrong parameters are issued, including on-chain testing as part of the governance transaction. As such, we do not wish to put parameter checks in governance functions.

### Gas vs. Readability
There is a balance between achieving the minimum gas cost, and make code readable enough to be understood and make bugs easier to find. We will reject any gas-saving suggestions that harm readability, unless they save gas in the thousands per transaction.
