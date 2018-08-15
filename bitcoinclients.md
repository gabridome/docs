# Bitcoin Trustlessness and different wallet types

Many of you have heard about light clients and full nodes and know at most what
the difference lay in.

I have Participated to a very interesting workshop in Stanford for Scaling Bitcoin
and our coordinator, Tadjie Dryja has done an excellent job in
laying down the current situation made of many more nuances I knew of.

These Nuances lead to different trade offs between security and performance
and relay on what your client actually know about the actual state of
the ledger.

## Full Validating Node

"A [full node][fullNode] is a program that fully validates transactions and blocks.".

The first types of wallet we examine is the one that can be activated with a Full 
Validating Node.

The simplest explanation I have read is the one reported by Doctor Adam Back when he
says that the full nodes are like the anti counter-fitting machines you can
find in many shops: for one, they ensure the merchant that he is receiving good money,
but also they help in maintaining the system clean of fake money in general.

While a wallet software main purpose should be the correct management of the personal 
finance of the user, For a full validating node, the wallet component is a by-product 
of its duty of keeping the system healthy and serve a correct version state of the 
Bitcoin economy, it can also be used by the user to manage his money.

To do this in a trustless way, the node must have [downloaded and verified ALL the 
transactions occurred in the Bitcoin network][IBD] and from then on, it receives all the new 
transactions and blocks and verify them.

It verifies the correctness of the birth of every bitcoin and the passage from a [Transaction 
Output][TXO] to the next until the [Transaction Output from which it is not been spent yet][UTXO].
The verified UTXO set is the map of where all the bitcoin in circulation are and of 
what is required to spend them. 

When a full validating node wreceives the transaction performs the security
checks necessary to verify that the transaction involves sound money before
putting it in the [mempool] and relaying it to the rest of the network:

* Is the signature on the transaction valid? (power of spend)
* are money created into/during the transaction? (accounting check)
* Are the funds being spent existent and unspent before? (double spend check)

This last check requires is the one requiring a legit copy of UTXO set.

"Fake bitcoins", meaning in this context those resulting from a transaction that 
tries to spend bitcoins created in an invalid way or to spend coins already 
spent in a previous transaction, are refused by full nodes as soon as the 
transaction tries to approach the first one in the propagation phase.

>*A wallet which is integrated with a full validating node have the maximum security 
and truslessness because it relies on the global informations of the Bitcoin economy*

In normal operation, a fully validating node takes [20 Gbytes/month][MinimumRequirements]
of data from your bandwith, to:

* relay transactions and blocks
* to serve blocks requested for IBD by other nodes
* to serve filtered blocks and transactions for SPV Clients

At the time of writing, the blockchain is 170 Gbytes in size and the UTXO set
is 2.6 Gbytes.

These resources are considered one of the biggest obstacle to the installation
and use of a full node and they are not suited to be used in low resources 
devices like mobile phones. Also to have the whole image of the Bitcoin economy, 
past and present, to manage the wealth and transactions of a single user could be 
seen as a little too much. For these reason through the years some alternative 
has been explorated.

## The Pruned Node and the importance of the UTXO set

From the single user perspective, the verification of the incoming money is the 
most important task a wallet must perform. To be sure to receive good money, at 
very least it is necessary to verify:

* that the money come from the legit UTXO set (money in circulation)
* that the user has correctly demonstrated the right to spend those bitcoins
* that no money has been created inside the transaction
* that he is not trying to spend those money ALSO somewhere else (double spending)

For these checks there is no a real need to have the whole history of the bitcoin 
transactions.

* The UTXO set is sufficient to check the origin of money
* to be a node and to observe the transactions on the network

helps avoiding the risk of double spending.

This is what a pruned node does. It builds the UTXO set in the radical way by 
downloading and verifing all the transaction history ([IBD]), but then it pruned it.

*You have to trust that your copy of the UTXO set is legit or to **trust** 
whoever give it to you, unless you have built it yourself after having 
validated all the transactions in the world.*

>A pruned node can be considered an excellent wallet foundation because it monitors 
the network against double spending and verify the origin of the money with a legit 
UTXO set.

### What do I loose if I run a pruned node?

For sure you loose the possibility to build again your UTXO set without
downloading and re-validating the entire blockchain in case the chainstate
database get corrupted.

