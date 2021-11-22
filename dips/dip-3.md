---
dip: 3
title: Diamonds Paid In CLOUT
author: RedPartyHat <@redpartyhat>
discussions-url: https://github.com/deso-protocol/dips/discussions/26
created: 2021-08-04
---

## One-Line Description

Pay for diamonds with CLOUT instead of creator coins.

## Explanation and Motivation

"Diamonds" are a tool that allow users to show their love for a creator's work on DeSo. A user
can give up to six diamonds per post and each diamond given comes with a reward for the creator. A
single diamond gives the creator ~0.005 USD (half a penny) at the time of writing. Each additional
diamond 10x's the reward, up to ~$500 USD for six diamonds. This has already become a significant
source of income for many creators.

Currently, the reward given with each diamond is paid in the giver's creator coin. This is great
for creating a dense creator coin ownership graph. Diamond recipient's often do not sell the creator
coins that they receive, which gives them a growing financial stake in the success of all of their
supporters. However, this is not great for compensating creators and lines diamond givers up for a
dump event if creators need to cash out for raw CLOUT in order to support themselves.

Making diamonds paid in CLOUT instead of creator coins will maximize the financial rewards for
creators and simplify the platform for newcomers.

## Specification

Implementation will be similar to any forking change. A blockheight will be used to give nodes all
nodes in the network time to upgrade. After that blockheight, all diamond transactions will be paid
with CLOUT instead of creator coin.

## Data Storage Change

This change will increase the storage requirements for diamonds on the chain. Creator coins are
stored using a balance model on the chain. Therefore, in the existing state, if a diamond receiver
already holds the creator coin of the diamond giver the database will only be updated. In contrast,
CLOUT is paid out using UTXOs. Each new diamond transaction will create an entry in the database
everytime a user is given a diamond. This will be a manageable and temporary increase in storage
that will be relieved when CLOUT is moved from UTXOs to a balance model.

## Backwards Compatibility

Diamonds given before the fork will not be compatible with the consensus logic after the fork and
vice versa.

## Security Considerations

This upgrade should not significantly impact the security of the chain. Typical care should be taken
in reviewing the code to ensure that there are no "money printer" bugs in the revised transaction.

## Alternate designs considered

An alternative design would be to provide users with the choice to pay for diamonds in either CLOUT
or creator coin. A sensible default could be chosen and the user's preference could be stored on
their profile entry. This option is attractive in that it maximizes choice for users but it adds
significant complexity. For example, it would lead to a confusing state for new creators where they
sometimes receive creator coins and sometimes receive CLOUT for diamonds. In this case, it is believed
that the benefits of simplicity and standardization outweigh the benefits of flexibility.

## Acknowledgements

Big thank you to all community members that participated in the discussion of this topic!
