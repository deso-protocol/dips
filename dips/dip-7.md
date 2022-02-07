---
dip: 7
title: Hyper Sync: The fastest and most scalable way to download a blockchain
author: Piotr Nojszewski <DeSo: @petern | GH: @AeonSw4n >
discussions-url: N/A
created: 2022-01-29
---

## One-Line Description

We present Hyper Sync: a new approach to node synchronization designed for infinite-state blockchains that is orders of magnitude faster than traditional block synchronization. We explain two technological innovations called EllipticSum and Ancestral Records that make Hyper Sync possible.

## Explanation and Motivation

In order to run a node and be a validator in a blockchain network, one needs to first synchronize with the blockchain state. Traditionally. this means first downloading all block headers in so-called header sync, and later downloading all the corresponding blocks during a block sync. In block sync in particular, the node processes all historical transactions, which describe the "state" of the blockchain, i.e. the records that are present in the node's database. When syncing blocks, we only care about the final state, which is the database after applying all the existing blocks. However, because we're processing all historical transactions during block sync, we will often modify the same db record multiple times, despite the fact that we only care about the final value. This is inefficient.

State sync comes to rescue. In this sync method, we would download the copy of the database, called the snapshot, at some recent block height, like yesterday or a week ago. Once done, we would only sync blocks from the snapshot height, as opposed to from the genesis block. Intuitively, and in practice, this process is significantly faster, orders of magnitude faster, than block sync. 

This is important. In order for a blockchain to be truly decentralized, involving thousands or more of participating nodes, the barrier to entry for new participants needs to be as low as possible. In particular, we believe one of such barriers to be the hardware and time requirements for running and syncing a node. To minimize these requirements, we designed Hyper Sync, which is a result of months of cryptographic and algorithmic research, yielding a significant improvement on the existing state-of-the-art blockchain state sync approaches, which turned out to be ill-suited for infinite-state blockchains like DeSo.

## How Hyper Sync Works

Hyper Sync involves the following components:
1. Snapshot creation and maintenance
2. Network messages for sending and receiving state data
3. Verifying integrity of snapshot data

### Ancestral Records - Snapshot creation and maintenance 

Many existing snapshot algorithms such as Ethereum's Warp/Snap/Fast sync or Solana's Snapshot sync work great for finite-state blockchains that consist of a few gigabytes of state data. In such architectures, we can simply clone the entire blockchain state and use this snapshot copy whenever sending data to syncing peers. However, infinite-state blockchains such as the DeSo blockchain can grow to terabytes, or even petabytes of overall state data. Cloning such a database on a regular basis would be an unacceptable computational overhead. Instead, Hyper Sync builds a data structure called Ancestral Records, which only keeps track of the minimal data required to serve the snapshot. Ancestral Records keep track of the historical data that was modified during the current snapshot period. 

For example, let's say we want to create a new snapshot every 1000 blocks, and we're currently at height 5000. Ancestral Records will first start at height 5000 as an empty database. Let's say we added the block 5001, which resulted in an update of a `(key, value_at_5000)` record into `(key, value_at_5001)` (Note that the DeSo blockchain uses a KV database to store the blockchain state). Hyper Sync would store the `(key, value_at_5000)` record in the Ancestral Records. If now a syncing node wanted to request state from the snapshot provider, we can reconstruct the state at block 5000, by combining the Ancestral Records with the main DB entries, in particular we would send `(key, value_at_5000)` from the Ancestral Records instead of the `(key, value_at_5001)` that is present in the main DB. Now, let's say we connect block 5002, which further modifies `(key, value_at_5001)` into `(key, value_at_5002)`. Ancestral Records will disregard this change, because we already know the true historical value for `key` at snapshot height 5000, which is `(key, value_at_5000)`.

Aside from keeping track of modified records, Ancestral Records need to store information about records that didn't exist in the database at snapshot height. If, for instance, there was no record in the main DB at height 5000 under `key_new` but we added `(key_new, value_new)` during the snapshot epoch (the period between blockheight 5000-6000), ancestral records will store a special record for the `key_new` to indicate that this record must be excluded while serving the snapshot. As for deleting entries from the main DB, it would be stored similarly to how we handled record updates.

