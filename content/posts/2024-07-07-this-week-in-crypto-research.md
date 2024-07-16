---
title: This Week in Cryptography Research July 15
date: 2024-07-15
tags: ["weekly"]
---

Welcome to the fourth issue of the cryptography research (bi-)weekly! This week I took a look at an interesting construction on top of OpenID to ensure privacy of relying parties to identity providers. Another topic focuses on how to do remote attestation of devices more efficiently. Both leverage some commitment schemes as part of the constructions. 

# OPPID: Single Sign-On with Oblivious Pairwise Pseudonyms([Link](https://eprint.iacr.org/2024/1124))


Single Sign-On (SSO) is a protocol that allows users to conveniently authenticate to many Relying Parties (RPs) through a central Identity Provider (IdP). For example, an application can create client IDs within Google, and Google authenticates users with "Allow xx Application to sign in with Google" and returns a token scoped to the RP. However, the RP itself is transparent to the IdP on every authentication request, since the IdP learns the RP that the user wants to access to generate the token. So the question comes: How can the IdP ensure that issuing tokens can still bind to the RP ID, but without learning the RP ID itself? At the same time, the token must be verifiable to the registered RP.

The paper proposes to have the IdP to maintain an updateable membership state. Whenever an RP registers with IdP, the RP receives a `cred` value (a signature committed on `rid`), and the membership state is updated. When initiating an authentication request, the user first obtains `(crid, orid)` (a commitment value and an opening). RP then generates `auth` based on `orid, crid, cred` along with a random session ID `sid`. What is `auth`? It is a proof that `cred` indeed commits to `rid` and `crid` is indeed a commitment over `rid`. 

Then IdP verifies `auth` and signs with its secret key `isk` over `crid` to produce a token $\pi$. Then this token can be finalized $\pi_final$ by opening the commitment, and a pseudonym ID can be derived. Finally, the $\pi_final$ can be verified against the IdP public key `ipk`. 

With some overhead to the original OpenID spec, this algorithm achieves the unobservability of the token. The IdP can blindly derive the user's pairwise pseudonym (since it only has `auth` not `rid`) for the targeted RP without learning the RP's identity and without requiring key material handled by the user. 

# From Interaction to Independence: zkSNARKs for Transparent and Non-Interactive Remote Attestation
([Link](https://eprint.iacr.org/2024/1068))

Remote attestation (RA) evaluate the integrity of software on remote devices, such as IoT devices. Previously, most of the RA techniques usually involved interactive communications, and/or pre-shared knowledge of devices and keys. The verifier sends a random challenge to
the device and the device's response is generated based on some secret value of the device. To address the lack of transparency and poor scalability of such methods, this paper proposes a protocol that is non-interactive, transparent, and publicly provable based on SNARKs.

In a nutshell, the manufacturer first commits to all the future responses of each device for every challenge. How? It builds a Merkle tree of a hash of the device public key, the challenge and its response `H(pk_i || c_i || r_i)`. This Merkle root is posted to the blockchain. 

During remote attestation, the devices are simply required to prove their knowledge of the correct responses to each challenge. So now everyone can verify the correctness of the attestation claim, and only the verified devices can reveal the commitment correctly. 

How does it work? The device queries the blockchain to retrieve the latest global
challenge. Then the device computes the attestation evidence, denoted as `r_i` and constructs a proof that shows that it has a commitment that `r_i` is part of the Merkle tree. Then the device submits its proof along with the public input and output of the circuit, i.e. challenge `c_i`, public key `pk_i`, and resulting final root `RT` to the blockchain.