---
dip: 4
title: Balance Model
author: RedPartyHat <@redpartyhat>
discussions-url: https://github.com/deso-protocol/dips/discussions/113
created: 2021-11-21
---

## One-Line Description

Move the DeSo blockchain from UTXOs to a balance model.

## Explanation and Motivation

The DeSo blockchain operates using the "UTXO model" for facilitating DESO transactions. 
Under the UTXO model, a user must specify all of the UTXOs ("Unspent Transaction Outputs")
that they are using to pay for a given transaction. For example, if a user receives 1 DESO
from 10 different users, they must specify all 10 UTXOs in order to send the 10 DESO total
to another account. Each UTXO is identified by the 32-byte transaction ID from which it 
originated and a byte to specify the index it has within that transaction. Therefore,
in the example given, 330 bytes are used to specify UTXOs for the transaction.

An alternative approach to tracking UTXOs is to track balances, hence the "balance model."
Under the balance model, no UTXOs are needed.  Instead, consensus tracks a single uint64
balance for every public key and uses it as the basis for approving transactions. This
significantly reduces transaction size, the amount of information stored on disk, and 
the complexity involved in transaction construction.

## Specification

There are three key parts to moving DeSo from UTXOs to the balance model:
  (1) Replace all uses of UTXOs when connecting/disconnecting transactions.
  (2) Add a Nonce (and TxnFeeNanos) to MsgDeSoTxn to prevent replay attacks.
  (3) Ensure that serialization is not broken by the nonce 

(1) UTXO adds and spends are currently specified by four convenient funtions: addUTXO(), 
unAddUTXO(), spendUTXO, and unSpendUTXO(). In order to replace the functionality provided
by these four functions, we will introduce four analagous balance model functions:
addBalance(), unAddBalance(), spendBalance(), unSpendBalance(). Use of the balance model
functions in consensus will be restricted by the BalanceModelBlockHeight constant. This
will also allow us to create a new TestBalanceModel test to wrap all existing unit tests
and test them under balance model conditions by setting BalanceModelBlockHeight to zero.

(2) Under the UTXO model, it is impossible to replay a transaction because the UTXOs used
by the replayed transaction are spent when the transaction is first used. Under the
balance model, this would no longer be the case. A transaction that sends 1 DESO from 
A to B could be replayed for as long as A remains solvent. In order to prevent replay,
a Nonce must be added to each transaction for uniqueness.

There are several ways to implement a nonce. Ethereum uses an incrementing integer that is
tracked by public key. Transactions are only accepted under this scheme if they have the
correct next expected nonce. Solana uses a slightly different scheme that requires
transactions to include a recent blockhash in the transaction. This has the advantage
of easing transaction creation and validation since the expected nonce does not need to be
looked up for each transactor's public key. A simple map of recent blockhashes would be
stored in memory instead. The downside of this scheme is that it can cause trouble for
some high-value participants that want to perform offline signing. See Solana's discussion
of this problem and their contract-enabled solution here:
https://docs.solana.com/implemented-proposals/durable-tx-nonces. Another Nonce solution
that was considered is the use of pure randomness. This would solve the offline signing 
problem and provide easy transaction construction. Unfortunately though, this would
require us to maintain an in-memory map of all used nonces until the end of time. In light
of all these trade-offs, we plan to implement a simple Ethereum-like incrementing nonce.

In addition to adding a nonce to the transaction, we will also need to add "TxnFeeNanos."
Transaction fees are currently implicitly specified in the transaction. That is, any 
transaction inputs that are not spent in the transaction outputs are fees. Since inputs
will not be specified under the balance model, we will need to explicitly state fees.

(3) TxnFeeNanos and Nonce parameters will be added to the end of serialized transactions.
These parameters will be deserialized if we do not run out of bytes before we reach their
expected location in the buffer. This will allow us to add the fields without breaking
transaction serialization in most cases. Most importantly, it will allow us to add the 
fields without breaking the existing blocks. Unfortunately, "transaction bundles" (used
for sending mempool transactions between nodes) group all transactions together in a long
concatenation of bytes with no demarcation. Transaction bundles expect the end of a
transaction to be known by the deserializer, which will no longer be true. We will need to
create a new type of transaction bundle message ("MsgTypeTransactionBundleV2") that
revises the serialization for these structs.

### Data Storage Change

Because UTXOs will no longer be needed for any transaction, data storage and transaction
sizes will be reduced.

## Backwards Compatibility

This change will create a hard fork as it will no longer support transactions that have 
UTXO inputs, a fundamental characteristic of transactions today. It will also change
serialization for transaction bundles, which will break node-to-node communication with
nodes that are not upgraded.

## Test Cases

This change will test all previous unit tests under balance model conditions in order to
ensure that all existing transactions behave as expected. It will also add some new 
network tests in order to test changes to the serialization of transaction bundles.

## Security Considerations
This change should not affect the security of the blockchain.  However, given that this
change will fundamentally alter how money flows through the protocol, it may introduce 
"money printer" bugs if it is not thoroughly reviewed and tested. That being said, it will
not affect basic public/private key security on the chain so any discrepancies
could be fixed with a rollback or fork if absolutely necessary.

## Alternate designs considered
Provide a list of other ideas that you considered and their pros and cons.
This will help community understand the constraints under which other ideas were
not chosen.

## Acknowledgements

Big thank you to @diamondhands0 and @AeonSw4n for their help finalizing the spec and
@tijno for starting the discussion.
