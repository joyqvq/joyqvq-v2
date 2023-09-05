---
title: A Tale of Privacy-focused Transactions
date: 2023-09-04
tags: ["crypto"]
---
## A Tale of Privacy-focused Transactions

[WIP]

Blockchains had been developed under one public verifiable state machine, where all transactions are sequenced globally under consensus. While people had enjoyed the benefit of a consistent and shared state for building applications and payments, a side effect of the public veriability is that all of the transactions are viewable from the public ledger by any one. The privacy and security concerns of this property had posed many risks and hindered adpotion of cryptocurrencies, because of its direct contradition to other monetary instruments like cash. Insufficient privacy can also result in a loss of fungibility. 

In UTXO based currencies, this is partially addressed by generating a new receiving address for each transaction so that they are unlinkable, but eventually you want to spend some or many output that belongs to different addresses which makes everything linked into one transaction.  

To achieve the property of digital "cash", we desire a blockchain be able to maintain a global state, but without needing to post all state transition publicly. 

Generally speaking, we care about privacy in respect to some of the following aspects: 

1. Hiding sender and/or receiver: this is to achive anonymity. data privacy
2. Hiding the amount: we call this confidentiality, also part of data privacy

There are two general directions that privacy transactions solution take:

1. Direct P2P: This is to enable privacy trasaction on the protocol level. i.e. every miner or validator can verify the new signature type with privacy specific schemes. 

2. Centralized Intermediatary: this is to rely on a dealer to mix coins such that the out and in coins from the pool cannot be linked. 

Now we unpack the different approaches in details. Note that for examples mentioned below are a general description of the original paper and the protocol spec, but the exact implementation details deployed may have upgraded or extended signaficantly. 

# Direct Privacy Transaction

The existing blockchains mostly fall into two categories of data model: UTXO based or account based model. The former does not contain smart contract functionality and the coins (inputs, outputs) themselves hold all the information for transaction validity. A transaction consists of inputs, outputs and scriptSig. The most prominent examples include Bitcoin, ZCash.

On the other hand, account based blockchains encompasses Ethereum, Solana etc where the account itself holds smart contracts and states themselves. 

## UTXO Based

Each UTXO contains an amount and a public key; transaction destroys UTXOs and create new ones that are lesser than the total amount, along signatures provided by private key associated with the corresponding public key.

### Confidential Transaction (Maxwell)

The first version of confidential transaction was discussed in Maxwell's [post](https://web.archive.org/web/20150628230410/https://people.xiph.org/~greg/confidential_values.txt) and also this [paper](this [paper](https://blockstream.com/bitcoin17-final41.pdf)). The protocol replaces the explicit UTXO amount with a homomorphic commitment to the amount, and attaches a range proof that the output amount is committed to a range.

The commitment scheme used is Pederson Commitment constructed based on Elliptic Curve points. A commitment to value x is calculated as $comm(r, x) = xG + rH$ where r is a blinding factor that is randomly sampled, G is the generator for the group and H is an additional generator. To accept the commitment C, check if C = mH + rG when the r and m are revealed. 

The elliptic curve construction does already satisfy the additive property: $comm(r1, data1) + comm(r2, data2) == comm(r1 + r2, data1 + data2)$.

Since Bitcoin transaction is a set of input coins and a set of output coins, the commitment scheme gives us a very simple construction: If the author can picks the blinding factor properly, a transaction can be verified by checking:

1. Verifies the sum of outputs of a transaction equals to the sum of inputs: (sum of commitments of all inputs + the tx_amount*H) - (sum of commitments of all outputs + fees*H) == 0.

2. Verifiers the sum of outputs is larger than 0. 

3. (Unchanged from Bitcoin protocol) Signatures of inputs are verified. 

Step 2 makes use of rangeproof, The protocol is built upon an simple OR proof that commits to either 0 or 1:

- To commit to 0: I give you a signature of hash of commitment using the private key as the blinding factor. 

