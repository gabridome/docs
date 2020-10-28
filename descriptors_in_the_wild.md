# Descriptors in the wild

Output Descriptors has brought new hopes for those who, like me, would like to see 
a greater interoperability in Bitcoin among different applications.

Toghether with an other innovation, PSBT, they allow to set up and use multi signature wallets, make coinjoins, use different harware wallets with different software wallets
and other things that I still don't know.

Like an explorator in a wild land, I have tried to observe Descriptors in two different 
"habitats" and see if they really can live in both without modifications bringing a lot
of interoperability to the whole environment.

Also, I have tried to use this interoperability to do something useful: build a multi 
signature multi software wallet.

Immagine Alice using Bitcoin Core as a Wallet and willing to put some bitcoin in common
with Bob who is using a "Last generation" wallet, Magical Bitcoin Wallet, which uses 
descriptors and miniscript natively.

With descriptors, we can set up a common multisignature wallet based on descriptors and
utilize PSBT to pass along the transaction to complete the workflow to spend the bitcoin 
in common.

In this post all the necessary step are illustrated to serve as a guide to avoid many 
possible mistakes.
We will build a 2of2 key set up that will be used cooperatively by Bitcoin Core and Magical Bitcoin Wallet.
We need:
* Magical Bitcoin Wallet
* Bitcoin Core
* Pycoin ku utility

1. We build an Extended Private Master Key for both wallet and we derive a BIP84 Extended Master Public.

```
# new Extended wallet data
#
export core_key=$(ku -n XTN -j create)
echo $core_key
# New Extended Master Private
#
export core_xprv=$(echo $core_key|jq -r '.wallet_key')
# Derived Extended Pubblic key
#
export core_xpub_84=$(ku -j -s 84H/0H/0H $core_xprv |jq -r '.public_version')
export core_fingerprint=$(echo $core_key|jq -r '.fingerprint')
echo $core_fingerprint
#
# Now I build the Extended public key to be communicated to Magical wallet's owner.
#
export core_xpub_84_for_rec_desc="[$core_fingerprint/84'/0'/0']$core_xpub_84/0/*"
export core_xpub_84_for_chg_desc="[$core_fingerprint/84'/0'/0']$core_xpub_84/1/*"
env |grep core_
```

In Magical we do the same:

```
# new Extended wallet data
export magical_key=$(ku -n XTN -j create)
# New Extended Master Private
export magical_xprv=$(echo $magical_key|jq -r '.wallet_key')
# Derived Extended Pubblic key
export magical_xpub_84=$(ku -j -s 84H/0H/0H $magical_xprv |jq -r '.public_version')
export magical_fingerprint=$(echo $magical_key|jq -r '.fingerprint')
# Now I build the xpub to be communicated.
export magical_xpub_84_for_rec_desc="[$magical_fingerprint/84'/0'/0']$magical_xpub_84/0/*"
export magical_xpub_84_for_chg_desc="[$magical_fingerprint/84'/0'/0']$magical_xpub_84/1/*"

env |grep magical_
```

2. We exchange the BIP84 Extended Master Public and we build a descriptor for each wallet:

To build a multisig wallet, each wallet owner must add his derived extended private key AND all
the extended public key of the other wallets involved in the multi signature set up in a newly 
created multi signature descriptor.

In our case, the multi signature descriptor for Bitcoin Core will be composed with:

* The BIP84 derived xpub from Magical Bitcoin wallet
* The BIP84 derived xprv from Core. 

Magical wallet's owner will send to Core's the derived xpub for this purpose.
This how the Core's multisig descriptor will look like

`export core_rec_desc="wsh(multi(2,$magical_xpub_84_for_rec_desc,$core_xprv/84'/0'/0'/0/*))"`

Where of course `$magical_xpub_84_for_rec_desc`is the derived master public created in Magical and 
received by Core's owner.

We add the necessary checksum using the specific `bitcoin-cli` call.

