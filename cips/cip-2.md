---
cip: 2
title: BitClout NFTs
author: diamondhands
discussions-url: https://github.com/bitclout/cips/pull/92
created: 2021-07-15
---

# One-Line Description

Add NFTs to the BitClout blockchain at the consensus level and to the reference UI

![An example of a BitClout NFT that we think needs to exist...](.gitbook/assets/frame-539.png)

# Explanation and Motivation

Copied from [here](https://docs.bitclout.com/bitclout-nfts) to make it easier to leave comments.

## What are NFTs?

Non-Fungible Tokens \(NFTs\) are digital assets that can be bought and sold, typically representing a piece of digital content. For example, an artist can publish a digital image as an NFT, and put it up for sale to the highest bidder. When they do this, the history of who owns the image can be tracked on the blockchain as a way of showing the art piece's provenance. And even though anyone in the world can typically see the image, there is only one person who provably owns it, just as if the piece were a painting hanging in a museum.

The easiest way to really understand NFTs, though, is to actually look at some examples. Below we list examples of popular NFT concepts, as well as popular NFT platforms, all of which served as the inspiration for the BitClout NFTs product. **Importantly, because BitClout is an inherently social platform, we anticipate the use-cases for NFTs will extend far beyond just digital content, and we discuss this in detail in the next section.**

Examples of popular NFT concepts:

* [Beeple's collage](https://www.theverge.com/2021/3/11/22325054/beeple-christies-nft-sale-cost-everydays-69-million)
* [CryptoPunks](https://www.larvalabs.com/cryptopunks)
* [Bored Apes](https://boredapeyachtclub.com/)
* [CryptoKitties](https://www.cryptokitties.co/)
* [NBA Topshots](https://nbatopshot.com/)

Popular NFT marketplaces:

* [OpenSea](https://opensea.io/)
* [Nifty Gateway](https://superrare.co/)
* [Rarible](https://rarible.com/)
* [SuperRare](https://superrare.co/)
* [Zora](https://zora.co/)
* [Foundation](https://foundation.app/)
* [Valuables by Cent](v.cent.co)

## Why NFTs on BitClout?

With BitClout, we've always been focused on creating innovative and engaging ways for creators to monetize. We started with creator coins, which allow you to invest in people you care about. Then we added diamonds, which allow you to give tips that show up as badges on the receiver's profile.

**NFTs are our third major breakthrough, and they tie everything together.** Before we decided to work on NFTs, we studied every major NFT platform to determine what was working in the space, and how we could make NFTs on BitClout really shine in a unique way. We came away very excited by the following possibilities...

### 1\) Mixing NFTs and Social Media

When someone buys a piece of art or a collectible item, they do so in part because it brings them personal joy, but in part because they want to show it off. A major superpower BitClout has is that every feature that's added to it has an inherent social component built-in, and NFTs are no exception.

In the case of BitClout NFTs, we have an opportunity to show off a user's NFT collection on their profile, and to allow users to engage around their NFTs via comments, likes, diamonds, and more. Suddenly, the act of buying an NFT shifts from a purely personal and/or economic motive to an inherently social one. In addition, because BitClout has a native concept of identity in the form of a user's profile, the reputation of the issuer is tied into the NFT in a much more meaningful way, especially for celebrities and superstars with pre-existing brands. This not only increases the value of BitClout NFTs, but we think it will also lead to all kinds of interesting dynamics that mix collecting, flexing, and social.

Below are just some examples of the possibilities...

#### **New NFT Use Cases**

* **Collectible ticket stubs.** If you were to sell tickets to a concert in the form of BitClout NFTs, then every attendee would automatically get a virtual ticket stub on their profile commemorating the event that their friends would get to see \(not to mention the extra promo you'll get from your coin-holders!\). Could you imagine if [@3LAU](https://bitclout.com/u/3LAU) sold his tickets as BitClout NFTs? This mechanic could also be used to sell tickets to exclusive events like the premier of a movie or an exclusive gala.
* **Physical memorabilia: The digital collector's room**. Imagine selling a physical piece of memorabilia, like a prop from a movie set, with an NFT attached, issued by the original seller, that the 

  buyer gets to flex on their profile. This turns a user's profile into an inventory of their collector's room, where you can see all of the cool things they own, both in the digital and physical world, with

  NFTs serving as certificates of authenticity issued and signed directly by the original seller. Could you imagine if someone like [@GeorgeTakei](https://bitclout.com/u/georgetakei) from Star Trek cleaned out his closet one day using BitClout NFTs?

* **Exclusive experiences.** Selling experiences as NFTs makes unique sense on BitClout. For example, creators with large followings can offer to have dinner or to host a Q&A with a handful of their biggest fans by minting and selling a "one of 10" NFT. With BitClout, because NFTs are inherently social, the creator can engage their followers by asking them to comment explaining why they want to join before they place a bid. The creator then has full control over determining the winners, and those winners not only get to meet the creator, but they also get to sport the fact that they did on their

  profiles forever. Maybe [@wolfofwallst](https://bitclout.com/u/wolfofwallst) could give Warren Buffet's charity lunch some competition!

* **Exclusive unlockable digital content.** BitClout NFTs have an "unlockable" portion that only the winner of the NFT gets to see. This creates interesting use-cases around selling hyper-exclusive digital goods. For example, an artist can drop an album a week early as an unlockable 1/10,000 NFT such that only her true fans who win the NFT are able to listen to it ahead of time. This would result in extra cashflow for the artist while still allowing them to capture the same streaming revenues a week later. Could you imagine getting early access to [@thechainsmokers](https://bitclout.com/u/thechainsmokers)' next album, and getting an NFT along with it?
* **Exclusive chat groups.** Creators can offer exclusive chat groups using BitClout NFTs to gate

  access. For example, a creator can sell a 1/100 NFT such that any current owner of the NFT is able

  to participate in an exclusive Telegram group, weekly Zoom call, etc... We already saw this happening with creators like [@craig](https://bitclout.com/u/craig), but it also makes sense for sports insiders like [@adamschefter](https://bitclout.com/u/adamschefter).

* **Interactive content.** For content creators, BitClout NFTs can be used as a way to solicit feedback from fans, or to guide the direction of content. For example, the creator of a podcast can sell an NFT where the winner gets to decide what their next episode is going to be about. The creator can solicit comments from users before they place their bids, and they have ultimate control over who they choose as the winner. Alternatively, music artists can offer to put an NFT winner's name in a song or include them in a music video, and the winner would have the NFT on their profile to commemorate the experience. The creator of a movie or short film could sell producer credits in the final cut as NFTs. Could you imagine if the winner of a BitClout NFT could decide the topic for [@shaanvp](https://bitclout.com/u/shaanvp)'s next show, or win a shoutout at the end of a [@jakepaul](https://bitclout.com/u/jakepaul) or [@loganpaul](https://bitclout.com/u/loganpaul) fight? How about a Clubhouse AMA with [@alexisohanian](https://bitclout.com/u/alexisohanian), where the winners of a "one of 10" NFT get to come on-stage first? Or maybe we can finally get [@BennyBlanco](https://bitclout.com/u/BennyBlanco) to finally bring us that BitClout Boys single we've all been waiting for, as gloriously sought-after NFT.
* **Counterfeit-proof Rolexes, Chanel handbags, etc...** Major brands have a big problem with

  counterfeitting: A real Rolex is worth much more than a knockoff, but knockoffs can often be

  so good that it's difficult to tell them apart. Now, imagine a solution based on BitClout NFTs

  whereby a luxury brand creates an official BitClout profile, and offers an NFT associated with

  every single sale of their products. Now, a user not only gets digital, unforgeable proof that

  they own a real item, but they also simultaneously get to show off their purchase on their profile that all of their friends can see. Then, if they ever resell their Rolex, they can transfer the NFT along with it, allowing it to serve as a certificate of authenticity issued and digitally-signed directly by the brand, and that tracks the provenance of the item for its entire lifetime.

* **Digital trading cards.** Any sufficiently-well-known creator can create digital trading cards of themselves simply by issuing a "one of N" NFT. All they need to do is create a unique piece of artwork, like a [cryptopunk](https://www.larvalabs.com/cryptopunks) drawing of themselves, and their biggest fans can sport it on their profiles. Notably, each BitClout NFT has a serial number, so each one will be special, even within the same issue. [@ab84](https://bitclout.com/u/ab84), could you be the first BitClout NFT trading card!
* **Fine art.** Major artists have shown that NFTs are going to be a big part of the future of fine art. They not only allow anyone to enjoy the artist's work, but they also do a much better job of tracking the ownership of a piece, which means the provenance can't be forged. The fact that BitClout also incorporates the artist's identity, via their profile, into the minting of an NFT should, we hope, further increase the value and utility of NFTs issued by artists. Artists on BitClout have already been innovating extremely fast, and we're so excited to take things to the next level with BitClout NFTs. Maybe we can even get [@beeple](https://bitclout.com/u/beeple) to finally claim his profile!
* **The future of Charity.** Charities can create profiles on BitClout, just like ordinary people. When they do this, anyone can elect to send them BitClout as part of the sale of their NFT. For example, someone could auction off a dinner with themselves, but specify that all the proceeds will go to The Red Cross. They would then be able to digitally prove that the funds went to that charity. Alternatively, a charity can participate in the fun directly by issuing NFTs of their own. For example, a charity could issue NFTs where each one represents a particular acre of trees that will be planted. This allows the owner to show off their contribution to any cause they care deeply about, which could significantly increase the amounts people are willing to give. It's a bit surprising that social media and charity aren't more closely linked today-- but we believe BitClout can finally change that, and make giving easier and more fun than ever before.
* **Owning a piece of history.** On BitClout, any post that a user makes can also be minted as an NFT and sold. The user who "owns" the resulting NFT can be seen as owning a piece of history. For example, if a sitting US president theoretically joined BitClout in the future and used it to make a monumental announcement, like the end of US COVID lockdowns, someone could own that very special post, and all proceeds could be donated to a charity of the president's choice. We'll also settle for another shirtless pic of [@chamath](https://bitclout.com/u/chamath), though, just to be clear.

The above list is just the beginning; it's just what we've come up with so far. We can't wait to see what the community produces once BitClout NFTs are actually out in the world.

In the past, NFTs and social media have been separate: You mint an NFT on some platform, and then post about it on social media. Now they can come together, as they were always meant to be, increasing engagement, reach, value, and monetization for creators.

### 2\) NFT Cashflows to Coin-Holders

Creator coins are a major BitClout superpower that we are taking to the next level with the launch of NFTs. On BitClout, a percentage of the sale of each NFT can be sent back to a creator's coin-holders as a cashflow \(including on secondary sales\). With this key feature, BitClout NFTs "close the loop" between a creator's activities on BitClout and the value of their coin.

**Suddenly, creator coins are no longer objects of pure speculation; rather, they are directly linked to a creator's activity on the platform. This means that, for the first time, followers can participate in a creator's growth rather than watching from the sidelines as they rise to stardom.** This has never been possible before, and it changes the relationship between a creator and their fans, from one in which fans pay for their work, to one in which they invest in the creator, and grow together.

Moreover, tying cashflows to creator coins makes it so that any creator who wants to market a new piece of content has a whole army of coin-holders that are invested in their success, and will help them spread the word. Distribution is no longer solely the creator's job, and they don't need to sign their life away to a corporation in order to get it. Your fans are your investors and your distributors at the same time because they're economically aligned with you in a way that wasn't possible before BitClout.

Finally, and perhaps most interestingly, these cashflows do not ultimately inhere to the creator themselves; rather, in the same way a Picasso painting continues to fetch a high price after its original primary sale, a creator's NFTs on BitClout can continue to trade and produce cashflows for creator coins long after the creator is gone. **Thus, in some sense, tying cashflows to creator coins makes owning them analogous to owning a percentage of every sale of every piece of work the creator has or will ever produce on BitClout. Could you imagine if Picasso had a creator coin linked to all of his works?**

## How BitClout NFTs Work \(With Visuals\)

The easiest way to see how BitClout NFTs will work is to check out [this deck](https://docs.google.com/presentation/d/1iklCqm85gjnqv_CyTG7-vXMBmSG1yhtBE_-oexUGZ0A/edit#slide=id.ge30f8df505_0_0).

Very simply, the steps to minting and selling a BitClout NFT are as follows:

* Create a post, which consists of a snippet of text and an embedded image or video. All NFTs on BitClout start as posts, and you can turn any pre-existing post into an NFT.

* Hit "Mint NFT" and select from the options:
  * You can mint either a "one of a kind" or "one of N" NFT. In the latter case, there will be multiple winners of the same piece of content.

  * The creator can set a creator royalty and a coin-holder royalty. This is a percentage of the sale that will go to the creator and to the creator's coin-holders as a cashflow. This cashflow hits on every **secondary sale** of the NFT as well. The BitClout platform does not take a fee. Note we are working on allowing arbitrary public keys to be specified as a means of programatically distributing proceeds to other accounts, such as charity accounts.

  * Optionally, the creator can set a piece of unlockable content that only the winner of the NFT will get access to. This feature enables hyper-exclusive experiences to be built on BitClout NFTs, like one of a kind songs that only the winner can listen to.

* Once an NFT is minted, users can bid on the NFT. They must have enough in their wallet to cover the bid, but nothing is withdrawn from their wallet until the auction is closed by the creator. This allows users to bid on as many things as they like.

* Whenever the creator is ready, they can close the auction by selecting a winner, or winners in the case of a "one of N" NFT. Importantly, the creator has full control over who gets to own their work; they don't have to give it to the highest bidder.

* Once the auction is over, the winner\(s\) get to show off the NFT on their profile. It shows up in their NFTs tab, and it can be pinned to their main page.

We didn't want to over-complicate things, and we believe this simple set of features enables all of the interesting use-cases described previously.

Finally, as a means of concentrating liquidity around certain NFTs, we have designed a system that allows node operators to schedule "showcases." Showcases work as follows:

* A node operator selects a collection of NFTs that they want to showcase.
* The node operator schedules these NFTs to "drop" at a certain time.
* At the scheduled time, the new NFTs are showcased on the Home page in their own tab.

Using this system, a node operator can curate a collection of NFTs every week, or even more frequently, and engage the community around them. **This, in some sense, allows node operators to serve as the curators of their own digital galleries, with each drop introducing a new exhibition.**

## Launching BitClout NFTs: A Community Contest

The launch of BitClout NFTs is a highly-anticipated event, and what better way to bring the product into this world than by working together with some of our most-beloved community members?

**Starting today, and ending on the launch date of NFTs \(TBD\), the core dev team will be accepting NFT submissions from all users to participate in bitclout.com's first major NFT drop!** Any member of the community can submit an NFT using [this form](https://docs.google.com/forms/d/1stA6z-w6j4tgSpgenbgSfrasSDjmgiMeFH6W-Qy2lJU/viewform), and the winners will be the first ones ever to be featured on bitclout.com's NFTs tab. Once the NFTs are up, all users will then be able to bid on them, and potentially win them!

And if you don't make it into the first drop, don't worry! Any NFT you mint always gets shown to all of your followers, it's visible on your profile, and it can even make the global feed. Additionally, we plan on doing a new drop of NFTs every week on bitclout.com so that new artists are continuously featured, and we hope to make this even more frequent in the future. Lastly, don't forget that new nodes are spinning up every day that offer superior experiences to bitclout.com, and your NFTs will automatically be available in all of these new nodes as well.

## Third-Party Apps

The best part about BitClout NFTs is that they are totally and 100% open, just like everything else on BitClout. This means that not only can third-party devs build custom experiences around NFTs, but they can also integrate NFTs seamlessly into their existing products. **It also leaves the door wide open for established NFT platforms like OpenSea, Rarible, and others to integrate BitClout NFTs as an added source of inventory.**

## A Note on Naming

We discussed and deliberated for a long time about whether we should call this new product "BitClout NFTs" or something else. The main issue is that very few people understand what the term "NFT" means, and even people in crypto struggle with it. Not only that, but it could come off as "nerdy," and alienate a mainstream audience. As such, we considered names like "Gems" or "Crystals," but these names had the drawback that **nobody** would understand them out of the gate.

With all of the above in mind, the term "NFT" seemed like the best choice mainly because it hooks into a hot topic that a lot of people across multiple industries are interested in, and want to learn more about. It is also a category that is drawing investment from major players such as Sotheby's and Christie's, and so even though it's a little-understood concept today, it is trending toward becoming a household term in much the same way terms like "the internet" or "website" were in the early dot-com era.

Lastly, the best part about BitClout is that anyone can improve the branding by launching a third-party app!

# Specification

Normally, major product releases like NFTs should have a very thorough specification
in their CIP. However, because our work on NFTs pre-dates the CIPs repo, we are going
to take a shortcut and simply link to the relevant PRs here:

* [frontend PR](https://github.com/bitclout/frontend/tree/rph/nifties)
* [core PR](https://github.com/bitclout/core/tree/rph/nifties)
* [backend PR](https://github.com/bitclout/backend/tree/rph/nifties)

# Backwards Compatibility

The launch of NFTs will require all nodes to upgrade to the latest version of the core
repo as soon as the first NFT is minted. Nodes that do not upgrade will not be able to
sync the latest blocks.

To ensure a smoothe transition, we are going to:

1. Disable the NFTs feature set until a certain block height
2. Merge the NFT changes into the main branch and cut a new release
3. Have everyone upgrade their nodes before the new block height hits

Please follow @diamondhands for announcements on this.

## Test Cases

See the PRs linked previously.

# Security Considerations

There are a few risks to be aware of with NFTs:

* The cashflows that go back to the creator coins will potentially break the price
computation for some coins, which has already been a minor issue. It's difficult to 
predict whether this will be bad enough to warrant rethinking how we display creator coins
and so we are taking a "wait and see" approach to this for now.
* The way NFTs are implemented, they introduce a new mechanic whereby the seller of
an NFT can spend UTXOs associated with *another user's public key.* Much care has been
taken to ensure that this does not result in any issues, but it's something to be
aware of.
