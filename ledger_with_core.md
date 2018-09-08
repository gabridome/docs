bitcoind -regtest
bitcoin-cli generate 102

```
./hwi.py -t "ledger" -d "IOService:/AppleACPIPlatformExpert/PCI0@0/AppleACPIPCI/EHC1@1D,7/EHC1@fd000000/PRT1@fd100000/IOUSBHostDevice@fd100000/AppleUSB20InternalHub@fd100000/AppleUSB20HubPort@fd140000/Nano S@fd140000/Nano S@0/IOUSBHostHIDDevice@fd140000,0" --testnet getxpub m/49h/1h/0h
=> b'e0c4000000'
<= b'1b30010208010003'9000
=> b'f026000000'
<= b''6d00
=> b'e04000000d03800000318000000180000000'
<= b'4104e0f45d003578817dffe1b8e32a60b79ce9faa0cf0715924288e955445fcabc2d5eccb89161c722b7ff61df8a2ad107fc3e8ad2d9e5c0c433f5a2470767b3a1cf226d794334344271586a58756d56514e5833376d5464325071547369544d7445414a3685ccb895c2f12e47d9bde2ba573774b9b175353c31f4f1c0e1f9053e4f12827d'9000
=> b'e040000009028000003180000001'
<= b'4104e08ecdbc92af7d98f11010f97317ccc74604f091014e69770c800b2f8b81e804f33270265dedf07240215c4d2257f2872bec03a1423bbea4bc23da9e7e9e7b25226d687046396f654b33757658707778684c3467316e544345704c6473437a46586e68b6f4df03e79406afab52bdd3a256cf83a0d11a130243e9187a092836f375338f'9000
{"xpub": "tpubDCD9NnZCwAYgQoYB6VqZ8huAaGNng8sZMRG7RyscnLeRW41QdyxmmMYjBhB9imfjZ72u67yxgNJLqJvZzpRDbaXRnvmokz2aSVcvY6MMaM3"}
```
Note: --testnet 
Path: getxpub m/49h/1h/0h 49h is for the purpose: p2sh-p2wpkh (BIP49). I cannot derive a native bech32 address for regtest.

```
$ ku -j -s0/0-5 tpubDCD9NnZCwAYgQoYB6VqZ8huAaGNng8sZMRG7RyscnLeRW41QdyxmmMYjBhB9imfjZ72u67yxgNJLqJvZzpRDbaXRnvmokz2aSVcvY6MMaM3|jq '.p2sh_segwit'
"2NFN1kLeJ4q4PXN8jZSbUcCi8qLRiDfm2B2"
"2NFcTKU3w4G8bgvuxP4dd6jCnTpJzqY1o2o"
"2N77EpHs8gmqubLkwjYfMFMd1f6kiAkShT7"
"2N93SdwskxEtoJdgCvKNjtMxGmN8wTGTtzh"
"2MxMn8cGSAMjjb7JF61mZoeRqrAxix8unwg"
"2N7S8yvxvhbFDM9QeLCLmmHSiRFCZhzPw9Z"
```

```
bitcoin-cli -regtest sendtoaddress "2NFN1kLeJ4q4PXN8jZSbUcCi8qLRiDfm2B2" 10
7b06b13740d8b4caa8eaaf0e50cac59948f27af2c55e597238d0bc430e83df6e
```

```
bitcoin-cli -regtest generate 1 # to confirm the transaction

$ bitcoin-cli -regtest gettransaction "7b06b13740d8b4caa8eaaf0e50cac59948f27af2c55e597238d0bc430e83df6e"
{
  "amount": -10.00000000,
  "fee": -0.00003720,
  "confirmations": 1,
  "blockhash": "72d517e17e819f22649bf708956afed67c9a9956f67680ff28546a662fbaf4b2",
  "blockindex": 2,
  "blocktime": 1536423253,
  "txid": "7b06b13740d8b4caa8eaaf0e50cac59948f27af2c55e597238d0bc430e83df6e",
  "walletconflicts": [
  ],
  "time": 1536422431,
  "timereceived": 1536422431,
  "bip125-replaceable": "no",
  "details": [
    {
      "address": "2NFN1kLeJ4q4PXN8jZSbUcCi8qLRiDfm2B2",
      "category": "send",
      "amount": -10.00000000,
      "vout": 0,
      "fee": -0.00003720,
      "abandoned": false
    }
  ],
  "hex": "020000000122a3deb6b1a1d72bc5681a126b567ca2c3b8ecfb70a9e8df3842f4606a3bf29b000000004847304402207fab7d7b62165e6eb4f7482bd8367a2fc3ec7b0652461375a60dc9842c20aef502203b9c4d40e0653ab4d3546b4d54d5d3c04aaff7f0c7fba5b33e05c50e731a4bc901feffffff0200ca9a3b0000000017a914f29b76104abfd586947a4a5379626cc55b4f5ced8778196bee000000001600141cea77990de27873a66622222cf065ea13755b6967000000"
}

```
```
bitcoin-cli -regtest scantxoutset start '["combo(tpubDCD9NnZCwAYgQoYB6VqZ8huAaGNng8sZMRG7RyscnLeRW41QdyxmmMYjBhB9imfjZ72u67yxgNJLqJvZzpRDbaXRnvmokz2aSVcvY6MMaM3/0/*)"]'
{
  "success": true,
  "searched_items": 106,
  "unspents": [
    {
      "txid": "7b06b13740d8b4caa8eaaf0e50cac59948f27af2c55e597238d0bc430e83df6e",
      "vout": 0,
      "scriptPubKey": "a914f29b76104abfd586947a4a5379626cc55b4f5ced87",
      "amount": 10.00000000,
      "height": 104
    }
  ],
  "total_amount": 10.00000000
```

