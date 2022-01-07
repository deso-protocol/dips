---
dip: 6
title: DAO Coins: The Future of Fundraising on the Internet
author: @diamondhands
discussions-url: https://github.com/deso-protocol/dips/discussions/188
created: 2022-12-06
---

## One-Line Description

This proposal introduces DAO Coins to the DeSo ecosystem

## Explanation and Motivation

What if raising money for a startup could be as easy as creating an Instagram
account? Pick a username, set a description, maybe add a profile picture, and
boom-- thousands of people from countries all over the world have given you
millions of dollars to pursue your dreams.

Today, after talking to over a
hundred founders, after studying every fundraising product we could get our
hands on, and after personally having raised over $400 million myself as an
entrepreneur, I am excited to announce a new product that we believe will
define the future of fundraising for entrepreneurial ventures and audacious
ideas. A product we think will mix investing and social better than ever before, and
make it easier than ever for founders to raise money to build the next big
thing.

**We call this new product: DAO Coins.**

Less than a year ago, we introduced a breakthrough social investing product to
the world called [Creator Coins](https://docs.deso.org/#what-are-social-tokens-aka-creator-coins). 
They made it so easy to invest in someone that
[a frenzy of speculation ensued](https://www.newyorker.com/tech/annals-of-technology/the-dark-democratizing-power-of-the-social-media-stock-market), 
and the investing continues today with volume of
about $100k-$2M per day. Creator Coins also allowed many creators to earn [hundreds of thousands
of dollars](https://cloutomy.com/rank/usd) off their content, where traditional social social platforms
had never paid them a penny.

**But Creator Coins were meant for *people*-- they were
never meant to support organizations.**

After the launch of Creator
Coins, [over two hundred entrepreneurial projects](https://bithunt.com/explore) sprung up, using Creator Coins
to finance everything from NFT platforms ([1](https://polygram.cc), [2](https://nftz.zone)) to 
search engines ([1](https://cloutavista.com), [2](https://searchclout.net)). We noticed founders
were missing key things they needed, like social governance features, access to
alternative fundraising models, flexibility in the coin supply, the ability to
set who's on their cap table, the ability to withhold coins for themselves,
and many other related things.

**Taking in all of this feedback, and studying all
existing solutions on the market today, DAO Coins are a re-imagination of the
Creator Coin concept to give projects what they need to not only raise money,
but also to run their organizations in the long-term.**

What's more, we believe
DAO Coins will challenge traditional corporate structures by offering founders
better access to capital, more liquidity, and more direct governance over their
organizations.


## How DAO Coins Work

There are two components to DAO Coins: A part that's on the blockchain, and an
“application layer” component that will be powered by [smart services](https://www.deso.org/blog/smart-services). Here, we
will mainly concern ourselves with the on-chain component.

In terms of what's actually needed on the DeSo blockchain to support DAO Coins,
that part is relatively simple:
* First, we are making it so that ordinary DeSo profiles can have DAO Coins. This means founders can
  start by simply creating a DeSo profile for their project on an app like
[diamondapp.com](https://diamondapp.com). The profile should contain a description of what the project
aims to achieve, and this description can be as complex as it needs to be.
  - For more complex projects that want to implement governance protocols, the
profile description can include a link to a terms of service or prospectus that states how 
funds raised by the DAO will be managed, among other things.
* Once a profile is created, the following
very simple on-chain functions become available on the profile:
  - **Mint new DAO
coins**, which can only be executed by the profile owner. Note that DAO coins are
distinct from Creator Coins mentioned earlier.
  - **Burn DAO coins**, which anyone
can execute to take coins they own out of circulation. 
  - **Disable minting of DAO
coins**, which prevents any new coins from ever being minted in the future (only
the profile owner can execute this one). 
* These functions can then be called or
delegated via derived keys to smart services that implement crowdsale mechanics, cap table
management tools, voting tools, etc... We will go into the possibilities for smart services
built on DAO Coins concretely in a follow-up post.
* Once
a profile is created, the DAO also gets access to all of the social features
native to the DeSo blockchain, including posting, following, engaging with
other users/DAOs, issuing NFTs, etc... 
* It will also be possible to configure a profile with an "m of n" multisignature. 
This will allow DAOs to have a board of
directors in much the same way a traditional company would. But this will be
discussed in a follow-up post.
* Users of smart services will be able to set spending limits on derived keys
in order to enable smart services to manage their balances in a more trustless
fashion. This will enable developers to build smart services that implement DeFi
products on DAO Coins, such as staking, yield farming, and lending protocols. 
But this will be discussed in a follow-up post.

Those familiar with Ethereum may recognize that DAO coins are similar in spirit
to the ERC-20 standard, and that's correct! However, unlike ERC-20 coins, DAO
coins come with a social identity baked in via the DeSo profile. This not only
makes it very easy to search for and find a particular DAO, but it also gives
the DAO's operators a built-in platform on which to make announcements,
accumulate followers, engage with other accounts, e.g. on 
[diamondapp.com](https://diamondapp.com), issue
NFTs, e.g. on [polygram.cc](https://polygram.cc), run governance votes, 
and much, much more, as we'll get
into later.

**Simply put, associating DAO coins with a native social identity sets the
DeSo community up to build products with much better UX than what's traditionally
been possible in the past.**

## The App Layer
Once the very simple on-chain primitive has been defined as above, there are
virtually limitless possibilities for the "app layer" to define different
schemes for DAO formation, governance, fundraising, etc... We will start to propose many
of these in concrete terms in a follow-up series we call "Request for Developers."
For now, though, we can't resist
teasing one product suggestion (with *many more* to come later):
* **Subreddit DAOs**: Imagine a Reddit competitor where each subreddit has its own
DAO Coin that community members can invest in. Funds raised could be allocated to
support moderators or they could be allocated toward causes that the subreddit's
community is excited about, like [investing in historical artifacts](https://www.constitutiondao.com/).
DAO Coins could also be rewarded to users based on activity and engagement.
This can be easily implemented as a Smart Service, with all assets and content
stored on the DeSo blockchain to enable portability across frontends.

## Code Changes

Broadly, there are two approaches to making these changes in the code.
We can either introduce a new bookkeeping structure for DAO Coins that
is distinct from Creator Coins, or we can re-use the bookkeeping that
currently powers Creator Coins to make it work for DAO Coins. After
some discussion, we propose the former because it allows a profile to
have both a DAO Coin and a Creator Coin simultaneously, which can be
useful.

Below is the technical proposal for implementing DAO Coins into the
DeSo blockchain:
* New TxnType
  - TxnTypeDAOCoin, similar to TxnTypeCreatorCoin
* Add new [CoinEntry](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_types.go#L777) to profile:
  - Call it DAOCoinEntry and add it [here](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_types.go#L850)
under the existing CreatorCoin CoinEntry.
  - This will be a second instance of CoinEntry with the exact same fields
* Changes to [db\_utils.go](https://github.com/deso-protocol/core/blob/main/lib/db_utils.go) are required:
  - Need [two new indexes](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/db_utils.go#L166) for DAO coins that are identical to what we 
currently have for CreatorCoins:
    * \_PrefixHODLerPKIDCreatorPKIDToBalanceEntry = []byte{33}
    * \_PrefixCreatorPKIDHODLerPKIDToBalanceEntry = []byte{34}
* \_connectDAOCoin
  - This will be similar to [\_connectCreatorCoin](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_creator_coin.go#L1428), 
only it supports different OperationTypes. Recall that CreatorCoin supports "buy" and "sell."
  - The operations that \_connectDAOCoin will support are as follows:
    * Mint: Creates DAO coins for the caller only if they are the own the profile associated with the DAO coin being called
    * Burn: Destroys DAO coins held by the caller
    * DisableMinting: Prevents new coins from ever being minted. Must be called by the owner of the profile associated with the DAO coin.
* \_connectDAOCoinTransfer
  - This will be virtually identical to [\_connectCreatorCoinTransfer](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_creator_coin.go#L1463), only it will act 
on the DAOCoin balances in the db, rather than on the CreatorCoin balances

## Deployment
We will need to set a hard fork block height for \_connectDAOCoin and \_connectDAOCoinTransfer.

TxnTypeDAOCoin will be automatically rejected in the parsing phase if a node hasn't upgraded.

## Other Considerations

### Using Creator Coins to Bootstrap a DAO

Fundraising tools for DAO Coins can be easily implemented as Smart Services without relying
on Creator Coins. Users are not
limited to using Creator Coins to bootstrap their DAO, as we describe in this
section, and none of this is prescriptive. However, because fundraising for a DAO using
a Creator Coin is something that's possible and straightforward, 
we figured we would
describe what it could look like.
* Someone wants to raise money for a new DAO
* They announce they're going to allow purchase of their Creator coin at a
  particular time by lowering their Founder Reward on their profile.
* They announce that the ability to purchase will be open for X amount of time,
  after which point the FR will be increased back to 100%.
* After the FR is increased to 100%, 100M DAO coins will be minted by the
  creator and 10% will be distributed to everyone who bought the Creator coin.
* This can be an airdrop OR the creator can require that the creator coin be
  "redeemed" for the DAO coin by having the purchaser transfer the creator coin
to him first. If he does this, then the creator can utilize the proceeds of the
DAO coin by selling the creator coins he receives on the bonding curve.

### Converting a Profile From a Creator Coin to a DAO Coin

Creator Coins work great for people. They allow anyone to instantly buy or
sell one's coin against a bonding curve that rewards people who invest early
in someone. That said, because Creator Coins were the only option initially,
there are some organizations that raised money with a Creator Coin who now
would benefit from converting to a DAO coin. This section is for these legacy
projects.

Projects that feel a DAO Coin would be a better fit for their needs can convert
their Creator Coin into a DAO Coin at any time by doing the following. Note that
this is completely optional, however, and that none of this is prescriptive:
* Pre-announce your intent to convert your Creator Coin holders into DAO Coin holders
* Set Founder Reward to 100% to halt purchases of your Creator Coin
* Mint however much DAO Coin you want to mint
* Optionally disable future minting to fix the supply of the DAO Coin
* Two options:
  - Airdrop the newly-minted DAO coins onto existing holders of the Creator
Coin. This allows the Creator Coin holders to cash out their money from the bonding
curve after receiving their airdrop.
  - Announce that Creator Coins can be redeemed for DAO Coins by sending them
to the profile's public key. This allows the profile owner to cash out the funds by
selling against the Creator Coin bonding curve.

Importantly, because DAO Coins have non-standard supply, converting to a DAO
coin may result in a profile no longer ranking on certain apps. As such, we
don't recommend this step unless one is sure that a DAO Coin is better for one's
needs.

### Secondary Liquidity

Because DAO Coins are not tied to a bonding curve like Creator Coins are, the 
DeSo blockchain will not
support on-chain secondary trading of DAO Coins initially. This, 
however, leaves wide open a great opportunity for someone to implement a simple Smart 
Service that allows for the trading of DAO Coins.

If DAO Coins are successful,
we imagine this Smart Service could get quite popular. Moreover, although the
DeSo blockchain can fairly easily support an on-chain order-book exchange for
DAO coins, we see no need to incorporate this functionality if smart services
are capable of adequately serving the need.


## Security Considerations

This upgrade should not impact the security of the chain. The implementation proposed
has very little risk of corrupting existing transaction types like Creator Coin
transactions.

## Alternate designs considered

As mentioned, we considered a design that uses the same balance entry for DAO Coins
and creator coins. However, we dismissed this approach because we think it's
beneficial to allow a profile to have a DAO Coin and a Creator Coin simultaneously.

## Acknowledgements

TODO
