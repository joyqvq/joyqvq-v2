---
title: A Tale of Privacy-focused Transactions
date: 2023-09-04
tags: ["crypto"]
---
## A Tale of Privacy-focused Transactions

Blockchains had been developed under one public verifiable state machine, where all transactions are sequenced globally under consensus. What comes as a tradeoff to providing a consistent and shared state, blockchain suffers from the fact that all of the transactions on the blockchain are viewable from the public ledger by anyone. The privacy and security concerns had posed many risks and hindered adoption of cryptocurrencies as digital cash. Insufficient privacy can also result in a loss of fungibility. This post explores several privacy enhancement and research directions in various blockchains models. 

In UTXO based currencies such as Bitcoin and Litecoin, privacy is partially addressed by having the sender generate a new receiving address for each transaction. The outputs are initially unlinkable, but eventually when they were to be spent in one transaction, all the UTXOs that belong to different addresses are linked as a result.

We desire a blockchain to be able to maintain a globally consistent state, but should not require every state transition to be posted publicly. Generally speaking, we care about privacy with respect to some of the following properties:

1. Hiding sender and/or receiver for data anonymity
2. Hiding the amount for confidentiality

There are two general directions that privacy transactions solution take:

1. Direct P2P Transfer: This typically involves protocol level changes so that every miner or validator on the network can verify the new signature type with privacy specific schemes.

2. Centralized Intermediary: This is to rely on a dealer to facilitate transfers such that the out and in coins from the pool cannot be linked.

Note that for examples mentioned below are a general description of the original paper and the protocol spec, but the exact implementation details deployed may have upgraded or extended significantly.

# Direct P2P Privacy Transaction

The existing blockchains mostly fall into two categories of data model: UTXO based or account based model. The former does not contain smart contract functionality and the coins (inputs, outputs) themselves hold all the information for transaction validity (a list of inputs, outputs and scriptSig). On the other hand, account based blockchains require the account itself holds programs and their states themselves. The approaches for different models diverge in many ways - let's dive into the details.

## UTXO Based Solutions

### Confidential Transaction (Maxwell)

Each UTXO contains an amount and a public key - a transaction destroys UTXOs and creates new ones that are less than the total amount, along signatures provided by private key associated with the corresponding public key.

The first version of confidential transaction was discussed in Maxwell's [post](https://web.archive.org/web/20150628230410/https://people.xiph.org/~greg/confidential_values.txt) and also this [paper](this [paper](https://blockstream.com/bitcoin17-final41.pdf)). The protocol replaces the explicit UTXO amount with a homomorphic commitment to the amount, and attaches a range proof that the output amount is committed to a range.

The commitment scheme used is called Pederson Commitment constructed based on Elliptic Curve points. A commitment $C$ to value $x$ is calculated as $comm(r, x) = xG + rH$. Note that $r$ is a randomly sampled blinding factor, $G$ is the generator for the group and $H$ is an additional generator. To check if the commitment $C$ is valid, we check that $C == mH + rG$ when the $r$ and $m$ are revealed. Note that this satisfies the additive property: $comm(r_1, data_1) + comm(r_2, data_2) == comm(r_1 + r_2, data_1 + data_2)$.

Since Bitcoin transaction is a set of input coins and a set of output coins, the commitment scheme gives us a very simple construction: If the author can picks the blinding factor properly, a transaction can be verified by checking:

1. (Unchanged from Bitcoin protocol) Signatures of all inputs are verified. 

2. Verifies the sum of outputs of a transaction equals to the sum of inputs: (sum of all commitments of each input + tx_amount * H) - (sum of all commitments of each output + fees * H) == 0.

3. Verifies the sum of outputs is larger than 0. 

Step 2 can be done via the commitment scheme and in step 3 we leverage a new primitive called range proof: proving a value is within a certain range without revealing the value itself.

How does rangeproof work? There are various mechanisms for this, here we describe the most common building block called a OR proof, which can prove that a value is committed to either 0 or 1 (without revealing the blinding factor):

- To commit to 0: I can simply provide a signature of hash of commitment using the private key as the blinding factor. 