- To commit to 1: I give you C, and you compute C': C' = C - 1H. Then I provide a ring signature over {C, C'}. The ring signature here proves that the signer knows the discrete log of at least one of the pubkeys. This step is interactive. 

We now can decompose the value to be committed C in base-2 (in fact, any base): $C = C_1*2^0 + C_1*2^1 + .. + C_5*2^1$. In order to prove C is in an range $[0, 32)$, we can just prove that C_1...C_5 are committed to either 0 or 1 using the OR proof. 

Mimblewimble made an improvement to signature compressions. instead of providing signatures for each input, the sender can only provide one signature with the difference of the outputs and inputs as the public key.

Note that the size range proof itself can be greatly compressed thanks to [Bulletproof](https://eprint.iacr.org/2017/1066.pdf). This is an active research area and can provide efficiency to range proof used in confidential transaction as a drop in replacement. 

### Monero

Monero is a privacy transaction protocol deployed as an altcoin instead of on Bitcoin, but developed with similar confidential transaction from above:

1. Inputs are signed with Schnorr-style multilayered linkable spontaneous anonymous group signatures (a particular type of ring signature): The idea here is that you have a private key and several associated auxiliary keys. It is important to prove knowledge of all private keys, but linkability only applies to the primary. 

2. output amounts (communicated to recipients via ECDH) are concealed with Pedersen commitments and proven in a legitimate range with Bulletproofs

uses a special type of signature scheme to hide the origins and destinations of transactions among a set of UTXOs chosen by the sender (anonymity set)

size of signature increases linearly with the size of the anonymity set

//todo: https://web.getmonero.org/library/Zero-to-Monero-2-0-0.pdf

### ZCash

Senders and recipients are hidden among the group of people
who use shielded addresses. 

Both Monero and ZCash utilize a set of nullifiers which grows linear in the number of transactions. 

zkSNARK is used as an additional cryptosystem with pairing based cryptography. 

Instead, Zerocoin authenticates coins by proving, in zero-knowledge, that they belong to a public list of valid coins (which can be maintained on the block chain).

The simplest construction of a confidential transaction 

tx mint: coin = (r, sn, commit_r(sn))
tx spend: reveal sn, i know r such that comm(r) is in the list

issue: sn is revealed and cannot be transferred again
solution: extending this for direct anonymous payment
 
(sk, pk)
user samples \rho
set sn = prf_(sk, sn)(\rho)

our tx: rt, sn_1_old, sn_2_old, cm_1_new, cm_2_new, v_pub, info
addr = prf_x()
rt: root of merkle tree

from groth16 to halo

-  it
contains the serial numbers of the consumed coins, commitments of the created coins, and a zero knowledge
proof attesting that the serial numbers belong to coins created in the past 

revealing a coin's serial number ensures that a coin
cannot be consumed more than once (the same serial number would appear twice).

https://zips.z.cash/protocol/protocol.pdf
https://eprint.iacr.org/2014/349.pdf

## Account based

not compatible bc smart contract application holds some states

### Zether
keep account balance encrypted, methods to deposit and withdraw through proofs

range proof + twisted elgamal encryption

keygen: pk = g^sk

encrypt(pk, x): samples a blinding factor r, (c1, c2) = (g^r, g^x*pk^r)

decrypt(sk, (c1, c2)): g^x = c2/c1^sk

https://eprint.iacr.org/2019/191.pdf

### Stealth Address

# Centralized Dealer 

The above solutions usually require a protocol change to enable or subject to scalability issue, some faster solution can be used to rely on a trusted centralized dealer to mix coins. a central party of a mix with multisig tx involving many collaborating bitcoin uwallets. requires users to be online. There may be a delay to reclaim coins, and it is possible that the operator to trace or steal coin, or subject to legal liability. 

## Coinjoin
https://bitcointalk.org/index.php?topic=279249.0
https://en.bitcoin.it/wiki/CoinJoin

## Tornado cash

https://www.zellic.io/blog/how-does-tornado-cash-work


# References

1. Maxwell's CT: https://web.archive.org/web/20150628230410/https://people.xiph.org/~greg/confidential_values.txt
2. Confidential asset paper: https://blockstream.com/bitcoin17-final41.pdf
3. Monero explainer: https://web.getmonero.org/library/Zero-to-Monero-2-0-0.pdf
2. ZCash spec: https://zips.z.cash/protocol/protocol.pdf
2. ZCash original paper: https://eprint.iacr.org/2014/349.pdf
4. Bulletproof original paper: https://eprint.iacr.org/2017/1066.pdf
5. Zether paper: https://eprint.iacr.org/2019/191.pdf
