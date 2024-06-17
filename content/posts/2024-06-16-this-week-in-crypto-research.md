---
title: This Week in Cryptography Research June 16
date: 2024-06-16
tags: ["weekly"]
---

This is the augural issue for This Week in Cryptography Research. I hope to summarize 2 papers I read this week and highlight the main contributions. I focus on papers that are most interesting to me, and has a strong preference to practical applications and constructions over theories and attacks. 

# Atomic and Fair Data Exchange via Blockchain ([Link](https://eprint.iacr.org/2024/418))

I came across this paper while reading about various data storage solutions related to the blockchain. The paper highlighted a flaw with the current proof of storage design: the data is stored off-chain (since storing data on a blockchain is prohibitively expensive) with a succinct commitment on-chain attesting the data is stored. The user only pays when storing, but does not pay when querying or reading the data. 

The construction starts with a simple atomic exchange protocol: a user wishes to obtain a data from server. He first locks a small payment along with the data commitment. The data is posted by the server and the payment can be claimed if only if the data matches the commitment. Where is this data being posted? It will still be expensive to posted it on-chain, so the protocol designs to have the data exchange off-chain but only to have the verifying key committed on-chain. 

The protocol is described as follows: The server first encrypts the data, then outputs a verification key, a ciphertext and a zero-knowledge proof of correct encryption. The client can now lock the payment on-chain if and only if the proof verifies against the ciphertext and commitment. Then the server can claim the payment if and only if it provides a decryption key that matches the verification key. Then the client reads the decryption key from on-chain and decrypts the data. 

A few more details: How does commitment work? KZG polynomial commitment scheme is used for its constant size commitments and batchable opening proof. How can we prove that the encryption is correct? There are two instantiations using exponential ElGamal encryption and public-key version of Paillier encryption - in particular, the equality proof for ElGamal encryption is shown [here](https://crypto.stackexchange.com/questions/30010/is-there-a-way-to-prove-equality-of-plaintext-that-was-encrypted-using-different). 

The protocol also extends to a multi-client setting where multiple clients can download the same data with some time saving. This is useful when multiple light clients request the same block data. How can we compress the algorithm? The proof generation for different clients can be saved with some preprocessing. 

# Nebula: A Privacy-First Platform for Data Backhaul ([Link](https://eprint.iacr.org/2024/409.pdf))

This paper focuses on a very practical use case of building decentralized data collection network. One of the biggest challenge in this space is how to retrieve and utilize data from large scale networks of low power and dispersed sensors with incentives while preserving privacy of the sensor. An example is how AirTags can crowdsource local network for data on the lost items, while rewarding the sensors to provide more data, but without leaking sensitive data from the participants of the network. Nebula devices a decentralized network for such data platform, where the sensors contributing the data are rewarded through micropayment, while improving location privacy. 

The construction involves the platform providers (the platform that registers application servers, sensors and coordinates payments), the application server (the server that receives data), and data mules (devices such as phones that has intermittent Internet access, that detects sensors, read and upload data from sensors to application server). 

The protocol is instantiated with unlinkable tokens - What is it? A server generates PRF secret key `k` and reveals its commitment as `Y`. A client picks a random input to the server, that the server evaluates it to `t = prk_k(x)` and outputs an equivalence proof. Later the result `t` can be evaluated as a valid output against the public commitment `Y` corresponds to `k`. 

The architecture is designed as follows: 1. The application server pre-purchase unlinkable tokens 2. The mules encounters sensors, verify identity by authenticating the sensor certificates to the CA, then receives data from sensor 3. Mules authenticates with the application server, delivers payload and redeem tokens. Since the tokens are not linked to who purchased them, the application cannot identify the sensors that provide the data. 

The paper goes into detailed evaluation around device energy consumptions and economic incentives that shows that the system only has a small overhead (5% energy and 3MB storage) for the benefit of not revealing its location to any server. The use of cryptography in this system is light, but the contribution is still very solid from its comprehensive system design and practicality. 

