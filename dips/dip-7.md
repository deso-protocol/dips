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

In order to run a node and be a validator in a blockchain network, one needs to first synchronize with the blockchain state. Traditionally, this means first downloading all block headers in so-called header sync, and later downloading all the corresponding blocks during a block sync. When syncing blocks, we go block by block from genesis up to the blockchain tip, modifying the database with each block. The node processes all historical transactions, which describe the "state" of the blockchain, i.e. the records that are present in the node's database. After applying all the existing blocks, we will arrive at the current state, which is when the node is ready to functionally participate in the consensus. However, because we're processing all historical transactions during block sync, the blocks often overlap with their modifications, updating the same database records multiple times. This is actually inefficient, because we only really care about the final values of each database records.

State sync comes to rescue. In this sync method, we would avoid the duplicate record updates by directly downloading a copy of the database, called the snapshot. The snapshot represents the state of the blockchain at some recent block height, like yesterday, or a week ago. Once done, we would only sync blocks from the snapshot height, as opposed to from the genesis block. Intuitively, and in practice, this process is significantly faster, orders of magnitude faster, than block sync. 

This is important because in order for a blockchain to be truly decentralized, involving thousands or more of participating nodes, the barrier to entry for new participants needs to be as low as possible. In particular, we believe one of such barriers to be the hardware and time requirements for running and syncing a node. To minimize these requirements, we decided to add state sync into the DeSo node toolset; however, we quickly found out that the existing state sync approaches turned out to be costly and unscalable for infinite-state blockchains like DeSo. In response, we designed Hyper Sync, which is a result of months of cryptographic and distributed systems research, yielding a significant improvement to the existing state-of-the-art blockchain state sync approaches.

## How Hyper Sync Works

On a high-level, Hyper Sync involves three main components:
1. Snapshot creation and maintenance
2. Peer messages for requesting and sending snapshot data
3. Computing a checksum to verify integrity of snapshot data

It's also worth mentioning that Hyper Sync has two sides:
- A syncing node
- A fully synced node

We will reference these two scenarios when describing individual components.

### 1. Ancestral Records - Snapshot creation and maintenance 

Snapshots are created and maintained by the fully synced nodes. So why do we even need to snapshot the database? The reason for taking this snapshot is so that we send immutable data to the syncing peer -- if we were instead to send him current blockchain state, each newly added block would possibly conflict with the already sent data. And how do we make a snapshot? Many existing snapshot strategies such as Ethereum's Warp/Snap/Fast sync or Solana's Snapshot sync work great for finite-state blockchains that consist of a few gigabytes of state data. In such architectures, creating a snapshot can be simply done by periodically cloning the entire blockchain state. For example the snapshot period could be 1000 blocks, meaning we make the snapshot at heights 0, 1000, 2000, 3000, etc. This snapshot copy would then be served when sending data to syncing peers. However, infinite-state blockchains such as the DeSo blockchain can grow to terabytes of overall state data. Cloning such a database on a regular basis would be an unacceptable computational overhead. 

Hyper Sync employs a different strategy to snapshot creation. To avoid cloning the state, we build a data structure called Ancestral Records, which only stores the minimal amount of data required to serve the snapshot. Ancestral Records save us space and computation, which is balanced throughout the snapshot epoch - the block heights in-between snapshot heights. The Ancestral Records keep track of the historical values of database records that were modified during the current snapshot epoch. 

To explain how Ancestral Records work, let's walk through an example. Consider again the snapshot period of 1000 blocks, and that the current blockchain tip is at height 5000. Ancestral Records would start as a fresh, empty database at height 5000. Now, we mined block 5001, which contained a transaction that resulted in an update of an existing record `("my_prime", 127)` into a record `("my_prime", 8191)`, i.e. the `"my_prime"` record existed at block 5000 with value `127` and block 5001 modified it to `8191` (Note that the DeSo blockchain uses a key-value database to store the blockchain state). Now, Hyper Sync would store the `("my_prime", 127)` record in the Ancestral Records. Furthermore, let's say we connect block 5002, which further modifies `("my_prime", 8191)` into `("my_prime", 131071)`. Ancestral Records will disregard this change, because we already know the true historical value for `"my_prime"` at snapshot height 5000 to be `("my_prime", 127)`. If now a syncing node wanted to request state from the snapshot provider, we can reconstruct the snapshot of the state at block 5000 by combining the Ancestral Records with the Main Database entries. In particular, we would send `("my_prime", 127)` from the Ancestral Records instead of the `("my_prime", 131071)` that is present in the Main Database.

