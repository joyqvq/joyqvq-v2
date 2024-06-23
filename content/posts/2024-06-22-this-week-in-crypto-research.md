---
title: This Week in Cryptography Research June 22
date: 2024-06-22
tags: ["weekly"]
---

Welcome to the second issue of the cryptography research weekly! This week I read about randomness implementation considerations and encryption techniques. 

# VRaaS: Verifiable Randomness as a Service on Blockchains([Link](https://eprint.iacr.org/2024/957.pdf))

This paper provides a formalized framework for verifiable randomness that captures core features such as unbiasability, unpredictability, and public verifiability but also the onchain components design. The randomness requester first sends a request to smart contract and records a unique ID onchain. The VRF committee reads the value and computes VRF output that generates a publicly verifiable proof of correctness of the generated randomness. The smart contract verifies the proof and then returns the output to the requester via a callback function that fulfills the request. The smart contract also records the payment and ensures the atomicity of the service offering against the payable fee.

The key observation of the paper is that it is necessary to post two on-chain transactions where the requested randomness transaction is posted with a sequence number. The uniqueness and unbiasability can only be guaranteed by the unique sequence number. Removing this transaction (and replacing it with a direct message to the requester) enables a malicious requester to deny the output if it is unfavorable and keep sampling until a favorable one is obtained. 

It discusses how Pyth VRF does not fulfill this flow and can be exploited. The VRF server pre-computes `x_1, x_2..., x_n` and computes onchain. A request samples `x_u` and send `H(x_u)` to smart contract. The contract assigns sequence number `i` and stores `i -> H(x_u)` onchain, returns to the requester, and increment `i`. Now the requester calls the VRF service to compute `x_i`. Since now the requester can compute offchain its randomness as `r =  H(x_i, x_u)` and he can choose abort to publish `r` because `x_u` is private to the requester.

# FABESA: Fast (and Anonymous) Attribute-Based Encryption under Standard Assumptions ([Link](https://eprint.iacr.org/2024/986.pdf))

The idea of encryption based on access control has been an interesting research topic for me to read. This is a very practical construction that can be used for data sharing within organizations with defined access control and can even be applied in public blockchains given its privacy guarantees over ciphertext and policy structure. In addition, this paper includes a very good section on background and definitions and also provides a very comprehensive list of implementations and benchmarks. 

FABESA proposes a key-policy and ciphertext-policy ABE schemes that (1) support both AND and OR gates for access policies, (2) have no restriction on the size and type of policies or attributes, (3) achieve adaptive security under the standard DLIN assumption, and (4) only need 4 pairings for decryption.

What is key-policy attribute based encryption? It means the ciphertext contains an attribute set and it can be only decrypted with a secret key with the correct access policy. On the other hand, ciphertext policy attribute based encryption means the access policies associated with the ciphertext and attribute sets attached to the secret key.

In addition to expressiveness, security, and efficiency, the main contribution in this paper is ciphertext anonymity. The ciphertext does not reveal the message itself nor the attribute values.

We won't dive into the algorithms, but the intuition here is that a payload `pl_x` associated with the secret key is generated during key generation, and another payload `pl_y` associated with the ciphertext is generated during encryption. Note that the policy name is revealed but the value is hidden (i.e. partially hidden structure).

```
setup -> pk, msk
keygen(pk, msk, i) -> sk_x, pl_x
enc(pk, y, msg) -> ct_y, pl_y
dec(pl_x, pl_y, ct_y, sk_x) -> msg
```

How to achieve anonymity? Inner product encryption is designed to encode policy and attributes in equal sized vectors `x` and `y` and embedded into the key and ciphertext. The decryption only succeeds when $x \cdot y = 1$. 

For example, for a policy `A = (a_1 OR b_1) AND c_1` can be transformed to a polynomial $P(x_1, x_2, x_3) = r_1 \cdot (x_1 - a_1) \cdot (x_2 - b_1) + r_2 \cdot (x_3 - c_1)$. To unpact the polynomial, we have coefficient $ x = (a_1b_1r_1 - c_1r_2, -b_1r_1, -a_1r_1, r_2, 0, 0, 0, r_1, 0, 0)$ for $(1, x_1, x_2, x_3, x_1^2, x_2^2, x_3^2, x_1x_2, x_1x_3, x_2x_3)$. For an attribute set $(a_1, b_4, c_1)$ as $y = (1, a_1, b_4, c_1, 0, 0, 0, a_1b_4, 0, 0)$. There we have $x \cdot y = 0$. 

# Swiper: a new paradigm for efficient weighted distributed protocols ([Link](https://dl.acm.org/doi/pdf/10.1145/3662158.3662799))

This paper discussed a technique on how to transform a nominal model to a weighted model using weight reduction in a simple way. This is especially useful for weighted threshold systems such as weighted voting to be baked into the verification, instead of tracking the weight outside the signature verification itself. 

# SoK: Programmable Privacy in Distributed Systems ([Link](https://eprint.iacr.org/2024/982.pdf))

Long paper warning! A lot of good definitions, and the most useful paradigm I find is the computation framework that the data will go through in a privacy focused application.

- Independent computation: This is where the computation to the private input data is committed and attested via a proof. 
- Mediated computation: The computation is jointly computed by the users and some operators. The private inputs can be encoded with techniques such as MPC, FHE etc.
- Global computation: When the application level computation has to be executed by all nodes on the network and the proofs are verified. 