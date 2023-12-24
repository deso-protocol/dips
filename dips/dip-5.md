---
dip: 5
title: Rotating Messaging Keys
author: Piotr Nojszewski <GH: @AeonSw4n | DeSo: @petern> 
discussions-url: N/A
created: 2021-12-25
note: This Dip is now outdated. Check out Dip-8 on Access Groups for the latest DeSo messaging protocol.
---

## One-Line Description

Change to the PrivateMessage encryption/decryption protocol based on rotating messaging keys. Introduces DeSo V3 Messages.

## Explanation and Motivation
This proposal marks a third iteration of private messaging on the DeSo blockchain. For historic purposes, let us briefly reflect on the previous versions of the messaging protocols on the DeSo blockchain to see how the V3 change fits in. 

In V1 messages, users were able to send private messages to other users secured via asymmetric encryption. The sender would take the recipient's public key, and use it to encrypt the message with the AES protocol (AES-128-CTR). The encrypted message would then be included in a PrivateMessage transaction and broadcast to the network. The recipient could then take this encrypted message, use their private key to decrypt it, and read the message's contents. 

While effective, this messaging proposal had one important pitfall, namely users weren't able to decrypt messages they sent in chats. Web applications had to rely on local storage to cache sent messages; however, any attempt at reading these messages without the cache would yield the notorious "Message is not decryptable on this device" error. To fix this issue, the V2 messages were introduced.

The V2 messages solved the problem of undecryptable messages by changing the addressee of the message encryption. The two users who wanted to communicate would find a private key known secretly to both of them but no other user on the network, and encrypt/decrypt messages to this shared private key. This would be done through ECDH, which, given two users and respectively two key pairs (p, pG) and (q, qG), computes the shared secret by combining secret key of one party with the public key of another. Namely, we can arrive at a shared secret value p(qG) or q(pG), which is the same EC point. This shared secret would then be used as a seed for a KDF (SHA256 ConcatKDF) to derive a shared key pair. With this protocol, both parties were able to decrypt messages, without additional information.

For a good couple of months, V2 sustained all the needs of P2P messaging on the DeSo blockchain. However, as more apps were built, particularly mobile applications, there was an increasing need to handle messages without having access to user's seed. One important use case were derived keys. Derived keys allowed applications to sign transactions on behalf of a user without accessing their main seed. But derived keys didn't give developers the ability to encrypt/decrypt messages, which were still being encrypted to the user's main public key. We certainly can't give applications access to user's main seed, and the current messaging protocol didn't give any leeway to circumvent these limitations, hence a new protocol altogether was needed.

DeSo V3 messages solve the above problem, and give derived key holders the ability to handle user messages. We do this by introducing messaging keys, which will be registered on-chain and can be safely shared with third-party app developers. Messaging keys will be labeled and derived deterministically from user's main seed. If a user has registered a messaging key, messages can then be encrypted to this messaging key. Specifically, there will be a default messaging key, labeled "default-key", that the DeSo Identity Service will look for when handling messages. DeSo V3 Messages rollout is intended to be a non-forking change, so messaging keys will be passed in transaction's ExtraData, as opposed to in a new transaction.