Aside from keeping track of modified records, Ancestral Records need to store information about records that didn't exist in the database at snapshot height. If, for instance, there was no record in the Main Database at height 5000 under `"my_perfect"` but we added `("my_perfect", 8128)` during the snapshot epoch, Ancestral Records will store a special record for the `"my_perfect"` key to indicate that this record must be excluded while serving the snapshot. That is, we would not send `"my_perfect"` record to the syncing peer because it didn't exist at height 5000. As for removing entries from the Main Database, deletes are handled identically to updates of existing records.

We also make a small optimization that reduces the number of database reads when fetching the historical entries like `("my_prime", 127)` during modifications to the Main Database records. Namely, we use an in-memory [LRU cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) that we update anytime we read/write to the Main Database.

#### State data

All keys of the records in the DeSo key-value database contain single-byte prefixes that indicate the type of the record. These prefixes are always the first byte of the record's key. The comprehensive list of all database prefixes can be found [here](https://github.com/deso-protocol/core/blob/1b9d4d61499d66f3c3f2911496785aa6b95dd5d5/lib/db_utils.go#L38). 

Some prefixes store data that's not considered part of the state. For instance, prefixes that store blocks and headers don't constitute the state and are hence omitted when building Ancestral Records. The reason being that these records mostly serve archival purposes, and in fact aren't required for the node to be fully synced and correctly validating future blocks. Part of the hyper sync code change is to distinguish which prefixes are classified as state and which as non-state prefixes. State records are primarily those records that are modified during DeSo transactions. To save bandwidth and time only state prefixes will be downloaded during Hyper Sync. However, after a node executes Hyper Sync, it will still download all non-state records in the background, after it has fully synced with the blockchain.

### 2. Peer messages for requesting and sending snapshot data