createwallet "wallet_name" ( disable_private_keys )
getreceivedbyaddress "address" ( minconf )
listwallets
loadwallet "filename"
sendtoaddress "address" amount ( "comment" "comment_to" subtractfeefromamount replaceable conf_target "estimate_mode")
signrawtransactionwithwallet "hexstring" ( [{"txid":"id","vout":n,"scriptPubKey":"hex","redeemScript":"hex"},...] sighashtype )
unloadwallet ( "wallet_name" )
walletcreatefundedpsbt [{"txid":"id","vout":n},...] [{"address":amount},{"data":"hex"},...] ( locktime ) ( replaceable ) ( options bip32derivs )
walletprocesspsbt "psbt" ( sign "sighashtype" bip32derivs )



$ bitcoin-cli help walletcreatefundedpsbt
walletcreatefundedpsbt [{"txid":"id","vout":n},...] [{"address":amount},{"data":"hex"},...] ( locktime ) ( replaceable ) ( options bip32derivs )

Creates and funds a transaction in the Partially Signed Transaction format. Inputs will be added if supplied inputs are not enough
Implements the Creator and Updater roles.

Arguments:
1. "inputs"                (array, required) A json array of json objects
     [
       {
         "txid":"id",      (string, required) The transaction id
         "vout":n,         (numeric, required) The output number
         "sequence":n      (numeric, optional) The sequence number
       } 
       ,...
     ]
2. "outputs"               (array, required) a json array with outputs (key-value pairs)
   [
    {
      "address": x.xxx,    (obj, optional) A key-value pair. The key (string) is the bitcoin address, the value (float or string) is the amount in BTC
    },
    {
      "data": "hex"        (obj, optional) A key-value pair. The key must be "data", the value is hex encoded data
    }
    ,...                     More key-value pairs of the above form. For compatibility reasons, a dictionary, which holds the key-value pairs directly, is also
                             accepted as second parameter.
   ]
3. locktime                  (numeric, optional, default=0) Raw locktime. Non-0 value also locktime-activates inputs
                             Allows this transaction to be replaced by a transaction with higher fees. If provided, it is an error if explicit sequence numbers are incompatible.
4. options                 (object, optional)
   {
     "changeAddress"          (string, optional, default pool address) The bitcoin address to receive the change
     "changePosition"         (numeric, optional, default random) The index of the change output
     "change_type"            (string, optional) The output type to use. Only valid if changeAddress is not specified. Options are "legacy", "p2sh-segwit", and "bech32". Default is set by -changetype.
     "includeWatching"        (boolean, optional, default false) Also select inputs which are watch only
     "lockUnspents"           (boolean, optional, default false) Lock selected unspent outputs
     "feeRate"                (numeric, optional, default not set: makes wallet determine the fee) Set a specific fee rate in BTC/kB
     "subtractFeeFromOutputs" (array, optional) A json array of integers.
                              The fee will be equally deducted from the amount of each specified output.
                              The outputs are specified by their zero-based index, before any change output is added.
                              Those recipients will receive less bitcoins than you enter in their corresponding amount field.
                              If no outputs are specified here, the sender pays the fee.
                                  [vout_index,...]
     "replaceable"            (boolean, optional) Marks this transaction as BIP125 replaceable.
                              Allows this transaction to be replaced by a transaction with higher fees
     "conf_target"            (numeric, optional) Confirmation target (in blocks)
     "estimate_mode"          (string, optional, default=UNSET) The fee estimate mode, must be one of:
         "UNSET"
         "ECONOMICAL"
         "CONSERVATIVE"
   }
5. bip32derivs                    (boolean, optional, default=false) If true, includes the BIP 32 derivation paths for public keys if we know them

Result:
{
  "psbt": "value",        (string)  The resulting raw transaction (base64-encoded string)
  "fee":       n,         (numeric) Fee in BTC the resulting transaction pays
  "changepos": n          (numeric) The position of the added change output, or -1
}

Examples:

Create a transaction with no inputs
> bitcoin-cli walletcreatefundedpsbt "[{\"txid\":\"myid\",\"vout\":0}]" "[{\"data\":\"00010203\"}]"

bitcoin-cli help walletprocesspsbt
walletprocesspsbt "psbt" ( sign "sighashtype" bip32derivs )

Update a PSBT with input information from our wallet and then sign inputs
that we can sign for.

Requires wallet passphrase to be set with walletpassphrase call.

Arguments:
1. "psbt"                      (string, required) The transaction base64 string
2. sign                          (boolean, optional, default=true) Also sign the transaction when updating
3. "sighashtype"            (string, optional, default=ALL) The signature hash type to sign with if not specified by the PSBT. Must be one of
       "ALL"
       "NONE"
       "SINGLE"
       "ALL|ANYONECANPAY"
       "NONE|ANYONECANPAY"
       "SINGLE|ANYONECANPAY"
4. bip32derivs                    (boolean, optional, default=false) If true, includes the BIP 32 derivation paths for public keys if we know them

Result:
{
  "psbt" : "value",          (string) The base64-encoded partially signed transaction
  "complete" : true|false,   (boolean) If the transaction has a complete set of signatures
  ]
}

Examples:
> bitcoin-cli walletprocesspsbt "psbt"