We also make a small optimization that reduces the number of database reads when fetching the historical entries like `(key, value_at_5000)` during modifications to the main DB records. Namely, we use an in-memory LRU cache that we update anytime we read/write to the main DB.

#### State data

All records in the DeSo KV database are indexed by prefixes, which are single-byte prefixes that indicate the type of the record. Not all data in node's database is considered state data. For instance, blocks and headers are omitted when building ancestral records. Part of the hyper sync code change is to distinguish which prefixes are classified as state and which as non-state prefixes. State records are primarily the records that are modified during DeSo transactions. Only state prefixes will be downloaded during hyper sync.

### Network messages 

We add two new peer messages to the DeSo network protocol, which are `MsgTypeGetSnapshot` with `MsgType = 17` and `MsgTypeSnapshotData` with `MsgType = 18`. These messages follow a similar pattern to the header and block messages. The syncing node will first fetch all headers as usual. Once done, the syncing node will attempt switching to state sync by first checking if it's connected to any hyper sync peers. If there are hyper sync peers, then it will send the `MsgTypeGetSnapshot` message to these peers, indicating the state prefix that it wants to sync and the last received key within that prefix. We support a multi-peer sync, where more than one peer will be sending snapshot chunks to the syncing peer. This is managed by the `server.HyperSyncProgress` object, which assigns a state prefix to each peer for syncing. Once a state prefix is completed, the `server.HyperSyncProgress` will assign a new prefix to the peer. When a node receives a `MsgTypeGetSnapshot` message, it will fetch a snapshot chunk by combining its main DB with ancestral DB. The resulting chunk will have a size of up to 16MB and will be an array of <key, value> pairs, sorted increasingly by keys. This state chunk is then sent in a `MsgTypeSnapshotData` to the syncing peer. The message will also indicate whether there are more records under the prefix in node's db, or if the prefix has been exhausted. Once the syncing node completes all prefixes, it will start syncing blocks from the snapshot height, until it arrives at a fully synced state. 
 
### EllipticSum: Verifying integrity of snapshot data

