---
dip: 5
title: Group chats on DeSo
author: Piotr Nojszewski <GH: @AeonSw4n | DeSo: @petern>
discussions-url: N/A
created: 2021-12-25
---

## One-Line Description

This change describes the new framework of messages V3 on the DeSo blockchain that enables a wide-range of possible applications including private group chats, public channels, access sharing, and many more.

## Explanation and Motivation
This proposal marks a third iteration of private messaging on the DeSo blockchain. Let us briefly reflect on the previous versions of the messaging protocols on the DeSo blockchain to see how the V3 change fits in.

In V1 messages, users were able to send private messages to each other, secured via asymmetric encryption. The sender would take the recipient's public key, use it to encrypt the message with the [AES protocol](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) (AES-128-CTR), and broadcast it to the DeSo network. The recipient could then take this encrypted message, and use their private key to decrypt it, and read the message's contents. However, in this rudimentary framework, users were unable to decrypt the messages they've sent (only recipients could).

The V2 messages further improved the V1 messages by allowing both parties to decrypt sent messages. The two users who wanted to communicate would first find a key pair secretly known to only both of them but no other user on the network. Then they would encrypt/decrypt messages to this shared key pair. This was done through [ECDH protocol](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie–Hellman), which allows users to independently compute a shared secret and use it to derive the secret key pair using a [KDF](https://en.wikipedia.org/wiki/Key_derivation_function) (SHA256 ConcatKDF). With this protocol, both parties were able to decrypt messages, without additional information.

Messages V3 offer an entirely different paradigm for communicating on the DeSo blockchain. The V3 messages is a natural expansion of the V2 messages, enabling multi-member (multi-party) group chats. In particular, in backwards-compatibility fashion it captures the V2 and V1 use-cases, which are simple group messages. We're very excited about the abstraction that we've developed to facilitate the V3 multi-member chats, especially because it offers a plethora of possible communication applications. We can't wait to see what next-generation Web3 application the community will build with this V3 framework.

The V3 architectural decisions were made to maximally optimize space and computation. It is crucial to prioritize efficiency in our blockchain setting because it ultimately translates to low transaction costs. In particular, we want to store as few copies of messages, group chats, etc. entries in the database as possible without sacrificing computational and indexing efficiency. Our current design stores at most two group chat entries for each chat, and two message entries per each sent message. Importantly, these space requirements remain fixed regardless of the number of members or messages in a group chat. Moreover, querying messages for a group chat is super efficient and requires only a single db scan on an aligned storage section. With this introduction, let's get into the details of the new V3 messages.

## How Groups Are Defined
The first step to creating a group chat is to register a messaging group.

### Messaging Groups
The main change that V3 messages introduce is the creation of groups. The first step to create a group is to register it using the new `TxnTypeMessagingGroup` transaction, which contains the `MessagingGroupMetadata` metadata. Check out the definition of the `MessagingGroupMetadata` structure at the bottom of this section. The user who sends this `TxnTypeMessagingGroup` transaction is considered the group owner. We will refer to this user as the owner of the `GroupOwnerPublicKey` key pair. To explain how the groups work, let's go over each field in the [`MessagingGroupMetadata` struct](https://github.com/deso-protocol/core/blob/1b9d4d61499d66f3c3f2911496785aa6b95dd5d5/lib/network.go#L4952):

#### MessagingPublicKey
Groups will have their own public keys called `MessagingPublicKey` and all messages in the group will be addressed to this public key. This group public key, `MessagingPublicKey` can be a completely random key. However, it is important that the group owner will not lose access to the corresponding private key `MessagingPrivateKey`. To ensure `MessagingPrivateKey` doesn't get lost, we will propose two standards later in the doc that will enable safe storage of the private key.

#### MessagingGroupKeyName
Groups are also assigned human-readable names through `MessagingGroupKeyName`, which are 32-character strings (UTF-8 byte arrays), e.g. `"my-group"`, `"friends-group"`... These names are used to identify the groups and are unique within the owner's registered groups. That is, a user can only register one group for a given group key name, `MessagingGroupKeyName`, but nothing prevents another user from registering a different group with the same key name. More formally, each group is uniquely identified by the pair `(GroupOwnerPublicKey, MessagingGroupKeyName)`.

#### MessagingGroupMembers
Groups can have members; although, they don't necessarily have to. In fact, 1-member group chats are a very interesting use-case, and we'll return to them later. Group members are optionally included in the `MessagingGroupMembers` list as part of the transaction's `MessagingGroupMetadata` metadata. The `MessagingGroupMembers` is a list of `MessagingGroupMember` objects which provide information about each group member. We will explain this structure in detail in the next section.

#### GroupOwnerSignature
You may also notice the field `GroupOwnerSignature` which, as explained in the comment, is only necessary for a special group key having `MessagingGroupKeyName = "default-key"`. For now ignore this as we will return to the significance of the `"default-key"` later when explaining 1-member group keys.

```go
type MessagingGroupMetadata struct {
	// MessagingPublicKey is the key others will use to encrypt messages. The
	// GroupOwnerPublicKey is used for indexing, but the MessagingPublicKey is the
	// actual key used to encrypt/decrypt messages.
	MessagingPublicKey    []byte
	// MessagingGroupKeyName is the name of the messaging key. This is used to identify
	// the message public key. You can pass any 1-32 character string (byte array).
	MessagingGroupKeyName []byte
	// This value is the signature of the following using the private key
	// of the GroupOwnerPublicKey (aka txn.PublicKey):
	// - Sha256DoubleHash(MessagingPublicKey || MessagingGroupKeyName)
	//
	// This signature is only required when setting up a group where
	// - MessagingGroupKeyName = "default-key"
	// In this case, we want to make sure that people don't accidentally register
	// this group name with a derived key, and forcing this signature ensures that.
	// The reason is that if someone accidentally registers the default-key with
	// the wrong public key, then they won't be able to receive messages cross-device
	// anymore.
	//
	// This field is not critical and can be removed in the future.
	GroupOwnerSignature   []byte

	MessagingGroupMembers []*MessagingGroupMember
}
```
Now let's look into what goes into the `MessagingGroupMembers` field.

### Messaging Group Members
Messaging groups can be registered with members, but they can also start as 1-member groups with empty `MessagingGroupMembers`. Group members can always be added at a later time. To add new members, we would resubmit the `TxnTypeMessagingGroup` transaction with identical `MessagingPublicKey`, `MessagingGroupKeyName`, and `GroupOwnerSignature` (optionally), yet with new `MessagingGroupMembers`. To save transaction size, consensus will only accept `TxnTypeMessagingGroup` transaction that append new non-existing members in the `MessagingGroupMembers` list. It is worth noting that currently, it is not possible to directly remove members from an already registered group. We will return to this topic in the limitations section.

#### GroupMemberPublicKey and GroupMemberKeyName
The `MessagingGroupMember` struct contains the `GroupMemberPublicKey`, which is the main public key of the group participant, and most importantly it includes the `GroupMemberKeyName`, which is the messaging group key name of the member. This is perhaps the key design component of the V3 messages. Group members are added by both their public key and by their messaging group key names. That is, by the `(GroupMemberPublicKey, GroupMemberKeyName)` pairs. Does this look familiar? This index is almost identical to the previously mentioned index `(GroupOwnerPublicKey, MessagingGroupKeyName)` for registering a group. That's because it is. We enable the recursion of groups having other groups as members. Allowing groups to be members of other groups might seem counter-intuitive at first but as you will see this design offers an immensely wide range of applications.

Also, non-existing groups can't be added as members. When adding a member through the `TxnTypeMessagingGroup` transaction, consensus will verify if `(GroupMemberPublicKey, GroupMemberKeyName)` has been previously registered by `GroupMemberPublicKey`.

#### EncryptedKey
Finally, `MessagingGroupMember` object contains the `EncryptedKey` byte array. The `EncryptedKey` is what will ultimately allow the group members to decrypt the group chat messages. Let's consider the private key `MessagingPrivateKey` corresponding to group's `MessagingPublicKey`. The idea is to share this `MessagingPrivateKey` with group members via the `EncryptedKey`. Let's say we want to add a member with a group key `(GroupMemberPublicKey, GroupMemberKeyName)`. And let's say this member group key has the public key `GroupMemberMessagingPublicKey`. The `EncryptedKey` would then store the `MessagingPrivateKey` encrypted to `GroupMemberMessagingPublicKey`, or more formally `EncryptedKey = AES(MessagingPrivateKey, GroupMemberMessagingPublicKey)` (AES is the proposed encryption standard, with `ciphertext = AES(plaintext, recipient)`). So the encrypted key just contains the group's main messaging key encrypted to each of the group members.

```go
type MessagingGroupMember struct {
	// GroupMemberPublicKey is the main public key of the group chat member.
	// Importantly, it isn't a messaging public key.
	GroupMemberPublicKey *PublicKey

	// GroupMemberKeyName determines the key of the recipient that the
	// encrypted key is addressed to. We allow adding recipients by their
	// messaging keys. It suffices to specify the recipient's main public key
	// and recipient's messaging key name for the consensus to know how to
	// index the recipient. That's why we don't actually store the messaging
	// public key in the MessagingGroupMember entry.
	GroupMemberKeyName *GroupKeyName

	// EncryptedKey is the encrypted messaging public key, addressed to the recipient.
	EncryptedKey []byte
}
```

## Special Group Chats
By now, you probably have many questions about our group design. The group chat design is intentionally made quite abstract. Our goal is to provide developers with an unrestrained primitive that can be used to implement a wide variety of communication apps ranging from messengers, discussion boards, public channels, or even an email client -- all living atop a blockchain.

With that said, let's look at some examples of groups that showcase the generality and possible applications of our design.

### The Base Group Key

```go
GroupOwnerPublicKey: GroupOwnerPublicKey,
MessagingGroupMetadata{
    MessagingPublicKey: GroupOwnerPublicKey,
    MessagingGroupKeyName: [0, 0, 0, ...],
    GroupOwnerSignature: nil,
    MessagingGroupMembers: nil,
}
```
The **base group key** is a special 1-member group chat. This group chat has the `MessagingGroupKeyName` of 32 zero bytes (i.e. `[0, 0, 0 ...]`) with the `MessagingPublicKey` set to user's public key. This means that the base group key is basically a _group chat expansion_ of user's main public key `GroupOwnerPublicKey`. The base key is automatically registered for every user and is used in place of user's main public key. In particular, this allows for backwards compatibility with V2/V1 messages. The V3 messages treat previous V2/V1 messages as group chats between users' base keys.

### The Default Group Key
```go
GroupOwnerPublicKey: GroupOwnerPublicKey,
MessagingGroupMetadata{
    MessagingPublicKey: DefaultMessagingPublicKey,
    MessagingGroupKeyName: "default-key",
    GroupOwnerSignature: ECDSA(DefaultMessagingPublicKey || "default-key"),
    MessagingGroupMembers: nil,
}
```
The default group key is another special 1-member group chat. The default key has the `MessagingGroupKeyName` equal to `"default-key"` padded with zero bytes at the end to fit into the 32-character standard. Contrary to the base group key, it needs to be registered with a transaction. The default key is intended to serve as a replacement for the base key. Whenever a user has registered the default key, messaging apps should use it for sending messages to that user. The default group key is intended to be deterministically derived from user's main keys `GroupOwnerPublicKey, GroupOwnerPrivateKey`. The private key associated with the default key is computed from user's main private key according to the following formula: `DefaultMessagingPrivateKey = sha256(sha256(GroupOwnerPrivateKey) || sha256("default-key"))`. This formula was chosen because it offers a simple deterministic derivation of the default key.

Now, you might ask, why use the default group key instead of the base group key?

The base key has one fundamental issue, which is access-sharing. Because base key's `MessagingPublicKey` is equal to user's main key, sharing it with third-party apps would be equivalent to giving away your seed phrase. This is unacceptable for obvious reasons. On the other hand, the default group key can be shared with trusted third-party applications without compromising the privacy of user's main seed phrase. In particular, mobile apps are expected to be using the default key along with the derived keys to offer full support for offline transactions. These apps won't have to communicate with the DeSo Identity Service, except for one-time requests to retrieve these keys.

_Note: The `DefaultMessagingPublicKey` in the code definition above is taken from `DefaultMessagingPrivateKey`, and the `"default-key"` in `MessagingGroupKeyName` and `GroupOwnerSignature` is padded with `[0, 0, 0, ...]` at the end to fit into 32 bytes._

### The Channel Group Key
```go
GroupOwnerPublicKey: Secp256k1 base point (0x0279BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798),
MessagingGroupMetadata{
    MessagingPublicKey: CustomPublicKey,
    MessagingGroupKeyName: CustomKeyName,
    GroupOwnerSignature: nil,
    MessagingGroupMembers: nil,
}
```
Similarly to how we can have special messaging keys by defining constant `MessagingGroupKeyName`, we can have special group owners, `GroupOwnerPublicKey`, to facilitate open group chats, or channels. The channel group key is chosen as a specific public key called the Secp256k1 base point, `G`, equal to `0x0279BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798` in compressed format, with the associated private key `k = 0x01`. Messaging groups created by the base key are regarded as the _channel groups_. For instance, they can be used to store unencrypted, public content. This enables apps like a Web3 Discord, where all messaging and financial data is accessible on the blockchain. The `MessagingGroupKeyName` could be used by developers to implement channel discovery, so users can search groups. Moreover, one could count the number of members in a group to have a metric on group relevancy. With all these features, channels are an exciting application of the group chat design.

## Group Messages

Messages are always sent between two groups. This also extends to the past V2/V1 messages, which are conversations between two base group keys. Messages should be encrypted similarly to V2 messages, using a shared secret derived from groups' `MessagingPublicKey`. To send a message in V3 standard, we will be passing the public keys of the sender and the recipient along with their group key names. That is, messages are indexed by the pairs `((SenderPublicKey, SenderGroupKeyName), (RecipientPublicKey, RecipientGroupKeyName))`

If a message is part of a group chat, then the recipient of the message will be set as the group owner with the group key name as `RecipientGroupKeyName`. To decrypt messages sent to the group chat, we would fetch the messaging group corresponding to the message and look for the `EncryptedKey` in our `MessagingGroupMember` field. We would then decrypt the `EncryptedKey` and use it to finally decrypt the group message.


## Standards
As discussed, the first step to creating a group chat is to register the group messaging key. The next step is to add group members. It's worth noting that the group's public key, `MessagingPublicKey` can be an arbitrary public key. Though, it is important that the group owner has access to the private key corresponding to `MessagingPublicKey`. The group owner adds members to the group by sharing the group's private key with them via `EncryptedKey`. This means that the owner should not lose access to the `MessagingPublicKey`. To support interoperability between different messaging clients on DeSo, we suggest the usage the following **Standard 1** and optionally **Standard 2** to ensure private key associated with `MessagingPublicKey` doesn't get lost.
- **Standard 1:** Add the group owner as the GroupMember. Quite simply, we would add a group member with `GroupMemberPublicKey := GroupOwnerPublicKey` and `GroupMemberKeyName := "default-key"` containing the `EncryptedKey` set as `MessagingPrivateKey` encrypted to the group owner's default key.
- **Standard 2:** `MessagingPublicKey` is deterministically derived from `GroupOwnerPublicKey`. Let's call the private key associated with `MessagingPublicKey` as `MessagingPrivateKey`, and one associated with `GroupOwnerPublicKey` as `GroupOwnerPrivateKey`. Given the `GroupOwnerPrivateKey` and `GroupKeyName`, we could compute: `MessagingPrivateKey = sha256(sha256(GroupOwnerPrivateKey) || sha256(MessagingGroupKeyName))` (`||` is [concatenation](https://en.wikipedia.org/wiki/Concatenation)). This simple deterministic derivation allows the group owner to always be able to create the `EncryptedKey` entry for group members, as long as the owner has access to the `GroupOwnerPrivateKey`.

## Suggested Developer Approach
### Registering a Group Chat
1. Choose group members. Enforce that only default-keys can be added as group chat recipients
2. Choose group name
3. Hit identity with group owner, group name, group members []publicKey
4. Take payload from identity and use it in backend API to register the messaging group
5. Take the transaction hex and sign it in identity
6. Submit signed txn

### Sending Group Chats
1. Select group chat for conversation with the group key name
2. Identity
3. ...

### Reading Group chats
1. Fetch message and corresponding group chat.
2. Send the message along with decryption key to identity to decrypt

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
