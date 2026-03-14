---
layout: post
title: "Understanding ECDSA Signatures in Blockchain: From Math to Wallets"
date: 2025-05-23
categories: analysis
tags: [ecrecover]

---


The backbone of blockchain is cryptography because it ensures data security, integrity, and trust without needing a central authority.

One key innovation is ECDSA,which is used to Proof of ownership in Ethereum is achieved using public and private key pairs that are used to create digital signatures. Signatures are analogous to having to provide personal ID when withdrawing from the bank - the person needs to verify that they are the owner of the account.

### What is ECDSA 
The Elliptic Curve Digital Signature Algorithm (ECDSA) is based on Elliptic Curve Cryptography (ECC) and is used to generate keys, authenticate, sign, and verify messages

 **Signatures** provide a means for authentication in blockchain technology, allowing operations, such as sending transactions, to be verified that they have originated from the intended signer


### Basics Of Elliptici curve cryptography:

ECC is based on algebraic structures of elliptic curves over finite fields.
Ethereum uses:

> Curve: secp256k1

> Equation: y² = x³ + 7 over a finite field F_p where p is a 256-bit prime.

 It’s a smooth, symmetric curve (about the x-axis). Every operation is modulo a large prime, so the curve becomes a finite set of points

 #### Graph Concept:

 ![ECDSA]({{ '../assets/images/ECDSA.jpg' | relative_url }})

> Points on the curve can be added together geometrically.

> Multiplying a point (e.g. G) repeatedly is called scalar multiplication and is used for key generation

<!-- #### Visula Representation:

    ![ECDSA](../images/cycle.png) -->



#### KeyGeneration:

> Private Key (p): A random number in [1, n-1], where n is the curve's order.

> Public Key (Q): A point on the curve computed by:
      Q = p * G
where G is the generator point (fixed for secp256k1).

**Security Basis**:
It’s computationally infeasible to reverse:

Given Q and G, it’s almost impossible to find p
This is called the Elliptic Curve Discrete Logarithm Problem (ECDLP)

#### Signing Mechanism(r,s,and v are formed):

***Steps to create Signature Creation***:

 msg: message (usually hashed using keccak256)

 p: private key

 n: order of the curve


 ```yaml

 steps :

 1.Hash the message:h = keccak256(msg)

2.Generate a random nonce k (must be secret and unique every time!)

3.Compute random point R: R = k * G, take its x coordinate → r = R.x mod n  (Think of r like a fingerprint for that unique signature instance.)

4.Compute s: s = k⁻¹ * (h + r * p) mod n  (It ties your private key to the specific message hash securely.)

5.Compute v: The recovery ID, to tell which of the two possible public keys to recover.( helps Ethereum get the signer's address using ecrecover().     )
v = 27 + recovery_id → 27 or 28

**Final Signature = (r, s, v)**
```

#### Signature Verification:
Use (r, s, v) to verify that the signer owns the private key.

```yaml
In Ethereum:

1.Recompute h = keccak256(msg)

2.Compute:
u1 = s⁻¹ * h mod n
u2 = s⁻¹ * r mod n

3.Compute point:
R' = u1 * G + u2 * Q

4.If R'.x mod n == r, the signature is valid

```
On-chain in ethereum:
```solidity
address signer = ecrecover(hash, v, r, s);

```
#### Why ECDSA is Better (for now)

1.Smaller keys/signatures than RSA

2.Efficient on-chain operations (especially in Ethereum)

3.Widely adopted (Bitcoin, Ethereum, etc.)

4.But: vulnerable to quantum computing in the long term


### Why ECDSA needed in Blockchain

ECDSA is used in blockchain to securely sign transactions so that only the person with the private key can authorize actions, and everyone else can verify it using the public key — ensuring authenticity, integrity, and non-repudiation of data without a central authority.

In simple terms:

ECDSA proves "I am the owner of this wallet" without revealing your private key, which is essential for trust and security in a decentralized network.

### How it works :
let us take an example to understand the ECDSA in ethereum:

Nithinkumar wants to send 1 ETH to Uday. He uses an Ethereum wallet (e.g., MetaMask) to sign the transaction.

1.Message to be signed (Transaction Data):
 ```yaml
{
  "to": "0xudayAddress",
  "value": 1 ETH,
  "nonce": 0,
  "gasPrice": 20 Gwei,
  "gasLimit": 21000
}
 ```
 2.The wallet hashes the transaction and signs it using nithin's private key with ECDSA, producing a signature:

 ```yaml
 Signature = (r, s, v)
 ```
 3.This signed transaction is broadcasted to the Ethereum network.
 4.Ethereum nodes receive the transaction, use the (r, s, v) values to:

 ```text
      Every Ethereum transaction includes:
       
        to address
        value
        nonce
        gasPrice
        gasLimit
        data
        And the signature (r, s, v)
```

The node takes all of the transaction fields except the signature, and computes a Keccak256 hash of this transaction — let’s call it msgHash.

Using msgHash and the (r, s, v) values, the node uses the ECDSA recovery algorithm to reconstruct the public key that created the signature.

```solidity
publicKey = ecrecover(msgHash, v, r, s)//This is a standard algorithm available in Ethereum and cryptography libraries.

```
**derive the ethereum address using public key**
Once the public key is recovered:

Ethereum Address: Last 20 bytes of keccak256(public key)

```yaml
Use the secp256k1 curve and multiply the private key with the generator point G:

publicKey = privateKey * G

Public key is a 128-character hexadecimal (512 bits) if uncompressed (starts with 0x04 + x + y).

Hash only the x and y part (not the 0x04 prefix).

keccak256(publicKey[1:])  // public key without 0x04 prefix

ethereum address = last_20_bytes(keccak256(pubkey))

```

This produces a 20-byte (40-hex-character) Ethereum address.

5.Verify that the recovered address matches the sender's address in the transaction.

If the signature is valid and Nithinkumar has enough ETH, the transaction is accepted and mined into a block.

### Security Concerns :

#### Replay Attack:

Why Malleability Happens:

```yaml
Due to curve symmetry, two values of s can produce valid signatures:
    s
    n - s
```
Both are valid, allowing attackers to create a second signature without the private key.

>This allows replay attacks across networks (Ethereum ↔ BSC) or in contracts that don't check strict signature formats.

#### How to Prevent :

1.Enforce low s values: s ≤ n/2

2.Normalize signatures before verifying

3.Use EIP-155 to include chain ID in transactions

#### How Private Keys Can Be Stolen(if u neglect the nonce) :
ECDSA is secure only if:

> ⚠️ Never reuse the same nonce `k` — this will leak your private key!

Private Key Leaks Happen If:
   Same k reused → Attacker can compute p directly!

   Predictable k (bad RNG) → s becomes solvable

   Weak entropy during key generation

#### Exploit case:
If two signatures have the same r, attacker can solve for k, then back-calculate p.
given:
```text
s1 = k⁻¹ * (h1 + r * p)
s2 = k⁻¹ * (h2 + r * p)

```
Subtract:
```text
s1 - s2 = k⁻¹ * (h1 - h2)
⇒ k = (h1 - h2) / (s1 - s2)
⇒ p = ((s * k - h) / r) mod n
```
***Attacker got the private Key game over***

### Conclusion:

ECDSA keeps blockchain transactions safe and verified.

It proves who owns a wallet without showing the private key.

In this blog, we saw how ECDSA works and why it’s used in blockchains like Bitcoin and Ethereum. It’s what makes sure only the real owner can send crypto.

It’s a small part of crypto, but very important for trust.