## Specification
The change will involve all components of the DeSo stack, including Core, Backend, Identity, and Frontend. In addition, app developers should adjust how they communicate with the DeSo Identity Service to include the messaging party in message decryption. Here are Github pull requests for reference:
- [DeSo core](https://github.com/deso-protocol/core/pull/170)
- [DeSo backend](https://github.com/deso-protocol/backend/pull/253)
- [DeSo identity](https://github.com/deso-protocol/identity/pull/107)
- [DeSo node frontend](https://github.com/deso-protocol/frontend/pull/543)

### Technical changes

#### Core
A conditional statement will be added in `_connectBasicTransfer()` to look for transaction's `ExtraData` and check for three fields: `MessagingPublicKey`, `MessagingKeyName`, and `MessagingKeySignature`. These respectively will contain the public key and its name along with an owner-signed hash of `(MessagingPublicKey + MessagingKeyName)`. The signature is provided because we want derived key holders to submit messaging keys; therefore, the transaction signature alone wouldn't be sufficient to prove authorization of the messaging key. Similarly, disconnecting a messaging key will happen in `_disconnectBasicTransfer()` and will delete the messaging entry from the `UtxoView` and DB. Core will only accept one key for a label, an attempt to submit a different public key for the same key name will silently error.

We also change how `PrivateMessage` transactions are connected. To send a V3 message, a `PrivateMessage` transaction should have `Version: 3` field in metadata, as well as `ExtraData` containing a message party information comprising `SenderMessagingPublicKey`, `SenderMessagingKeyName`, `RecipientMessagingPublicKey`, and `RecipientMessagingKeyName`. These fields will describe the messaging keys for the sender and recipient used when encrypting the private message. The fields will be added by core in `_connectPrivateMessage()` which will use them to construct a message and message party entries. Removing the entries from `UtxoView` or DB will happen in `_disconnectPrivateMessage()`.

#### Backend

We modify both `/api/v0/get-messages-stateless` and `/api/v0/send-message-stateless` endpoints to include the messaging party information. We also add an endpoint to fetch user messaging keys.

#### Identity

Initially, users will still be using the V2 message protocol with shared secrets between their primary keys. But as soon as a user registers a messaging key named "default-key", the DeSo Identity Service will be encrypting messages to this public key. The "default-key" messaging key will be returned in `/derive` Window API endpoint. It is suggested to pass the messaging key information in the `ExtraData` of the transaction authorizing a derived key. We also modify the `encrypt` and `decrypt` methods in the iframe API. The `encrypt` method will make a request to the Backend API to look for sender's and recipient's default messaging keys. If either messaging key exists, `encrypt` will create a V3 message using one or both of the messaging keys and return messaging party information along with the encrypted message. The `decrypt` method will make no additional Backend API requests, and will require the messaging party to be passed along with the encrypted message. We make a modification to the request payload to `decrypt` to require `Version: 3` field, along with `SenderMessagingPublicKey`, `SenderMessagingKeyName`, `RecipientMessagingPublicKey`, and `RecipientMessagingKeyName` fields for each message entry.

#### Frontend

A small change is needed in how frontend communicates with the DeSo Identity Service to include the messaging party information.

### Data Storage Change

We define two new prefixes to the Badger DB that represent the prefix for messaging keys and message parties:
- Prefix: `[]byte{57}` stores messaging keys with the entire key `<prefix, PublicKey [33]byte, KeyName [32]byte> -> <>`. The key name is padded with `byte(0)` to always fill 32 bytes to prevent overlapping indexes. The primary user key has a key name of 32 zero bytes. Messaging key entry is encoded using custom deterministic encoding.
- Prefix: `[]byte{58}` stores message party entries with the entire key `<prefix, public key (33 bytes) , uint64 big-endian timestamp (10 bytes)>` which is identical to the key of message entries stored under prefix `[]byte{12}`. The message parties entries are intended to augment message entries and should be fetched along with the message entries to construct V3 messages. Message party entries are encoded using custom deterministic encoding. 

## Backwards Compatibility
DeSo V3 Messages are entirely backwards compatible because all new information is passed in `ExtraData`. New clients will not return on error related to V3 messages, instead they would just abort connecting/disconnecting the V3 message information.

## Test Cases
We test connecting/disconnecting of both messaging key and message party entries. Two new tests are added: `TestMessagingKeys()` and `TestMessageParty()` with standard unit tests for new DB records:
- Testing that entries are correctly connected/disconnected to `UtxoView` and flushed to DB.
- Testing that entries are correctly connected/disconnected to `Mempool` and mined into blocks.
- Testing that entries are correctly rolled back when disconnecting a block.

An additional test is included in `TestMessagingKeys()`:
- Test that only one public key can exist for a given key name.

An additional test is included in `TestMessageParty()`:
- Test sending a V3 message without authorizing a messaging key, which should fail.

## Security Considerations
This change doesn't affect security of any other DeSo transaction. Messaging keys are secured with owner-signed signatures and so they can't be spoofed by adversaries. The DeSo Identity Service remains unaffected with all functionality unrelated to messages.

## Alternate designs considered
There was an alternative design considered based on generating the messaging keys randomly; however, this design wouldn't allow the DeSo Identity to easily derive a private key given a messaging key, and attempts to fix this would arrive at similar method to the proposed design of pseudo-random key derivation with a trapdoor. Another potential design was making messaging keys a new transaction; however, this would result in a forking change.

## Acknowledgements
Special thanks to @diamondhands0 and @maebeam for helping with shaping the DeSo V3 Messages protocol.