`export core_rec_desc_chksum=$core_rec_desc#$(bitcoin-cli -testnet getdescriptorinfo $core_rec_desc|jq -r '.checksum')`

We repeat the same to build the descriptor to receive the change.

```
export core_chg_desc="wsh(multi(2,$magical_xpub_84_for_chg_desc,$core_xprv/84'/0'/0'/1/*))"
export core_chg_desc_chksum=$core_chg_desc#$(bitcoin-cli -testnet getdescriptorinfo $core_chg_desc|jq -r '.checksum')
```

Ready to import the descriptor into Core:


3. For Core we create an experimental empty Descriptor Wallet and we import the descriptors created for this wallet.

```
bitcoin-cli -testnet createwallet "multisig2of2withmagical" false true "" false true false
bitcoin-cli -testnet -rpcwallet=multisig2of2withmagical importdescriptors "[{\"desc\":\"$core_rec_desc_chksum\",\"timestamp\":\"now\",\"active\":true,\"internal\":false},{\"desc\":\"$core_chg_desc_chksum\",\"timestamp\":\"now\",\"active\":true,\"internal\":true}]"
export first_address=$(bitcoin-cli -testnet -rpcwallet=multisig2of2withmagical getnewaddress)
echo $first_address
```
3. For Magical we set the derivation for receiving addresses and change addresses in the command line (maybe setting an alias)

Building the descriptor:

`export magical_rec_desc="wsh(multi(2,$magical_xprv/84'/0'/0'/0/*,$core_xpub_84_for_rec_desc))"`

Please note that the order of the extended key in the descriptor MUST be the same in the 2 wallets.
We have chosen to put Magical first. In alternative, it could be safer by chosing a `soretedmulti` multisignature setup.

```
export magical_rec_desc_chksum=$magical_rec_desc#$(bitcoin-cli -testnet getdescriptorinfo $magical_rec_desc|jq -r '.checksum')
export magical_chg_desc="wsh(multi(2,$magical_xprv/84'/0'/0'/1/*,$core_xpub_84_for_chg_desc))"
export magical_chg_desc_chksum=$magical_chg_desc#$(bitcoin-cli -testnet getdescriptorinfo $magical_chg_desc|jq -r '.checksum')
```

To take a look at the variables:
```
env |grep core_
env |grep magical_
```

Now we use Magical to get the first multisig address:

`repl -d "$magical_rec_desc_chksum" -c "$magical_chg_desc_chksum" -n testnet -w $magical_fingerprint get_new_address`

Et voilà: the newly created address in Core is the same of the newly created address in Magical. this is part of the miracle
of descriptors' interoperability.

6. we ask for testnet coins giving the first created address.
I Have already a multisig account in Core so I can do

```
export psbt=$(bitcoin-cli -testnet -rpcwallet=coremultisig2of2 walletcreatefundedpsbt "[]" "[{\"$first_address\":0.00002}]"|jq -r '.psbt')
export psbt=$(bitcoin-cli -testnet -rpcwallet=coremultisig2of2 walletprocesspsbt $psbt|jq -r '.psbt')
export tx=$(bitcoin-cli -testnet -rpcwallet=coremultisig2of2 finalizepsbt $psbt|jq)
export tx=$(bitcoin-cli -testnet -rpcwallet=coremultisig2of2 finalizepsbt $psbt|jq -r '.hex')
bitcoin-cli -testnet sendrawtransaction $tx
# after the first confirmation:
bitcoin-cli -testnet -rpcwallet=multisig2of2withmagical getbalance "*" 1
```

6. we send satoshis to this address and we check its arrival in the two wallets.

