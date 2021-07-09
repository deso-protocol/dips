---
cip: 2
title: Rugpull Disclosure System 
author: Bloated Devish (@bloated-devish)
discussions-url: N/A
created: 2021-07-09
---

## One-Line Description

A method to responsibly disclose Rugpulling (coin owners selling their own coin, often but not always, in an effort to pump and dump) their own coin

## Explanation and Motivation

There has been a significant number of people who have been rug pulled recently as of May 2021. Several users have complained that they didn't anticipate the creator selling their own coin. This is especially relevant to new users, who are still learning how creator coin prices work on an Automated Market Maker.

### Goals and Non Goals

This CIP will not prevent anyone from selling their coin in perpetuity. This is because there are always valid reasons for creators (or anyone) to sell their coin. This CIP will not prevent any supporter from selling the coin they purchased. From the [vision](https://docs.bitclout.com/the-vision), the core feature of creator coins is to allow fans to be able to support/oppose creator's action with their buy/sell of the coin. The protection this CIP offers can be circumvented by situations mentioned below in the Weakness section. However, in these circumstances, we will anticipate the coin buyers to do the due diligence to cover these scenarios (or anticipate nodes offering tools to identify these scenarios for coin buyers). These scenarios are not differentiable from a legitimate situation where lack of confidence or an action by the creator can cause the coin price to tumble. In such or similar scenarios, this CIP will err on the side of ensuring the original use case envisioned in the vision doc be prioritized.

## Specification

Every account profile will have an extra field called ```RugPullProtectionStatus``` and ```RugPullEndTime```.

```
type UpdateProfileMetadata struct {
	...
	RugPullProtectionStatus bool
	// Timestamp at which protection is disabled. Any time >= the field value will have protection disabled.
        RugPullEndTime          timestamp
}
```

This field will be updated by creators voluntarily and would be available for nodes to display if a creator has opted into it. The default state is false and a creator needs to turn on for the feature to be true. For the top 15000 profiles, this feature will be by default turned on. Turning off this feature will use disclosure mechanism to notify clients that the feature is being turned off.

When ```RugPullProtectionStatus``` is not set, then the feature is off. When the feature is turned on ```RugPullProtectionStatus``` is set to ```True``` with ```RugPullEndTime``` unset. When creator intends to turn off the protection, the field ```RugPullEndTime``` is set with the actual end time.

The delay between the protection status request to be turned to false and actually enforced is proposed to a minimum of 15 days by the chain (but can be increased by the node operator). Once turned off, none of the disclosure requirements are applicable.

#### Profile Rugpull Disclosure
Creators can turn on rugpull protection anytime. When this is turned on, the owner's coin sale would be subject to disclosure. A creator can turn on rugpull protection back anytime.

Here are the possible scenarios when a creator can turn it on or off.

- Current status: Status missing and ```RugPullEndTime``` missing.
  - On: Possible to turn on. Will set ```RugPullProtectionStatus``` to True.
  - Off: No-op.
- Status on
  - On: No-op.
  - Off: Possible to turn off. Will trigger a disclosure and update profile with ```RugPullEndTime``` set with a future timestamp.
- Status on with ```RugPullEndTime``` set up with future timestamp.
  - On: Updates profile with ```RugPullendTime``` cleared.
  - Off: No-op
-Status on with ```RugPullEndTime``` set up with past timestamp. (similar to 1).
  - On: Updates profile with ```RugPullProtectionStatus``` to True and ```RugPullEndTime``` cleared.
  - Off: No-op

There will be no scenario where ```RugPullProtectionStatus``` is missing and ```RugPullEndTime``` is set.
The effect of toggling to On will be immediate. Nodes will make clear to creator that they can't sell any of their coins the next 15 days (or more based on the ```RugPullEndTime```) when initiating turning off.

### Coin Sales
Coin sales can be of the following scenarios:

- Coin sale by the creator of their own coins that they either bought or received.
- Coin sale by supporter, where the supporter received the coin, either from the creator or someone else.
- Coin sale by supporter of the coin, where the supporter purchased the coin.

When rugpull protection is enabled, the disclosure affects 1 and 2. The disclosure does not affect 3 under any circumstance. When the creator tries to sell their own coin, they are expected to first issue an ```IntentToSell``` operation for their creator coin. The remained of this section assumes that rugpull protection is enabled.

```
type CreatorCoinOperationType uint8

const (
	CreatorCoinOperationTypeBuy          CreatorCoinOperationType = 0
	CreatorCoinOperationTypeSell         CreatorCoinOperationType = 1
	CreatorCoinOperationTypeAddBitClout  CreatorCoinOperationType = 2
+	CreatorCoinOperationTypeIntentToSell CreatorCoinOperationType = 3
)

+type CreatorCoinLiquidationType uint8

+const (
+	CreatorCoinLiquidationTypeRegular 	CreatorCoinLiquidationType = 0
+	CreatorCoinLiquidationTypeTransfer	CreatorCoinLiquidationType = 1
+	CreatorCoinLiquidationTypeCreatorSale   CreatorCoinLiquidationType = 3
+)

type CreatorCoinMetadataa struct {
	...
	BitCloutToSellNanos       uint64
	CreatorCoinToSellNanos    uint64
	BitCloutToAddNanos        uint64
+	ReasonPostForIntentToSell *BlockHash
+	LiquidationType           CreatorCoinLiquidationType
        ...
+	// Maximum number of coins that are allowed between this time interval. There is no required minimum distribution.
+	MaxCreatorCoinsToSell  uint64
+	SellStartDate          timestamp
+	SellDuration           duration
+	// Applicable when a supporter sells transferred coins.
+	TransferredCoinHolderPK     []byte
}
```

For scenario 1, the blockchain will first set up an intent to sell, with an appropriate reason, maximum amount to sell and when the sell starts and duration of sell. The coin sale will never be earlier than 7 days from the date of issue of intent to sell. The sell duration is a positive duration (and should not be more than 15 days). If the creator tries to sell more than the ```MaxCreatorCoinsToSell``` within the sale eligible duration, the chain will not allow the creator to sell. The chain will sum the sale of coins from the beginning of SellStartDate to check this. The field ```CreatorCoinLiquidationType``` will aid the chain in indexing these transactions. The blockchain will NOT allow sale of coins before the ```SellStartDate``` and beyond ```SellStartDate + SellDuration``` (without another disclosure). Nodes can use the ```ReasonPostForIntentToSell``` field to provide the hash to their post, which can be used by nodes to link to the post, with preview on the reason to sell. This field is optional. A creator is not obligated to sell any of their coins in the provided period. They may choose to sell no coins, if their circumstance changes. However, creators cannot withdraw their intent to sell. This is to prevent creators spamming the chain with unwanted intent to sell.

```MaxCreatorCoinsToSell```, ```SellStartDate```, ```SellDuration``` are required fields. When a supporter sells a transferred coin ```TransferredCoinHolderPK``` field needs to be set. If this field is not set, then the transaction can only be made by the coin creator.

For scenario 2, there are two requirements that BOTH need to be satisfied:

- Any sale of these coins below 0.1% of the total coin supply will not need any disclosure.
(AND)
- Should aggregate sale of coins in the last 24 hour window (not calendar window) with the transferred coin sale exceed 10% of the overall supply, the supporter's coin sale will be immediately subject to disclosure.

If requirement 2 is satisfied, if the creator already has a disclosure in place, the supporter will be subject to the same disclosure. If there is no such disclosure, the supporter will be able to create a disclosure to the maximum of the amount that they possess. This disclosure shall also have ```MaxCreatorCoinsToSell```, ```ReasonPostForIntentToSell```, ```SellStartDate```, ```SellDuration``` and additionally ```TransferredCoinHolderPK``` indicating the public key of the supporter who intends to sell.

When pre-validating this disclosure, nodes need to ensure that:
- The supporter PK is different from the creator PK.
- The supporter has enough coins as mentioned in the disclosure. 
- MaxCreatorCoinsToSell is > 0 and at least 50% of the coins they possess.

Supporter intending to sell may be selling only a small fraction of the entire coin supply but still be subjected to the disclosure. Ways to avoid being subject to disclosure would be to reduce the amount being sold and trying again later. Nodes can provide the ability for supporters to schedule sales when the threshold drops below 10%.

If the supporter wants to dump their coins, because they aren't aligned with the creator anymore and they don't want to associate with the creator, the only option then is to transfer the coins back to the creator and lose their share or to send it to a burn address to lose those coins permanently. If the coins that the supporter possesses is transferred from another supporter, they would still be subject to the same rule. This effectively means, coins transferred from the creator in general are less mobile entities than coins purchased, as they are aligned with LONG strategy for the coin. Users who want to support the creator and retain the ability to sell it anytime should prefer to buy the coins rather than obtain transferred coins.

Scenario 3 has no restrictions and the supporter is free to buy and sell the coin the way they desire.

It is possible to have an inflight profile disclosure protection status toggle to false at the same time with an intent to sell disclosure. In real world, there is a good chance of profile update triggering intent to sell for transfer coins.


### Sale of transferred coins
To allow easy indexing, coins sold that were obtained as transfer will have an additional bit that indicates this was a transfer sale.

This allows validators to be able to lookup all transactions in the last 24 hours to compute the sale of transfer sales.

### Strengths
This CIP effectively creates a second class of coin mobility for transfer coins and creators owning their own coin. It also incentivizes the transfer coin holders to hold the creator coins long term.
It does not force anyone to sell or not sell. The core of the CIP is disclosure and the market may choose to ignore the disclosure because the creator has a valid reason to sell.

### Weakness
Situation below is a valid circumstance where rug pulling might still happen, but will not be prevented by the CIP.

-  Creator opens account A, FR at 100%. Creators buy their own coin in small quantities.
-  Creator creates another unrelated account B (or reuses an existing account). Seeds the account with CLOUT and buys up lots of coins for account A created in 1 above. This is a legitimate way to both transfer CLOUT to A (if the FR is high) and to hold coins of A.
-  The collusion between B and A is not public. Other users continue to buy coins of A.
-  B sells all of A's coin and cashes out. Coin value drops as the dump was already done.
-  A pretends to be shocked.

The telltale sign of this scheme is unknown accounts like B owning large amounts of coins. Scammers can potentially create multiple popular accounts and rug pull all of them at the same time. However, the barrier to do them is high due to the solution above. If collusion between A and B is not publicly known, then this scenario above is no different from the creator coin market reacting to abhorrent behavior by the creator.


## Backwards Compatibility
There are no issues with backwards compatability.

## Test Cases
Test cases will be created in the core and identity repositories

## Security Considerations
No additional security requirements.

## Data Increase
This CIP, if implemented as is, is expected to add x number of additional profile update operations and y additional Intent to Sell operations on the chain.

## Acknowledgements
The author would like to acknowledge Nigel (http://bitclout.com/u/nigeleccles) and V (http://bitclout.com/u/v) for their thoughtful discussion in Discord, which lead to this CIP.
