---
title: This Week in Cryptography Research June 30
date: 2024-06-30
tags: ["weekly"]
---

Welcome to the third issue of the cryptography research weekly! This week I read about adaptor signature contruction 

# Adaptor Signatures: New Security Definition and A Generic Construction for NP Relations([Link](https://eprint.iacr.org/2024/1051.pdf))

I have read about adaptor signatures in [Bitcoin land](https://bitcoinops.org/en/topics/adaptor-signatures/) a while ago, this paper offered a generic construction that is not only particular to ECDSA or Schnorr. 

An Adaptor signature scheme is defined based on a relation R where there is a signer and a receiver. The signer can pre-sign a message (e.g., a transaction) with respect to some instance `Y` to obtain a pre-signature. The receiver can then be adapted to a full signature with the knowledge of the witness `y` for `Y` that satisfies the relation `(Y, y) ∈ R`. This is useful for applications like atomic swaps, where the sender can produce a pre-signature, and only if the receiver can fulfill something off chain to obtain the witness, can he complete the full signature, thus executing a transaction that claims a payment. 

An issue with the witness extractability arises, the adapted signature σ is uploaded to the blockchain, making the witness accessible to everyone on the network. This paper formalizes a technique that makes the witness extractable only with both the pre-signature and the full signature, but not just with only one of them. In addition, the paper generalizes the adaptor signature scheme to any NP relations by proving the completeness of the zero-knowledge protocol for the Hamiltonian cycle problem.

We first take a look at the definition for Trapdoor Commitments with Specific Adaptable Message: 
```
keygen -> (ck, td) commitment key and trapdoor
commit(ck, m) -> (c, d) a commitment and an opening
verify(c, d, ck, m) -> 0/1
td_open(td, c, d, ck, m, m_0) -> d with an adaptable message m_0, an adapted opening d can be obtained. 
```

Observe that if the commitment `c` of the specific adaptable message `m_0` serves is in the pre-signed message, then with the knowledge of the trapdoor, one can open the commitment `c` to the real message to be signed and hence form a valid adapted signature. By making the commitment key the Hamiltonian problem instance and the trapdoor is the Hamiltonian cycle witness, the paper proves the existence of one-way functions implies the existence of witness hiding adaptor signatures for any NP relation.

# Enhancing Local Verification: Aggregate and Multi-Signature Schemes ([Link](https://eprint.iacr.org/2024/1055.pdf))

This paper discussed a technique to improve efficiency to aggregated signature verification. Traditionally, for a set of signatures produced over different messages that aggregates to one signature, the verification algorithm needs to take the whole list of messages to verify against the aggregated signature. Locally verifiable signatures allow efficient verification by enabling a verifier to check the presence of a particular message in an aggregate signature without accessing the entire set of messages.

```
setup -> (sk_i, vk_i) for each signer i
sign(m, sk_i, agg_vk) -> sig_i
aggregate(sig_i, agg_vk, m) -> agg_sig
aggregate_verify(agg_sig, agg_vk, m) -> 0/1
local_open(agg_vk, {j_i}_i=1..k, m) -> aux
local_aggregate_verify(aux, {vk_i}_i=1..k, m, agg_vk, agg_sig) -> 0/1
```

How is `aux` calculated? $aux = e(H_0(m), \prod_{i=1, i \neq j_1..j_k}^{n} vk_{i}^{a_{i}})$ where $H_0$ is the hash function in $G_1$. This way, the local aggregate verify can check the following: 

$e(agg_sig, g_2) == aux \cdot e(H_0(m), \prod_{i=1}^k vk_{j_i}^{a_{j_i}})$

This is more efficient than the aggregate verify that checks: 
$e(agg_sig, g_2) == e(H_0(m), \prod_{i=1}^{n} vk_i^{a_i})$

With the local aggregate verify, it can efficiently check 
signers' signatures are included in the multi-signature. As the number of signatures to be verified increases, the verification phase operates more efficiently. 