```
export psbt=$(bitcoin-cli -testnet -rpcwallet=multisig2of2withmagical walletcreatefundedpsbt "[]" "[{\"tb1qrcesfj9f2d7x40xs6ztnlrcgxhh6vsw8658hjdhdy6qgkf6nfrds9rp79a\":0.000012}]"|jq -r '.psbt')
bitcoin-cli -testnet -rpcwallet=multisig2of2withmagical walletprocesspsbt $psbt
{
  "psbt": "cHNidP8BAIkCAAAAATj90EC+NAuXj7y6SseZJucoJM6sGnUcVm9koTveZECTAAAAAAD+////AmACAAAAAAAAIgAg98ol9j4AalD71E0mV5QV0uM6/vCT+pi2twxr/zrvLROwBAAAAAAAACIAIB4zBMipU3xqvNDQlz+PCDXvpkHH1Q95Nu0mgIsnU0jbAAAAAAABAIkCAAAAAQS+ObgGG6UwtvaO3KYph2E3/ws7Q83RbmR3rxC0fKYSAQAAAAD+////AtAHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNDAHQAAAAAAACIAIBQpiDTgPIMt0ld8cmuYqlY+EIPjvrmMqZruDhs61hQNAAAAAAEBK9AHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNAiAgNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfUcwRAIgS6x0i1J1HRzllIPf4WlFY+Dl8kCCLK81TL2djZxTFXMCICJVBKkKNxu1w1mRVor6iFTSVXiJjmWwBXVeJLISvBwAAQEFR1IhArn3tec7n7318rnWqf0dIIwtLtfxo6Zt0HV70UvZYaWvIQNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfVKuIgYCufe15zufvfXyudap/R0gjC0u1/Gjpm3QdXvRS9lhpa8YNEw2cFQAAIAAAACAAAAAgAAAAAAAAAAAIgYDbdI+wHtIgO6pS6Ja7jai1pDwPeidKoDCytTO1jhjAH0YO/laXFQAAIAAAACAAAAAgAAAAAAAAAAAAAEBR1IhAqccvA3rL13D1K4GeWjcahDsO3P8oaVNBttk4MlCKXIcIQLHKhjmPuCQjyS77ZfaMN2tdgNKcf/+57VXGZhz/UWTl1KuIgICpxy8DesvXcPUrgZ5aNxqEOw7c/yhpU0G22TgyUIpchwYNEw2cFQAAIAAAACAAAAAgAEAAAADAAAAIgICxyoY5j7gkI8ku+2X2jDdrXYDSnH//ue1VxmYc/1Fk5cYO/laXFQAAIAAAACAAAAAgAEAAAADAAAAAAA=",
  "complete": false <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
}
```

Exactly. We have processed the transaction with Core but we miss in the wallet one of the 
necessary key of the multisig 2of2 setup. The one contained inside Magical.

The psbt is sent over to the Magical wallet owner who tries to sign the transaction.