Nodes in the DeSo blockchain communicate through peer messages. The current list of these messages can be found [here](https://github.com/deso-protocol/core/blob/1b9d4d61499d66f3c3f2911496785aa6b95dd5d5/lib/server.go#L1489). In simple terms, peer messages are specially encoded network packets sent between peers to indicate certain actions such as asking for headers, blocks, or sending headers, blocks, etc.  We add two new peer messages to the DeSo network protocol, which are `MsgTypeGetSnapshot` with `MsgType = 17` and `MsgTypeSnapshotData` with `MsgType = 18`. These messages follow a similar pattern to the header and block messages. 

A node executing Hyper Sync will first request all blockchain headers from its peers. Once done, the syncing node will attempt switching to state sync by first checking if it's connected to any hyper sync peers. If there are Hyper Sync peers, then it will send the `MsgTypeGetSnapshot` message to these peers, indicating the state prefix that it wants to sync and the last received key within that prefix. Upon receiving the `MsgTypeGetSnapshot` message, the node will scan its Main Database and Ancestral Records and respond with a `MsgTypeSnapshotData` message containing an appropriate chunk of its most recent snapshot. The chunk will have a size of up to 16MB and will be an array of `<key, value>` pairs, sorted lexicographically by keys. The `MsgTypeSnapshotData` will also indicate whether there are more records under the prefix in node's database, or if the prefix has been exhausted.

#### Multi-Peer Sync
We support a multi-peer sync, where more than one peer will be sending snapshot chunks to the syncing peer. This approach is much more efficient and speeds up the syncing process even further. The multi-peer sync is managed by the `HyperSyncProgress` object, where the syncing node assigns a state prefix to each snapshot peer for syncing. We decided to split the state records by prefixes, because it's an easy up-front way for the syncing node to divide the work needed to sync the state. It also has low computation overhead for verifying the integrity of the state data. In particular, in this approach it is easy to determine which nodes might be sending invalid snapshot data, but we'll return to this issue in the next section on EllipticSum. Once a snapshot peer sends all records to the syncing node within its assigned state prefix, the `HyperSyncProgress` will assign a new prefix to the snapshot peer. Once the syncing node completes all prefixes, it will start syncing blocks from the snapshot height, until it arrives at a fully synced state. Afterwards, it will sync the remaining blocks in the background for archival purposes.

#### Eliminating stragglers
The multi-peer sync is significantly faster than the single-peer sync; however, it comes with its pitfalls. In particular, it is more likely that during the sync process a connected snapshot peer might experience downtime, network drops, or just generally be slow. If unhandled, this could jeopardize the efficiency of hyper sync, especially if the straggling peer was assigned a prefix containing a lot of data. To solve this, we re-assign database prefixes when all the remaining prefixes have been completed, or are in-progress. The straggling peers would be removed and faster peers would continue the syncing work on that prefix. To be specific, we measure the "speed" of the snapshot peers by the amount of data that we've received from them. With this approach, a few straggling peers have a minuscule impact on the overall progress of Hyper Sync.
 
### 3. EllipticSum: Computing a checksum to verify integrity of snapshot data
*Disclaimer: This section contains high levels of cryptography and abstract algebra. Read at your own risk.*

#### Introduction
Perhaps one of the most exciting parts about Hyper Sync is verifying that the state data received from peers is a valid state chunk. Note that the syncing peer has no knowledge of the blockchain data aside from the header chain that it downloaded prior to the state sync. It follows that we need some sort of cryptographic mechanism that allows verifying data integrity of the received snapshot chunks. Just as a thought experiment, consider what would have happened if we didn't properly verify the validity of the snapshot data received from the peers. A malicious node could basically send us arbitrary state - e.g. say that they have millions of DeSo, and we wouldn't know if that's true or not. So how do we prevent this?

Traditionally, this has been done by building a Merkle tree of all state records and the syncing node would be rebuilding this Merkle tree with the received records. The root hash of the tree, also referred to as **state checksum**, would then be included in the header chain, and the syncing node can match their local tree root hash with the hash included in the header chain. However, this solution can only work for blockchains with limited amount of state data, as each record would require at least 64 bytes of additional data to store the hashes in the Merkle tree (64 bytes = 32 bytes for the record hash + 32 bytes for the tree layers), not including the indexes that represent the hash position in the tree. For instance on Ethereum this currently amounts to about [**20-25gb additional storage!**](https://blog.ethereum.org/2021/03/03/geth-v1-10-0/) It follows that Merkle trees simply do not work for snapshotting infinite-state blockchains like DeSo.

Seeing the inefficiency of using Merkle trees for verifying snapshot integrity, Solana tried a different approach. Instead of hashing all the data into a Merkle tree, the data [was padded to 440 byte string using a PRNG and then XORed together](https://github.com/solana-labs/solana/blob/master/docs/src/implemented-proposals/snapshot-verification.md). This approach was interesting for a couple of reasons, which for convenience we will refer to as **state checksum properties**:
1. **The checksum operation is associative and commutative.**
   1. In particular, `(stateChecksum ^ recordA) ^ recordB = (stateChecksum ^ recordB) ^ recordA`, which means we don't have to worry about the order of operations.
2. **Adding and removing elements is trivial.**
   1. Adding elements is straightforward and so is removing. Finding inverses for XOR is trivial because for all `X` we have `X ^ X = 0`. This allows for easy state modifications. For instance, if we want to update a record from `recordOld` to `recordNew` and we have `stateChecksum = stateChecksumOld ^ recordOld` already includes `recordOld`, we can simply do `stateChecksum ^ recordOld ^ recordNew = (stateChecksumOld ^ recordOld) ^ recordOld ^ recordNew = stateChecksumOld ^ recordNew` to get the correct checksum for the modification.
3. **Low storage and computational overhead.** 
   1. We just need to store a single 440 bytes checksum to verify integrity of state data, and XOR computation is extremely fast.

It seemed that Solana's XOR was the optimal solution for verifying data integrity. So what went wrong? It transpired that the XOR checksum was vulnerable to a critical exploit that made it possible to break the scheme with [just under a few minutes of computation](https://reading.supply/@alexatnear/how-i-broke-solanas-state-hash-6r51RR). Namely, given an XOR checksum X, we are able to quickly find arbitrary records `r1, r2, ..., rn` such that `X ^ r1 ^ r2 ^ ... ^ rn = X`. This exploit essentially allowed for passing arbitrary records to syncing peers, and as a result Solana had to switch to Merkle trees.

Nonetheless, we found Solana's approach to computing a state checksum interesting and in fact it served as one of the inspirations for EllipticSum. In order to explain our checksum function, let us reflect on why exactly the XOR checksum didn't work. The XOR checksum was based on the [k-sum problem](https://link.springer.com/content/pdf/10.1007%2F3-540-45708-9_19.pdf), as described in the paper by D. Wagner, which states that given a group `G` with operation `+`, the identity element `0`, and given a multiset (set with repetitions) of uniformly random and independently drawn elements from `G`, we want to find elements `x1, x2, ..., xk` such that `x1 + x2 + ... xk = 0` under the group operation. To be precise, Wagner defined the k-sum problem in a different manner, and we redefined it for the sake of brevity. We recommend reading the [paper](https://link.springer.com/content/pdf/10.1007%2F3-540-45708-9_19.pdf) for a more thorough analysis.

In his paper, Wagner found a lower bound on the computational complexity of algorithms solving the k-sum problem for cyclic groups to be `O(p ** 1/2)` where `p` is the smallest prime dividing the order of the group `G` (the lower bound is really a combined result of the work of V. Shoup, V. I. Nechaev, and W. Dai). In addition, the discrete logarithm problem has to be difficult in `G`, so groups like `(Z/nZ, +)` don't satisfy this lower bound. In the case of Solana, the choice of the group was the Galois field `GF(2 ** n, ^)`. The order of `GF(2 ** n)` is simply divisible by `2`; therefore, Wagner's lower bound did not apply to that particular group. In turns, the polynomial dynamic solution to the k-sum problem was found.

#### EllipticSum Construction

Having described the theoretical background in the previous section, we are now able to proceed to our EllipticSum construction. The security of the EllipticSum construction is based on the k-sum problem in the elliptic curve group [Ristretto255](https://datatracker.ietf.org/doc/html/draft-hdevalence-cfrg-ristretto-01) first [described by M. Hamburg](https://www.shiftleft.org/papers/decaf/decaf.pdf) and constructed from [D. J. Bernstein's curve25519](https://cr.yp.to/ecdh/curve25519-20060209.pdf). The Ristretto255 group is cyclic, has a difficult DLP, gives about 128 bits of security (126 to be exact) according to Wagner's lower bound, while maintaining high efficiency that's comparable to the XOR construction. We chose elliptic curves over `(Z/pZ)^*` groups because of the more efficient group operations. And we chose the Ristretto255 group because it transpired to be the fastest for our use-case. In turns, the state checksum is represented as a point on the Ristretto255 curve. The most crucial part of the EllipticSum construction is how database records are mapped onto the curve. It is important that discrete logarithms of the points corresponding to these records are not revealed, otherwise we could be prone to pre-image attacks. To see how we solve these issues, let us describe the details of EllipticSum protocol:

**Step 1.** Given a state record `R = <key, value>`, we first encode the record as:
```
encode(key, value) = len(key + value) || len(key) || key || len(value) || value
```
Which is similar to the DER encoding of ECDSA signatures, and it guarantees a distinct encoding. That is for any `<key', value'>`, we have `encode(key, value) = encode(key', value')` iff `key = key'` and `value = value'`.

**Step 2.** We map the encoded `<key, value>` to a point on the Ristretto255 group following the recent cryptographic construction called [`hash_to_curve` construction](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-hash-to-curve-13) for hashing arbitrary strings to elliptic curve points without revealing discrete logarithms of the output points. The hash we use is based on the [Elligator2 mapping](https://dl.acm.org/doi/pdf/10.1145/2508859.2516734). We denote the hashed point as:
```
hashedR = hash_to_curve(encode(key, value), ristretto255)
```

**Step 3.** To add the `hashedR` to the `stateChecksum` by simply computing `stateChecksum += hashedR` according to the Ristretto255 group operation.

It is worth noting that the EllipticSum trivially fulfills the **checksum property 1.** that we previously outlined. **Property 2.** is also straightforward, as the inverse to an elliptic point `P = (x, y)` is simply `P = (x, -y)` which is fast to compute. As for **property 3.** the Ristretto255 curve checksum requires 33 bytes of storage and the additional structures we store in Hyper Sync do not exceed a kilobyte. We found that most modern processors can compute over 100,000 hash to curve operations per second on a single thread. Hyper Sync computes the state checksum by running `hash_to_curve` in parallel, which makes it extremely efficient.

#### Verifying state integrity

EllipticSum checksum computation will occur whenever records are flushed to the Main Database. We will keep track of partial checksums on a per-prefix basis, so that syncing nodes can immediately verify the integrity of data from data received on a prefix. This list of ec points will be sent to syncing peers. The entire state can be represented in a compressed format as a single hash of the prefix-ordered checksum ec points, and can be put in block headers for increased security.

### Code Changes
The core code can be found in [this PR](https://github.com/deso-protocol/core/pull/200).

#### Snapshot

The `Snapshot` struct will handle the `AncestralRecordsDb`, `DatabaseCache`, and `Checksum`. The Snapshot struct will have its own thread that computes actions based on messages received to a channel called `OperationChannel`. Other threads can enqueue messages to the `OperationChannel` if they want to communicate with the `Snapshot`. These actions include:
   - `SnapshotOperationFlush` which requests a flush to the ancestral records.
     - To lower the computational overhead on the node's main block-processing thread, flushing to the ancestral records happens in the snapshot thread. Ancestral Records are stored in a separate badger DB and follow a 2-phase prepare-commit pattern. When flushing to the Main Database, the flushing thread first calls `Snapshot.PrepareAncestralRecordsFlush` to start the prepare phase. This will initialize a `AncestralCache` object, used to store the modified records, and add it to the `AncestralMemory`. 
     - `AncestralCache` contains two maps: `ExistingRecordsMap` and `NonExistingRecordsMap` that store the historical values of entries that were modified during flush. The `ExistingRecordsMap` stores `<key, value>` pairs of records that already existed in the Main Database. While, the `NonExistingRecordsMap` stores `<key, 0>` pairs to indicate that a new record was added to the Main Database at `key`. To slightly speed-up the flushing process, `AncestralCache` also stores a list `RecordsKeyList` that just stores the combined set of keys among both `ExistingRecordsMap` and `NonExistingRecordsMap`.
     - `AncestralMemory` is a concurrency-safe bi-directional queue (deque) that allows us to pop `AncestralCache` objects from one end and push to the other end. After the Main Database flush is completed, we will signal the second, commit phase by enqueuing the `SnapshotOperationFlush` to `OperationChannel` to indicate that the Snapshot thread can remove a `AncestralMemory` object from the beginning of the queue and flush it to Ancestral Records.
     - The flush to the Ancestral Records will pop the first element of the `AncestralMemory` queue and add all of its entries to the Ancestral Records database. Based on if a particular record existed in the `ExistingRecordsMap` or `NonExistingRecordsMap`, the `<key, value>` record will be stored as follows in the database (note that non-existing records will have an empty `value`): 
     ```
     <_prefixAncestralRecord [1]byte, SnapshotBlockHeight [8]byte, key []byte> ->
        <value []byte, existence_byte [1]byte>
     ```
     - The `_prefixAncestralRecord = []byte{0}` and the `SnapshotBlockHeight` is just the block at which we've taken the snapshot. Given an ancestral record, we can easily retrieve the original Main Database record by slicing the first 9 bytes off of the key, and popping the last byte in the value.
     - While the above pattern splits the computation among the main and snapshot threads, there is an important concurrency problem we must address. Namely, what would happen if a syncing peer requested a chunk of the database right when we're flushing to either of the databases? We will fix this with non-blocking semaphore counters: `MainDBSemaphore` and `AncestralDBSemaphore`. Both counters start at `0` and they will fulfill the following invariant: **when `MainDBSemaphore` and `AncestralDBSemaphore` are non-equal then the database is being modified**. Whenever a flush takes place, in the prepare phase we increment the `MainDBSemaphore` by `+1`. Note that the semaphores are non-equal because the flush to ancestral records hasn't been processed yet. Then, in the commit phase we will atomically increment both `MainDBSemaphore` and `AncestralDBSemaphore` by `+1`. The semaphores are still non-equal because the Ancestral Records flush is just starting. When we finish the Ancestral Records flush, we increment the `AncestralDBSemaphore` by `+1`. Finally, the semaphores are equal because we finished both Main Database and Ancestral Records flush. We leave the proof of why the invariant will always hold regardless of concurrency to the reader. Now, returning to our original problem, we will respond to the syncing peer with a database chunk only when the `MainDBSemaphore` and `AncestralDBSemaphore` counters are equal.
   - `SnapshotOperationProcessBlock` which signals that a new block has been processed.
     - This operation will pass the newly added `BlockNode` to the Snapshot structure. The Snapshot will determine whether a new Snapshot epoch should start or not. The new epoch starts whenever the `BlockNode` height is divisible by the `SnapshotBlockHeightPeriod`. In such a case, we will update the `SnapshotBlockHeight` to the new snapshot height and remove all ancestral records from the past snapshot epoch (note that we can infer the epoch of an ancestral record by looking at its key). BadgerDB makes this operation very fast by incrementing the version on a prefix and removing the records in the background. If we're processing blocks within the current epoch, we do nothing. The edge-case is block rollbacks, particularly a rollback that would span blocks within past snapshot epochs. In such case we will mark the `brokenSnapshot` flag as true, remove all ancestral records from the current epoch, and do nothing until we start processing blocks in the current epoch, where we would set `brokenSnapshot` as false.
   - `SnapshotOperationProcessChunk` which is called after a syncing node receives a snapshot chunk from a peer and wants to add it to the Main Database.
     - This operation will be passed a pointer to the Main Database and the snapshot chunk. The processing of a snapshot chunk is delegated to the snapshot thread so that the main thread can focus on requesting snapshot chunks from peers and also so that the snapshot thread can compute the checksum associated with the synced state. Processing of a chunk means adding all records in the chunk to the Main Database and computing the EllipticSum checksum for these entries.
   - `SnapshotOperationChecksumAdd` which adds bytes to the state checksum.
     - This operation is passed bytes of previously mentioned `encode(key, value)`, implemented in `EncodeKeyValue`, and adds it to the state checksum by calling the `Checksum.AddBytes`.
   - `SnapshotOperationChecksumRemove` which removes bytes from the state checksum.
     - This operation is passed bytes of previously mentioned `encode(key, value)`, implemented in `EncodeKeyValue`, and removes it from the state checksum by calling the `Checksum.RemoveBytes`.
   - `SnapshotOperationChecksumPrint` used for debugging to print the checksum bytes.


#### Database changes

1. **Database Operations**. Another crucial modification to the codebase is that the `Snapshot` pointer will be passed to all functions interacting with the Main Database. In addition, all database operations are modified to use one of the following wrappers around BadgerDB operations:
   - `DBSetWithTxn` is a wrapper around Badger's `txn.Set` and will be used for modifying database records.
     - The `DBSetWithTxn` function will be passed a `<key, value>` pair that we want to set in the database. We will first lookup the existing entry under `key` by calling `DBGetWithTxn`. If a record exists, then it'll be a candidate for an ancestral record, so we will add it to the latest `AncestralCache`. We will also add the new record to the Main Database and consequently to the `DatabaseCache`, which stores the LRU database records in-memory. Moreover, we'll add the `<key, value>` record to the state checksum. It is worth noting that if there's an exiting record in the Main Database in place of `key` then we'll first remove it from the state checksum before adding the new record.
   - `DBGetWithTxn` is a wrapper around Badger's `txn.Get` and will be used for retrieving database records.
     - The `DBGetWithTxn` will be passed a key that we want to fetch and check its existence in the `DatabaseChecksum`. If the key exists in the cache, then we'll return it; otherwise, we'll look for the entry in the Main Database. If the entry exists in the Main Database, we'll add it to the `DatabaseCache` and return it. If it doesn't exist then we'll just return the `ErrKeyNotFound` from badger.
   - `DBDeleteWithTxn` is a wrapper around Badger's `txn.Delete` and will be used for deleting database records.
     - The `DBDeleteWithTxn` works very similarly to `DBSetWithTxn` with the exception that we'll be removing the `<key, value>` entry from the Main Database and from the state checksum. It is worth noting that the delete function will also be adding records to existing database record to the `AncestralCache`.

2. **Prefixes**. We also make an important change to how the database prefixes are handled. In particular the prefixes will now be defined as part of the `DBPrefixes` struct and will have the following pattern:
```
PrefixPubKeyUtxoKey []byte `prefix_id:"[7]" is_state:"true"`
```

- The values of the prefixes will be passed in the tag in place of the `prefix_id` which supports byte-array prefixes. Although currently, most prefixes are single-byte. In addition, we add the `is_state` boolean label which determines if a record is supposed to be part of the state or not. The `is_state` tag can be omitted, in which case the record will be deemed non-state. This pattern allows for arbitrary labels for the database prefixes so we think this is a useful change considering forward-compatibility. 

3. **Encoders**. We modify how all entries are encoded in the Main Database by introducing deterministic encoders. The previous gob encoding transpired to be non-deterministic; hence, all entries will have a custom `encode` and `decode` functions that will be used when storing entries in the database.

#### Checksum

The checksum struct implements the EllipticSum and is responsible for computing the state checksum, in particular handling adding and removing bytes, exposing two functions: `AddBytes` and `RemoveBytes`. To make the checksum computation faster, these operations will follow a worker pool pattern, where we'll spawn a bounded number of threads, as needed, to handle the EllipticSum computation. The parallelization mainly affects the `hash_to_curve` routine, which computes a Legendre symbol for elements in the 2**255-19 field, and hence is the most expensive operation in EllipticSum.

It is also worth noting that we set the `dst string` for the Ristretto255-Elligator2 `hash_to_curve` as the byte array `[68 69 83 79 45 69 76 76 73 80 84 73 67 45 83 85 77 58 82 105 115 116 114 101 116 116 111 50 53 53]`

#### Network changes
As previously mentioned two new peer messages are added:
- `MsgTypeGetSnapshot` has `MsgType = 17` and is used for requesting snapshot from peers. 
  - It will contain a `Prefix` for the database prefix that the node wants to sync and the `SnapshotStartKey` byte array that indicates the smallest database key, lexicographically, that the node is requesting from the peer. The peer will scan its Main Database and Ancestral Records starting with `SnapshotStartKey` and respond with an appropriate `MsgTypeSnapshotData`.
- `MsgTypeSnapshotData` has `MsgType = 18` and is used for sending snapshot data to peers.
  - It will contain information about the Snapshot such as the `SnapshotHeight` and `SnapshotBlockHash`, the snapshot checksum stored in `SnapshotChecksum`, the snapshot data stored as a list of `<key, value>` pairs sorted within `DBEntry` structs and sorted lexicographically by keys, the `Prefix` of the snapshot data entries, and a flag `SnapshotChunkFull` that indicates whether the node has more entries to fetch within the `Prefix`. The syncing node will then process this snapshot data chunk by enqueuing a `SnapshotOperationProcessChunk` operation in the Snapshot's `OperationChannel`.     
  
Moreover, a new peer service flag is added called `SFHyperSync = 1 << 1` that indicates a peer supports Hyper Sync. 

#### Server changes

As previously noted, the Hyper Sync node will switch to state sync after completing header sync. The HyperSync progress will be managed by the `SyncProgress` structure that will assign peers to each state prefix and will oversee the completion of these prefixes. Once all prefixes have been completed, the syncing node will switch to block sync from the snapshot height. After it arrives at a fully synced status, it will download the remaining blocks from genesis up to snapshot height in the background. These blocks will not be validated, aside from verifying that they match the header chain hashes.

### Deployment
After the addition of this DIP and completion of the PR, we will deploy the first Hyper Sync nodes on the DeSo testnet and after an expected week-long QA, Hyper Sync will be rolled out on the mainnet with a two-week notice.

### Testing
As of now, testing of Hyper Sync is still in progress; however, the following strategy is employed:
- Unit testing of individual Hyper Sync components. These components include:
  - Checksum
  - Ancestral Records
  - Encoders
  - DatabaseCache
  - AncestralMemory
  - Network messages
- Integration and system testing through automated node tests. We will deploy a new testing framework that spawns multiple containerized nodes and intercepts their communication to analyze and test the behavior of individual nodes and the network as a whole. This framework is particularly exciting from forward-compatibility perspective as we expect it to be essential in the rollout of the more advanced changes such as moving to Proof of Stake.
- Acceptance testing in the DeSo testnet. After the other stages of the testing process are concluded, we will formally open Hyper Sync for the developer community, so we can ensure Hyper Sync software is of highest quality.

### Security Considerations
This update has significant impact on the security of the chain and requires exceptionally thorough testing. To ensure security is maintained in the worst-case scenario, Hyper Sync will be an optional node configuration setting, and traditional header-block sync will still be maintained as a valid syncing method.

### Acknowledgments
TODO.