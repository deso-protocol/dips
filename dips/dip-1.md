---
dip: 1
title: Simple Key Delegation
author: Mae Beam <@maebeam>
discussions-url: N/A
created: 2021-07-07
---

## One-Line Description

Allow users to delegate signing permissions to another private key using only ExtraData

## Explanation and Motivation

Today, iOS and Android apps have full access to a user’s key material. This means
any third-party app can steal the user's key material after they log in, even if
this login ostensibly occurs in a DeSo Identity window.

To prevent this,
our options are to either require all
transaction signing to happen outside of third party apps or to provide third
party apps with easy access to key material that expires at some point. The
latter would make it so that the key material the app gets isn't useful after
some period of time, limiting the attack surface.

Most users don’t want to approve every transaction they perform on a third
party app. Core product features like diamonds make transferring DESO and creator coins an
action users do multiple times per day. Furthermore, we have learned there is
no way to run a signing service in the background on iOS. For all these
reasons, we should solve for providing third party apps with easy access to
expiring key material.

iOS provides an API called ASWebAuthenticationSession which allows users to
sign in using Safari cookies and local storage. This API lets us return
expiring key material to third party apps. Users will need to re-approve access
every 1-4 weeks.

A user creates a new public/private key pair and hashes the new public key with
an expiring block height and signs the hash with their true public key. The
user includes the new public key, the expiring block height, and the signature
in the transaction's ExtraData. Consensus validates the hash and signature.

## Specification

### Logging In
A third party app can request full signing privileges for 1-4 weeks. Identity is aware of the current block height and picks a height 1-4 weeks in the future.

When the user approves the request, Identity:
1. Generates a new public and private key pair
2. Hashes the combination of expiring block height + new public key
3. Signs the hash with the user’s true public key

The following information is returned to the third party app:
- True public key
- New public key
- New private key
- Signed hash

### Constructing and Signing Transactions
When you construct a transaction the following information is included in ExtraData:
- Expiring block height
- New public key
- Signed hash
- The transaction’s PublicKey is set to the true public key. The transaction is signed using the new private key.

### Validating Transactions
If the relevant ExtraData is present, consensus:
1. Hashes the expiring block height + new public key
2. Verifies the hashes match and the hash was signed by the true public key
3. Verifies the current block height is less than or equal to the expiring block height

Two other approaches were considered but they didn't end up working

### ❌ Use a BIP-32 key derived at block height N to sign transactions
This idea would’ve worked perfectly. Unfortunately, due to the way BIP-44 derived keys work, knowing an extended parent public key and the private key of a non-hardened child allows you to derive the private key for all the siblings. This means an app would be able to derive the private key for any block height which totally defeats the purpose.

Interesting reads
- https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- https://github.com/bitcoin/bips/blob/master/bip-0044.mediawik
- https://bitcoin.stackexchange.com/questions/100529/proving-two-public-keys-are-derived-from-the-same-seed-without-revealing-the-xp
- https://bitcoin.stackexchange.com/questions/90649/consequences-of-sibling-keys-when-xpub-and-xprv-are-leaked

#### Logging In
Identity returns the following information:
- Extended public key for your address
- Block height of the derived key
- Private key of the derived key

#### Constructing Transactions
When you construct a transaction the following information is included in ExtraData:
- Extended public key for your address
- Block height of the derived key
- Private key of the derived key

#### Signing Transactions
The transaction is signed using the private key of the derived key.

### ❌ Submit a transaction that authorizes a public key for N blocks
This idea requires users to submit transactions to the chain. This means new users without any funds would need to be subsidized and the Identity layer would become too complex.

## Backwards Compatibility
There are no issues with backwards compatability

## Test Cases
Test cases will be created in the core and identity repositories

## Security Considerations
A change to signature validation in consensus is extremely risky. Adequate test cases will need to be written to be sure a fake transaction cannot sneak in.