Also, from an altruistic point of view, you loose the possibility to serve
the blocks or the single transaction to other nodes and to SPV clients
simply because you don't keep it.


## Is there a way to know the UTXO set is legit without rebuilding and validating the whole history?

One way would be to relay on an improved blockchain in which the headers of 
the blocks commit not only to the transactions in the block but also to the 
whole UTXO set. 

It is of course a different type of validation because one would rely 
on the merkle root in the header of the block or inside the coinbase 
transaction instead of personally build and verify the history but, 
as long you choose to trust the header because it embeds the right amount 
of proof of work and/or because you can get the same header from other 
peers, this could be considered enough.

These proposals go under the name of UTXO commitments.
What you trust here is not your own calculation of the UTXO set but the fact
that the copy in your possess (in whatever way you have obtained it) is the
same the miners have certified with a good amount of proof of work in the 
latest block.

There also a [proposal][TXOCommitments] by Peter Todd in which the node or 
the wallet keeps all the TXOs (spent and unspent) in [Merkle Mountain Ranges][MMR] 
with a serie of caching mechanisms to help the intense cpu and I/O overhead 
necessary to keep the system in synch.

[In an other proposal]by Bram Cohen, the payer can also can also prove a 
payment is valid providing the proof that the particular output he is spending 
was in the UTXO set and result spent in the subsequent block. 
One way this last proof of non-presence could be supplied by keeping a merkle 
root of an ordered and numbered UTXO set. By providing the proof of the 
presence of preceding and subsequent UTXO, the payee may verify the non 
presence of the relevant UTXO. In this particular case the payee may also 
avoid keeping a copy of the UTXO set. [This idea][UTXOProofs] also imply the presence of a [bitfield][TXOBitfields] 
associated with each TXO. There an interesting [comparison between the two 
proposals](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2016-June/012758.html)
made by Bram Cohen. 

Obviously these operations are a burden but they can be shared between wallets
and mining nodes for instance by requiring the wallets to manage their part
of the UTXO set and to provide merkle proofs of their spending to the miners
along with the transaction. Rusty call these UTXO proofs.

>Future wallets which will relay on UTXO commitments and UTXO proofs 
will have in general a good security based on the proof of work embedded in 
the headers or in the contents of the blockchain.


## UHS

An other [proposal][UHS] which goes in this direction is Cory Fields' UHS, in which 
the receiver could have **the same security of an UTXO based wallet without keeping the 
whole UTXO set but just a set composed by an hash for each UTXO (UHS)**.
This hash is obtained by some agreed upon data from the preceding transaction 
output to verify the "power to spend" concatenated with the ID of the UTXO the outpoint. 
If this interesting proposal is adopted, future clients would store just 
the UHS which weight about half the UTXO set and verify in a very fast way 
the incoming transaction, by checking the existence of the hash of the data supplied 
by the payer in the UHS (after or while verifying the "power to spend").

## SPV validation

In the [SPV Validation Model][SPVValidation] The client to perform the validation:

* keeps all the block headers of the Blockchain
* request the transaction that want to validate to a node
* request the merkle proof that the transaction has been included into a certain block

This validation approach is vulnerable to two main weaknesses:

### the node could say the transaction doesn't exist in a block and don't provide the merkle proof(lying)

This could be mitigated by connecting and asking to more than one client but also this could be subject to [Network partitioning][NetworkPartitioning] or a [Sybil attack][SybilAttack].


[fullNode]: https://btcinformation.org/en/full-node
[MinimumRequirements]: https://btcinformation.org/en/full-node#minimum-requirements
[IBD]: https://btcinformation.org/en/glossary/initial-block-download
[TXO]: https://btcinformation.org/en/glossary/output
[UTXO]: https://btcinformation.org/en/glossary/unspent-transaction-output
[TXOCommitments]: https://petertodd.org/2016/delayed-txo-commitments
[MMR]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-February/013592.html
[UTXOProofs]: https://diyhpl.us/wiki/transcripts/sf-bitcoin-meetup/2017-07-08-bram-cohen-merkle-sets/
[TXOBitfields]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-March/013928.html
[UHS]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-May/015967.html
[SPVValidation]: https://btcinformation.org/en/developer-guide#simplified-payment-verification-spv
[NetworkPartitioning]: https://news.ycombinator.com/item?id=14594172
[SybilAttack]: https://en.wikipedia.org/wiki/Sybil_attack