```
repl -d "$magical_rec_desc_chksum" -c "$magical_chg_desc_chksum" -n testnet -w $magical_fingerprint sign --psbt "cHNidP8BAIkCAAAAATj90EC+NAuXj7y6SseZJucoJM6sGnUcVm9koTveZECTAAAAAAD+////AmACAAAAAAAAIgAg98ol9j4AalD71E0mV5QV0uM6/vCT+pi2twxr/zrvLROwBAAAAAAAACIAIB4zBMipU3xqvNDQlz+PCDXvpkHH1Q95Nu0mgIsnU0jbAAAAAAABAIkCAAAAAQS+ObgGG6UwtvaO3KYph2E3/ws7Q83RbmR3rxC0fKYSAQAAAAD+////AtAHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNDAHQAAAAAAACIAIBQpiDTgPIMt0ld8cmuYqlY+EIPjvrmMqZruDhs61hQNAAAAAAEBK9AHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNAiAgNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfUcwRAIgS6x0i1J1HRzllIPf4WlFY+Dl8kCCLK81TL2djZxTFXMCICJVBKkKNxu1w1mRVor6iFTSVXiJjmWwBXVeJLISvBwAAQEFR1IhArn3tec7n7318rnWqf0dIIwtLtfxo6Zt0HV70UvZYaWvIQNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfVKuIgYCufe15zufvfXyudap/R0gjC0u1/Gjpm3QdXvRS9lhpa8YNEw2cFQAAIAAAACAAAAAgAAAAAAAAAAAIgYDbdI+wHtIgO6pS6Ja7jai1pDwPeidKoDCytTO1jhjAH0YO/laXFQAAIAAAACAAAAAgAAAAAAAAAAAAAEBR1IhAqccvA3rL13D1K4GeWjcahDsO3P8oaVNBttk4MlCKXIcIQLHKhjmPuCQjyS77ZfaMN2tdgNKcf/+57VXGZhz/UWTl1KuIgICpxy8DesvXcPUrgZ5aNxqEOw7c/yhpU0G22TgyUIpchwYNEw2cFQAAIAAAACAAAAAgAEAAAADAAAAIgICxyoY5j7gkI8ku+2X2jDdrXYDSnH//ue1VxmYc/1Fk5cYO/laXFQAAIAAAACAAAAAgAEAAAADAAAAAAA="
{
  "is_finalized": true,
  "psbt": "cHNidP8BAIkCAAAAATj90EC+NAuXj7y6SseZJucoJM6sGnUcVm9koTveZECTAAAAAAD+////AmACAAAAAAAAIgAg98ol9j4AalD71E0mV5QV0uM6/vCT+pi2twxr/zrvLROwBAAAAAAAACIAIB4zBMipU3xqvNDQlz+PCDXvpkHH1Q95Nu0mgIsnU0jbAAAAAAABAIkCAAAAAQS+ObgGG6UwtvaO3KYph2E3/ws7Q83RbmR3rxC0fKYSAQAAAAD+////AtAHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNDAHQAAAAAAACIAIBQpiDTgPIMt0ld8cmuYqlY+EIPjvrmMqZruDhs61hQNAAAAAAEBK9AHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNAiAgNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfUcwRAIgS6x0i1J1HRzllIPf4WlFY+Dl8kCCLK81TL2djZxTFXMCICJVBKkKNxu1w1mRVor6iFTSVXiJjmWwBXVeJLISvBwAASICArn3tec7n7318rnWqf0dIIwtLtfxo6Zt0HV70UvZYaWvRzBEAiBkVDLgVEwvENnLx+04o7gGpGjFDBwAXTJmf8Yvo35oygIgbuBkHsvPC9jmZcMZ9P+Pwp01yxSaWo+5feyPmd3ai1kBAQVHUiECufe15zufvfXyudap/R0gjC0u1/Gjpm3QdXvRS9lhpa8hA23SPsB7SIDuqUuiWu42otaQ8D3onSqAwsrUztY4YwB9Uq4iBgNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfRg7+VpcVAAAgAAAAIAAAACAAAAAAAAAAAAiBgK597XnO5+99fK51qn9HSCMLS7X8aOmbdB1e9FL2WGlrxg0TDZwVAAAgAAAAIAAAACAAAAAAAAAAAABBwABCNoEAEcwRAIgZFQy4FRMLxDZy8ftOKO4BqRoxQwcAF0yZn/GL6N+aMoCIG7gZB7LzwvY5mXDGfT/j8KdNcsUmlqPuX3sj5nd2otZAUcwRAIgS6x0i1J1HRzllIPf4WlFY+Dl8kCCLK81TL2djZxTFXMCICJVBKkKNxu1w1mRVor6iFTSVXiJjmWwBXVeJLISvBwAAUdSIQK597XnO5+99fK51qn9HSCMLS7X8aOmbdB1e9FL2WGlryEDbdI+wHtIgO6pS6Ja7jai1pDwPeidKoDCytTO1jhjAH1SrgABAUdSIQKnHLwN6y9dw9SuBnlo3GoQ7Dtz/KGlTQbbZODJQilyHCECxyoY5j7gkI8ku+2X2jDdrXYDSnH//ue1VxmYc/1Fk5dSriICAqccvA3rL13D1K4GeWjcahDsO3P8oaVNBttk4MlCKXIcGDRMNnBUAACAAAAAgAAAAIABAAAAAwAAACICAscqGOY+4JCPJLvtl9ow3a12A0px//7ntVcZmHP9RZOXGDv5WlxUAACAAAAAgAAAAIABAAAAAwAAAAAA"
}
```
The signature has succeded and now we can broadcast the transction.
```
repl -d "$magical_rec_desc_chksum" -c "$magical_chg_desc_chksum" -n testnet -w $magical_fingerprint broadcast --psbt "cHNidP8BAIkCAAAAATj90EC+NAuXj7y6SseZJucoJM6sGnUcVm9koTveZECTAAAAAAD+////AmACAAAAAAAAIgAg98ol9j4AalD71E0mV5QV0uM6/vCT+pi2twxr/zrvLROwBAAAAAAAACIAIB4zBMipU3xqvNDQlz+PCDXvpkHH1Q95Nu0mgIsnU0jbAAAAAAABAIkCAAAAAQS+ObgGG6UwtvaO3KYph2E3/ws7Q83RbmR3rxC0fKYSAQAAAAD+////AtAHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNDAHQAAAAAAACIAIBQpiDTgPIMt0ld8cmuYqlY+EIPjvrmMqZruDhs61hQNAAAAAAEBK9AHAAAAAAAAIgAg6GXadcNj7k4yKUbnVlTLiedXQFXYdCBoNygop/PISNAiAgNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfUcwRAIgS6x0i1J1HRzllIPf4WlFY+Dl8kCCLK81TL2djZxTFXMCICJVBKkKNxu1w1mRVor6iFTSVXiJjmWwBXVeJLISvBwAASICArn3tec7n7318rnWqf0dIIwtLtfxo6Zt0HV70UvZYaWvRzBEAiBkVDLgVEwvENnLx+04o7gGpGjFDBwAXTJmf8Yvo35oygIgbuBkHsvPC9jmZcMZ9P+Pwp01yxSaWo+5feyPmd3ai1kBAQVHUiECufe15zufvfXyudap/R0gjC0u1/Gjpm3QdXvRS9lhpa8hA23SPsB7SIDuqUuiWu42otaQ8D3onSqAwsrUztY4YwB9Uq4iBgNt0j7Ae0iA7qlLolruNqLWkPA96J0qgMLK1M7WOGMAfRg7+VpcVAAAgAAAAIAAAACAAAAAAAAAAAAiBgK597XnO5+99fK51qn9HSCMLS7X8aOmbdB1e9FL2WGlrxg0TDZwVAAAgAAAAIAAAACAAAAAAAAAAAABBwABCNoEAEcwRAIgZFQy4FRMLxDZy8ftOKO4BqRoxQwcAF0yZn/GL6N+aMoCIG7gZB7LzwvY5mXDGfT/j8KdNcsUmlqPuX3sj5nd2otZAUcwRAIgS6x0i1J1HRzllIPf4WlFY+Dl8kCCLK81TL2djZxTFXMCICJVBKkKNxu1w1mRVor6iFTSVXiJjmWwBXVeJLISvBwAAUdSIQK597XnO5+99fK51qn9HSCMLS7X8aOmbdB1e9FL2WGlryEDbdI+wHtIgO6pS6Ja7jai1pDwPeidKoDCytTO1jhjAH1SrgABAUdSIQKnHLwN6y9dw9SuBnlo3GoQ7Dtz/KGlTQbbZODJQilyHCECxyoY5j7gkI8ku+2X2jDdrXYDSnH//ue1VxmYc/1Fk5dSriICAqccvA3rL13D1K4GeWjcahDsO3P8oaVNBttk4MlCKXIcGDRMNnBUAACAAAAAgAAAAIABAAAAAwAAACICAscqGOY+4JCPJLvtl9ow3a12A0px//7ntVcZmHP9RZOXGDv5WlxUAACAAAAAgAAAAIABAAAAAwAAAAAA"
{
  "txid": "a0b082e3b0579822d4a0b0fa95a4c4662f6b128ffd43fdcfe53c37473ce85dee"
}
```
