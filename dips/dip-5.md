---
dip: 5
title: Rotating Messaging Keys
author: Piotr Nojszewski <GH: @AeonSw4n | DeSo: @petern> 
discussions-url: N/A
created: 2021-12-25
---

## One-Line Description

Change to the PrivateMessage encryption/decryption protocol based on rotating messaging keys. Introduces DeSo V3 Messages.

## Explanation and Motivation
This proposal marks a third iteration of private messaging on the DeSo blockchain. For historic purposes, let us briefly reflect on the previous versions of the messaging protocols on the DeSo blockchain to see how the V3 change fits in. 

In V1 messages, users were able to send private messages to other users via asymmetric encryption. The sender would take the recipient's public key, and use it to encrypt the message with AES-128-CTR. The encrypted message would then be included in a PrivateMessage transaction and broadcast to the network. The recipient could then take this encrypted message, use their private key to decrypt it, and read the message's contents. 

While effective, this messaging proposal had one important pitfall, namely users weren't able to decrypt messages they sent in chats. Web applications had to rely on local storage to cache sent messages; however, any attempt at reading these messages without the cache, would yield the notorious "Message is not decryptable on this device" error. To solve this issue, the V2 messages were introduced.

The V2 messages solved the problem of undecryptable messages by changing the addressee of the message encryption. The two users who wanted to communicate would find a private key known secretly to both of them but no other user on the network, and encrypt/decrypt messages to this shared private key. This would be done through ECDH, which, given two users and respectively two key pairs (p, pG) and (q, qG), computes the shared secret by combining secret key of one party with the public key of another. Namely, we can arrive at a shared secret value p(qG) or q(pG), which give the same EC point. This shared secret would then be used as a seed for a KDF (SHA256 ConcatKDF) to derive a shared key pair. With this protocol, both parties were able to decrypt messages, without additional information.

For a good couple of months, V2 sustained all the needs of P2P messaging on the DeSo blockchain. However, as more apps were built, particularly mobile applications, there was an increasing need to handle messages without having access to user's seed. Derived keys allowed developers to sign transactions on behalf of a user, without the need to access user's main seed. But derived keys didn't give developers the ability to decrypt messages, which were still being encrypted to the user's main public key. We certainly can't give applications access to user's main seed, and the current messaging protocol didn't give any leeway to circumvent these limitations, hence a new protocol altogether was needed.

DeSo V3 messages solve the above problem, and give derived key holders the ability to decrypt user messages. We do this by introducing messaging keys, which will be registered on-chain and can be safely shared with third-party app developers. Messaging keys will be labeled and derived deterministically from user's main seed. If a user has registered a messaging key, messages can then be encrypted to this messaging key. There will be a default messaging key, labeled "default-key" that the DeSo Identity Service will specifically look for. DeSo V3 Messages rollout is intended to be a non-forking change, so messaging keys will be passed in transaction's ExtraData, as opposed to in a new transaction.

## Specification
The change will involve all components of the DeSo stack, including Core, Backend, and Identity. In addition, app developers should adjust how they communicate with the DeSo Identity Service to include the messaging party in message decryption.

### Technical changes

#### Core
A conditional statement will be added in `_connectBasicTransfer()` to look for transaction's `ExtraData` and check for three fields: `MessagingPublicKey`, `MessagingKeyName`, and `MessagingKeySignature`. These respectively will contain the public key and its name along with an owner-signed hash of `(MessagingPublicKey + MessagingKeyName)`. The signature is provided because we want derived key holders to submit messaging keys; therefore, the transaction signature alone wouldn't be sufficient to prove authorization of the messaging key. Similarly, disconnecting a messaging key will happen in `_disconnectBasicTransfer()` and will delete the messaging entry from the `UtxoView` and DB. Core will only accept one key for a label, an attempt to submit a different public key for the same key name will silently error.

We also change how `PrivateMessage` transactions are connected. To send a V3 message, a `PrivateMessage` transaction should have `Version: 3` field in metadata, as well as `ExtraData` containing a message party information comprising `SenderMessagingPublicKey`, `SenderMessagingKeyName`, `RecipientMessagingPublicKey`, and `RecipientMessagingKeyName`. These fields will describe the messaging keys for the sender and recipient used when encrypting the private message. The fields will be added by core in `_connectPrivateMessage()` which will use them to construct a message and message party entries. Removing the entries from `UtxoView` or DB will happen in `_disconnectPrivateMessage()`.

#### Backend

We modify both `/api/v0/get-messages-stateless` and `/api/v0/send-message-stateless` endpoints to include the messaging party information. We also add an endpoint to fetch user messaging keys.

#### Identity

Initially, users will still be using the V2 message protocol with shared secrets between their primary keys. But as soon as a user registers a messaging key named "default-key", the DeSo Identity Service will be encrypting messages to this public key. The "default-key" messaging key will be returned in `/derive` Window API endpoint, and it is suggested to pass it in the `ExtraData` of the transaction authorizing a derived key. We also modify the `encrypt` and `decrypt` methods in the iframe API. The `encrypt` method will make a request to the Backend API to look for sender's and recipient's default messaging keys. If either exists, `encrypt` will automatically

#### Frontend

### Data Storage Change
Identify any additional storage requirements in the blockchain due to this change. Carefully consider if there are opportunities to optimize storage requirements.

## Backwards Compatibility
All DIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The DIP must explain how the author proposes to deal with these incompatibilities.

## Test Cases
If this change requires more than just unit tests, include an explanation of how you're going to test your change to make sure it works.

## Security Considerations
All DIPs must contain a section that discusses the security
implications/considerations relevant to the proposed change. Include
possible attack vectors or ways this change could cause problems
down the road.

## Alternate designs considered
Provide a list of other ideas that you considered and their pros and cons.
This will help community understand the constraints under which other ideas were
not chosen.

## Acknowledgements
If there were additional community members that helped you guide / hone your
thoughts, or members who gave constructive feedback to make this DIP better.