#### Introduction
Perhaps one of the most exciting parts about hyper sync is verifying that the state data received from peers is a valid state chunk. Note that the syncing peer has no knowledge of the blockchain data aside from the header chain when it proceeds to state sync. It follows that we need some sort of cryptographic mechanism that allows verifying data integrity of the received snapshot chunks. Traditionally, this has been done by building a Merkle tree of all state records and the syncing node would be rebuilding this Merkle tree with the received records. The root hash of the tree would then be included in the header chain, and the syncing node can match their local tree root hash with the hash included in the header chain. However, this solution can only work for blockchains with limited amount of state data, as each record would require at least 64 bytes of additional data to store the hashes in the Merkle tree (Assuming each hash has 32 bytes, we have 64 bytes = 32 bytes for the record hash + 32 bytes for the tree layers), not including the indexes that represent the hash position in the tree. For instance on Ethereum this currently amounts to about [**20-25gb additional storage!**](https://blog.ethereum.org/2021/03/03/geth-v1-10-0/) In terms, Merkle trees in snapshots simply do not work for infinite-state blockchains like DeSo.

Seeing the inefficiency of using Merkle trees for verifying snapshot integrity, Solana tried a different approach. Instead of hashing all the data into a Merkle tree, the data [was padded to 440 byte string using a PRNG and then XORed together](https://github.com/solana-labs/solana/blob/master/docs/src/implemented-proposals/snapshot-verification.md). This approach was interesting for a couple of reasons, which for convenience we will refer to as **checksum properties**:
1. The XOR operation is associative and commutative so `(stateChecksum ^ recordA) ^ recordB = (state ^ recordB) ^ recordA`, which means we don't have to worry about order of operations.
2. Finding inverses is trivial because for all `X` we have `X ^ X = 0`. This allows easy state modifications because if we want to update a record from `recordOld -> recordNew` and `stateChecksum` already includes `recordOld`, we can simply do `stateChecksum ^ recordOld ^ recordNew` to get the correct checksum for `recordNew`. (Note that the `recordOld` previously present in the `stateChecksum` would cancel out)
3. Low storage and computational overhead. We just need to store a single 440 bytes checksum to verify integrity of state data.

It seemed that Solana's XOR was the optimal solution for verifying data integrity. However, it transpired that it was also vulnerable to a critical exploit that made it possible to break this scheme with just under a few minutes of computation, as [discovered by the NEAR Protocol](https://reading.supply/@alexatnear/how-i-broke-solanas-state-hash-6r51RR). Namely, given an XOR checksum X, we are able to quickly find arbitrary records `r1, r2, ..., rn` such that `X ^ r1 ^ r2 ^ ... ^ rn = X`. This exploit allows passing arbitrary records to syncing peers, so for instance an attacker would have been able to convince syncing nodes that he owned millions of SOL.

Solana's approach to state data integrity was interesting because it attempted to base its security off of a different computational problem than the hash collision problem that the security of Merkle trees is founded on. Namely, it was based on the [k-sum problem](https://link.springer.com/content/pdf/10.1007%2F3-540-45708-9_19.pdf), which states that given a group `G` with operation `+`, the identity element `0`, and `k` lists `Li` of uniformly random and independently drawn elements from `G`, we want to choose an element `xi` from each of the `k` lists so that the operation `x1 + x2 + ... xk = 0`. The formulation of the problem might appear slightly counter-intuitive, but solving this problem is essentially equivalent to saying: given a set of random elements from `G`, I want to determine a subset of these elements that adds up to `0`. In the case of Solana, the choice of the group was the Galois field `GF(2 ** n, ^)` which unfortunately has a simple polynomial dynamic solution to the k-sum problem. As a result, Solana switched to Merkle trees.

In his paper, Wagner found a lower bound on the computational complexity of algorithms solving the k-sum problem for cyclic groups to be `O(sqrt(p))` where `p` is the smallest prime dividing the order of the group `G`. In addition, the discrete logarithm problem has to be difficult in `G`, so groups like `(Z/nZ, +)` don't satisfy this lower bound. In case of `GF(2 ** n)` this prime is `2`; therefore, the lower bound did not apply to that particular group. 

#### EllipticSum Construction

Having described the theoretical background in the previous section, we are now able to proceed to our EllipticSum construction. The security of the EllipticSum construction is based on the k-sum problem in the elliptic curve group [Ristretto255](https://datatracker.ietf.org/doc/html/draft-hdevalence-cfrg-ristretto-01) first [described by M. Hamburg](https://www.shiftleft.org/papers/decaf/decaf.pdf) and constructed from D. J. Bernstein's [curve25519](https://cr.yp.to/ecdh/curve25519-20060209.pdf). It gives about 128 bits of security (126 to be exact) according to Wagner's lower bound, while maintaining high efficiency comparable to the XOR construction. We chose elliptic curves over `(Z/pZ)^*` groups because of the more efficient group operations. And we chose the Ristretto255 group because it transpired to be the fastest group for our use-case. In turns, the state checksum is represented as a point on the Ristretto255 curve. Now let's describe the details of EllipticSum:

**Step 1.** Given a state record `R = <key, value>`, we first encode the record as:
```
encode(key, value) = len(key + value) || len(key) || key || len(value) || value
```
Which is similar to the DER encoding of ECDSA signatures, and it guarantees a distinct encoding. That is for any `<key', value'>`, we have `encode(key, value) = encode(key', value')` iff `key = key'` and `value = value'`.

**Step 2.** We map the encoded `<key, value>` to a point on the Ristretto255 group following the [`hash_to_curve` construction](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-hash-to-curve-13) for hashing arbitrary strings to elliptic curve points without revealing discrete logarithms of the output points. The hash we use is based on the [Elligator2 mapping](https://dl.acm.org/doi/pdf/10.1145/2508859.2516734). We denote the hashed point as:
```
hashedR = hash_to_curve(encode(key, value), ristretto255)
```

**Step 3.** To add the `hashedR` to the `stateChecksum` we simply compute `stateChecksum += hashedR` according to the Ristretto255 group operation.

It is worth noting that the EllipticSum trivially fulfills the **checksum property** 1. that we previously outlined. Property 2. is also straightforward, as the inverse to an elliptic point `P = (x, y)` is simply `P = (x, -y)` which is fast to compute. As for property 3. the ristretto255 curve checksum requires 33 bytes of storage and the additional structures we store in hyper sync do not exceed a kilobyte. We found that most modern processors can compute over 100,000 hash to curve operations on a single thread. Hyper sync computes the state checksum by running `hash_to_curve` in parallel, which makes it extremely efficient.

#### Verifying state integrity

EllipticSum checksum computation will occur whenever records are flushed to the main DB. We will keep track of partial checksums on a per-prefix basis, so that syncing nodes can immediately verify the integrity of data from data received on a prefix. This list of ec points will be sent to syncing peers. The entire state can be represented in a compressed format as a single hash of the prefix-ordered checksum ec points, and can be put in block headers for increased security.


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
* New TxnTypes
    - TxnTypeDAOCoin, similar to TxnTypeCreatorCoin
    - TxnTypeDAOCoinTransfer, similar to TxnTypeCreatorCoinTransfer
* Add new [CoinEntry](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_types.go#L777) to profile:
    - Call it DAOCoinEntry and add it [here](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_types.go#L850)
    - This will be a second instance of CoinEntry with the exact same fields
* Add a new field to the CoinEntry:
    - Define a type TransferRestrictionStatus with the following values:
        * Unrestricted = 0 // default
        * ProfileOwnerOnly = 1 // Transfers to/from the profile owner are the only transfers allowed when this is set
        * ExistingDAOMembersOnly = 2 // Same as ProfileOwnerOnly, except that transfers among existing DAO coin holders are unrestricted
        * PermanentlyUnrestricted = 3
    - Set a field in the CoinEntry with the type of TransferRestrictionStatus
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
        * ModifyTransferRestriction: With this operation, the user can change the TransferRestrictionStatus of the DAO Coin. Below are the options:
            - Set TransferRestrictionStatus = 0
                * Can only be triggered if TransferRestrictionStatus != 3
            - Set TransferRestrictionStatus = 1
                * Can only be triggered if TransferRestrictionStatus != 3
            - Set TransferRestrictionStatus = 2
                * Can only be triggered if TransferRestrictionStatus != 3
            - Set TransferRestrictionStatus = 3
                * Can be triggered at any time. Irreversible once triggered.
* \_connectDAOCoinTransfer
    - This will be virtually identical to [\_connectCreatorCoinTransfer](https://github.com/deso-protocol/core/blob/5764fb519e19a0e44487cd592bee98cbaade6f71/lib/block_view_creator_coin.go#L1463), only it will act
      on the DAOCoin balances in the db, rather than on the CreatorCoin balances
    - Additionally, when transfer restriction is turned on, we only allow
      transfers to/from the profile owner. This is a bit of a hack, but it fully solves for the
      use-case of wanting a profile owner to approve transfers between other parties. To implement
      an approval UI, a smart service would simply need a derived key from the transferrer with
      permission to execute a transfer once the owner approves. The smart service can then execute
      the transfer from the transferrer to the owner to the transferee as an atomic unit only after
      the owner approves the transfer in the UI. To
      implement this constraint in consensus, the \_connectDAOCoinTransfer function should
      error immediately unless the following condition is met:
        * profileOwnerIsInTransfer := (profile owner is recipient) || (profile owner is sender)
        * daoMemberIsRecipient := (recipient has a non-zero balance of the DAO coin being transferred)
        * if (TransferRestrictionStatus == TransferRestrictionStatusProfileOwnerOnly) && !profileOwnerIsInTransfer {
            - error
        * } else if (TransferRestrictionStatus == TransferRestrictionStatusDAOMembersOnly) && !profileOwnerIsInTransfer && !daoMemberIsRecipient {
            - error
        * }

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