- To commit to 1: I can provide $C$ to the verifier, and the verifier computes $C'$: $C' = C - 1H$. By providing a ring signature over ${C, C'}$, I can prove that the signer knows the discrete log of at least one of the public keys corresponding to ${C, C'}$. This step is interactive. 

With the OR proof above, we can actually express any value in base-2 (or base-n) and create OR proof for a list of coefficients. That is, $C = C_1*2^0 + C_1*2^1 + .. + C_5*2^5$. For example, in order to prove C is in a range $[0, 32)$, we can just prove that $C_1...C_5$ are committed to either 0 or 1.

Mimblewimble made an improvement to signature compressions in step 1. Instead of providing signatures for each input, the sender can only provide one signature for the difference of the outputs and inputs as the public key. The range proof and commitment schemes are the same.

Note that the size range proof itself can also be greatly compressed thanks to [Bulletproof](https://eprint.iacr.org/2017/1066.pdf). This is an active research area and can provide efficiency to range proof used in confidential transactions as a drop in replacement.

### Monero

Monero is a privacy transaction protocol deployed as an altcoin, with several modifications built on top of Maxwell's CT construction.

Inputs are signed with Schnorr-style multilayered linkable spontaneous anonymous group signatures, a type of ring signature.

Each ring is a set of public keys, one of which belongs to the signer and the rest of which are unrelated. The signature is generated with that ring of keys, and anyone verifying it would not be able to tell which ring member was the actual signer. We call this an anonymity set. This also means the size of the signature increases linearly with the size of the anonymity set.

It is multilayer because ve a private key and several associated auxiliary keys. It is important to prove knowledge of all private keys, but linkability only applies to the primary.

For each output, a one time address is generated and communicated to recipients via ECDH. Similar to confidential transactions, outputs are concealed with Pedersen commitments and proven in a valid range using Bulletproofs.

### ZCash

ZCash takes the approach of zkSNARK which is based on a different set of primitives in zero-knowledge proofs. In short, ZCash authenticates coins by proving, in zero-knowledge, that they belong to a public list of valid coins maintained by the blockchain.

We define a coin in ZCash as a tuple: $coin = (r, sn, cm)$. $sn$ is the serial number (also called nullifier) of the coin that will later be revealed. This is necessary to ensure that a coin cannot be consumed more than once: the same serial number would appear twice. $cm$ is the commitment to $sn$ using $r$.

Mint transaction: The sender creates a transaction with commitment to the new coin's serial number $cm(sn_{new})$ and the coin value $v$ and reveals the consumed $sn$.

Pour transaction: The transaction contains the serial numbers of the consumed coins, commitments of the created coins, and a zero knowledge proof attesting that the serial numbers belong to coins created in the past. That is, $tx = (rt, sn_{old}, cm_{new}, v)$ where $rt$ is the root of the Merkle tree. 

The blockchain needs to maintain a Merkle tree of all coin commitments and all revealed serial numbers. Similar to Monero, the set of serial numbers grows linear in the number of transactions.

ZCash had initially started with Groth16 proof system for SNARK and now transitioned to Halo2 to avoid circuit-specific trusted setup and to improve upgradability. 

## Account Based Solutions

Monero and ZCash are both unique to the UTXO model based blockchains, which are not compatible for account based blockchains where every smart contract application holds some states themselves. The most promising solution proposed is Zether, which took the approach of encryption instead of commitment.

### Zether

In rough sketches, Zether was designed to keep account balance encrypted using Twisted Elgamal encryption and exposes methods for state transitions such as deposit and withdrawal through proofs.

What is Twisted Elgamal encryption? It consists of three algorithms:  

1. $keygen \rightarrow (sk, pk = g^{sk})$: $pk$ is used to encrypt and $sk$ is used to decrypt. 

2. $encrypt(pk, b)$: First the user samples a blinding factor $r$, the ciphertext is $(c_1, c_2) = (g^r, g^b*pk^r)$. 

3. $decrypt(sk, (c_1, c_2))$: First compute $g^b = c_2/c_1^{sk}$ then brutal-force to compute $b$. Later we show that we do not need to brute force the plain value of $b$, but just use range proof to prove its range.

Note that the encrypted ciphertext for balance $b$ here satisfies the homomorphic addition property (similar to Pedersen commitments). If the same $pk$ is used, one can add or subtract the encrypted balance $b$. This is how Zether keeps track of the balance for each ElGamal public key.

Zether also proposed a modified Bulletproof to be used along with $\sigma$ protocols to enable efficient proofs on algebraically-encoded values. This is less complex for Twisted Elgamal encrypted values compared with circuit-based Bulletproof and can be combined with ring signature for anonymity as well.

The Zether smart contract is defined with the following algorithms, suppose ZTH denotes the confidential token: 

1. Deposit ETH to ZTH (Fund Transaction): To fund an account with public key pk with b ZTH, one can send b ETH to the smart contract. ZSC generates an ElGamal encryption of b with randomness 0 (since b is anyway part of the transaction) and adds it to the encrypted balance associated with pk. 

2. Transfer ZTH to ZTH (Transfer Transaction): In order to transfer some $b$ amount of ZTH to a public key $pk'$ without revealing $b$ itself, one can encrypt $b$ under both $pk$ and $pk'$. A ZK proof is provided to show that the two ciphertexts are well-formed, they encrypt the same positive value, and the remaining balance associated with $pk$ is positive. 

3. Withdrawal ZTH to ETH (Burn transaction): Verifies a ZK proof that the sender has the $sk$. 

In addition, one can lock and unlock ZTH balance to a given smart contract account. This introduces the interoperability by keeping a locked balance to a designated smart contract. Zether will process only those transactions on accounts that come from the given smart contract till its unlocked. 

# Centralized Dealer 

There are other privacy enhancements involving a dealer or smart contract to facilitate the payment. This usually requires many users to be online and there may be a delay to reclaim coins of the payment. It is also possible for the operator to trace or steal coins, or even be subject to legal liability.

## Coinjoin

[Coinjoin](https://bitcointalk.org/index.php?topic=279249.0) is a mechanism to create bitcoin transactions with multiple senders and multiple receivers. For a transaction with multiple inputs, the signature for each input is completely independent of each other. This means that it's possible for Bitcoin users to agree on a set of inputs to spend, and a set of outputs to pay to, and then to individually and separately sign a transaction and later merge their signatures. This is also combined with Maxwell's CT protocol to aggregate transactions in MimbleWimble/Grin. 

## Tornado cash

Tornado cash is deployed as a smart contract on Ethereum that a user can deposit and withdraw coins committed to a Merkle tree maintained by the smart contract. When spending the deposits, the user submits a zero knowledge proof proving the membership of the commitment, as well as knowledge of the nullifier and the secret of the commitment. In details, the deposit and withdrawal works as:

1. Deposit: User generates $k$ (nullifier) and $r$ (secret). submit transaction with $C = H(k || r)$. insert $C$ to into the leftmost zero leaf in the Merkle tree (changes 20 elements in the tree).

2. Withdrawal:
- User computes Merkle proof $O$ ($C$ is in tree with root $R$)
- User computes nullifier hash $h = H(k)$
- User computes proof $\pi$ with private inputs $(l, k, r, O)$ such that $h = H(k)$ and $O$ is the $l$-th leaf $C = H(k || r)$. done using groth16 as the zkSNARK proving system.

# References

1. Maxwell's CT: https://web.archive.org/web/20150628230410/https://people.xiph.org/~greg/confidential_values.txt
2. Confidential asset paper: https://blockstream.com/bitcoin17-final41.pdf
3. Monero explainer: https://web.getmonero.org/library/Zero-to-Monero-2-0-0.pdf
4. ZCash spec: https://zips.z.cash/protocol/protocol.pdf
5. ZCash original paper: https://eprint.iacr.org/2014/349.pdf
6. Bulletproof original paper: https://eprint.iacr.org/2017/1066.pdf
7. Zether paper: https://eprint.iacr.org/2019/191.pdf
8. Tornado cash explained: https://www.zellic.io/blog/how-does-tornado-cash-work
