# Bitcoin clients types and security models

Many of you have heard about light clients and full nodes and know at most what
the difference lay in.

I have Participated to a very interesting workshop in Stanford for Scaling Bitcoin
and our coordinator Tadjie Dryja has done an excellent job in
laying down the current situation made of many more nuances I knew of.

These Nuances lead to different trade offs between security and performance
and relay on what your client actually know about the actual state of
the ledger.

## Full Validating Node

At the time of writing, the blockchain is 170 Gbytes in size and the UTXO set
is 4.5 Gbytes.

The IBD (Initial blocks Downloads and validation) takes 4 hours to almost a day
according to experts but for me has always been much more (beyond four days).

In normal operation, a fully validating node takes [insert here] Gbytes/month
of data from your bandwith, to:

* relay transactions and blocks
* to serve blocks requested for IBD by other nodes
* to serve filtered blocks and transactions for SPV Clients

These resources are considered one of the biggest obstacle to the installation
and use of a full node.

## But why should I run a fully validating node?

The simplest explanation I have read is the one reported by Doctor Adam Back when he
says that the full nodes are like the anti counter-fitting machines you can
find in many shops: for one, they ensure the merchant he is receiving good money
but also they help in maintaining the system clean of fake money in general.

"Fake bitcoins", meaning in this context a transaction that tries to spend
bitcoins created in an invalid way or to spend coins already spent in a previous
transaction, are refused by full nodes as soon as the transaction tries to 
approach the first one in the propagation phase.

A full validating node when receives the transaction performs the security
checks necessary to veriy that the transaction involves sound money before
putting it in the [mempool] and relaying it to the rest of the network.

### The full history of all the transactions

In Bitcoin, money come to life and exist only in transactions.
The node is able validate the incoming money because the node has a copy of 
all the legit transactions which have appeared in the network since inception 
included those in which all the bitcoins come into existence ([coinbase] transactions).

Transactions in Bitcoin take the money from previous transactions and allocate them 
in transaction outputs. If these outputs are not used in other transactions, they are 
"unspent". The entire set of all the NON spent Transaction outputs is called UTXO 
(Unspent Transaction Output).

Since a transaction output is considered unspent until is used in a subsequent
transaction in which is referred as input, is necessary to have the whole
transaction history to judge which transaction outputs are effectively unspent
so to build the UTXO set.

The UTXO set is the real kernel of the system because the most important checks
a full node do are:

* Is the signature on the transaction valid? (power of spend)
* are money created into/during the transaction? (accounting check)
* Are the funds being spent existent and unspent before? (double spend check)

This last check requires is the one requiring a legit copy of UTXO set.

The UTXO set must be shared by all the computers of the network because is the
essence of the Bitcoin economic system.

NO ONE LEGIT SATOSHI EXISTS OUT OF THE UTXO SET.

## Well. Why then we cannot just keep just the updated UTXO set?

The good news is that in a certain way we can, and various of the "nuances" we
talked about during the workshop are based on the assumption that if you have a
**valid** and up-to-date "map" of all the legit money in circulation, you don't
need all the documentation about how this money have been generated and moved
to its actual position.

Problem is that you cannot state that you have a valid copy of the UTXO set if
you haven't validated each single transaction that has contributed to create it.
This is of course because the UTXO set is the result of the history of all
the bitcoins from their creation to their latest *destination* (the last output). 

This is exactly the trade off:

*You have to trust that your copy of the UTXO set
is legit or to **trust** whoever give it to you, unless you have built it yourself
after having validated all the transactions in the world.*

## but could I drop all the huge history of the transactions after having used it to build my legit UTXO set?

This is exactly what a **pruned node** does.

## what do I loose if I run a pruned node?

For sure you loose the possibility to build again your UTXO set without
downloading and re-validating the entire blockchain in case the chainstate
database get corrupted.

Also, from an altruistic point of view, you loose the possibility to serve
the blocks or the single transaction to other nodes and to SPV clients.
This last consideration could be mitigated in the future by a project which
will allow pruned nodes to serve the *part* of the blockchain they haven't
pruned and still possess.

## Is there a way to know the UTXO set is legit without rebuilding and validating the whole history?

One way would be to relay on an improved blockchain in which the headers
commit not only to the transactions in the block but also to the whole UTXO set. It is of course a different type of
validation because one would rely on the merkle root in the header of the block instead of personally build and verify the history but as long you choose to trust the header (for instance because it has the right proof of work), this could be considered enough.

These proposal go under the name of UTXO commitments.

What you trust here is not your own calculation of the UTXO set but the fact
that the copy in your possess (in whatever way you have obtained it) is the
same the miners has certified in the latest block.
You can of course also prove a payment is valid providing the merkle proof
that the particular output you are spending was in the UTXO set and result
spent in the subsequent block (this last proof would require the output
ordering and the providing of the proof of the preceding and the subsequent
output).

Obviously these operations are a burden but they can be shared between wallets
and mining nodes for instance by requiring the wallets to manage their part
of the UTXO set and to provide merkle proofs of their spending to the miners
along with the transaction. Rusty call these UTXO proofs.
