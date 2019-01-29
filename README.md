# glacier-psbt

## What is this?

This is an investigation and proposed enhancement to the [Glacier
Protocol](https://glacierprotocol.org) (Bitcoin cold storage), to use
Partially Signed Bitcoin Transactions
[(PSBT)](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)
from a [Bitcoin Core](https://bitcoincore.org/) [full
node](https://bitcoin.org/en/full-node) instead of the current manual
UTXO selection and fee calculations that rely on third-party online
services.

For more discussion, see [GlacierProtocol
#54](https://github.com/GlacierProtocol/GlacierProtocol/issues/54).

## Should I use it?

Not yet. It is here for development, discussion, and testing.

## What's going to happen with this?

At first, it will document how to use `bitcoin-cli` by hand to create
and sign PSBTs for Glacier. Later I intend to add scripting to this,
and instructions to be placed in an appendix in the Glacier document.

With enough testing and acceptance, we could replace the primary
Glacier system with this one.

# Goals and Principles

* **Simple**: the non-technical target user of Glacier should be able
    to follow this process when it is completed. Until then, it might
    be highly technical.

* **Robust**: users should have few opportunities to screw up, and
    practically zero chance of losing funds, when following this
    process.

* **Auditable**: use standard mechanisms and interfaces so that
    advanced users can see that nothing fishy is going on.

# Assumptions

*These should be relaxed over time as this process is further developed.*

* The user has a Bitcoin Core v0.17.1 full node with a complete
  blockchain.

* The user is familiar with using
  [`bitcoin-cli`](https://bitcoin.org/en/developer-examples).

# Process 1: Importing Glacier address to online node

*Use case: the user has constructed a new Glacier address using the
quarantined laptops.*

*Use case: the user has a new PC and new Bitcoin Core installation.*

*Use case: the user has inherited Bitcoins stored using Glacier.*

## Command-line

```
### Create new wallet for cold storage address 2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC

### Wallet creation
bitcoin-cli -testnet createwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC" true

### Ugly: need to decodescript to get witness script, in case this is p2sh-segwit. Won't hurt in case it's legacy.
### See https://bitcoin.stackexchange.com/questions/83102/how-to-import-p2wsh-in-p2sh-multisig-as-watch-only
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC decodescript "5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae"

## Then take results["segwit"]["hex"] as a second redeemscript.

## Address import: once per address. Script will need to calculate timestamp based on user-entered creation date.
## I calculated this timestamp as 24 hours ago, since that's before I sent some coins to it
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC importmulti '[{ "scriptPubKey": { "address": "2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC" }, "timestamp":1545413762, "redeemscript":"5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae", "watchonly":true }, { "scriptPubKey": { "address": "2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC" }, "timestamp":1545413762, "redeemscript":"0020dabea3445c14e4a08d6705db4373bef467d4c64e7c8ddf149be50670de6878ae", "watchonly":true }]'

bitcoin-cli -testnet unloadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"


```

## Discussion

Every Glacier address must be imported into the full node as a
watch-only address so that (1) the balance can be displayed; and (2)
withdrawal transactions can be constructed in the future.

If the address is new, we can assume no deposits have been made to
this address, and avoid a blockchain rescan. If the address is old
(user has a new PC with a new Bitcoin Core installation for a
years-old address; an heir is attempting to claim their deceased
relative's Glacier coins) then a rescan is required, at least back to
the date the address was created.

Fortunately the Glacier instructions have always required the user to
print the date on the cold storage information page.

Each Glacier address should be put into its own wallet so that:

* Withdrawal transactions can be restricted to using only those UTXOs,
  simplifying the signing process.

* Withdrawal transactions can be built from a wallet containing no
  private keys, ensuring change addresses are entirely under our
  control.

# Process 2: Display balances of all Glacier addresses

*Use case: obvious*

## Command-line

```
### Caution: if already loaded, this bombs out
bitcoin-cli -testnet loadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"

## Check balance
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC getbalance "*" 0 true
bitcoin-cli -testnet unloadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
```

## Discussion

Since each Glacier address is in its own wallet, it is not simple or
obvious, via either GUI or CLI, how to view the balances.

Since the addresses are watch-only, `getwalletinfo` shows 0.

The script should ask the node for a list of wallets (using
`listwallets`), filter Glacier-\*, and show balances for each of them,
and a total balance.

# Process 3: Constructing an unsigned PSBT using online node

*Use case: user wishes to withdraw bitcoins from cold storage.*

## Command-line

Spending the entire balance:
```
bitcoin-cli -testnet loadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC walletcreatefundedpsbt \
            '[]' \
            '[{"2NCthaDAsJUJ2q5s1L4HhexUGMJ5t16vqer":0.17400000}]' \
            0 \
            '{"includeWatching":true, "subtractFeeFromOutputs":[0], "replaceable":true, "changeAddress":"2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"}' \
            false
bitcoin-cli -testnet unloadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"

```

Spending less than the entire balance:
```
bitcoin-cli -testnet loadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC walletcreatefundedpsbt \
            '[]' \
            '[{"2NCthaDAsJUJ2q5s1L4HhexUGMJ5t16vqer":0.00400000}]' \
            0 \
            '{"includeWatching":true, "replaceable":true, "changeAddress":"2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"}' \
            false
bitcoin-cli -testnet unloadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"

```


## Discussion

In order to simplify both the scripting and the explanation of the
process, we restrict the withdrawal UTXOs to a single Glacier address
at a time. (There might still be multiple inputs, since Glacier reuses
addresses.)

To spend the entire balance, we include the balance in the output and
use `subtractFeeFromOutputs` to allow Bitcoin Core to calculate the
maximum amount after fees.

Unfortunately `walletcreatefundedpsbt` does not seem to do dust
detection; if fees are high and inputs are small, it could waste money
by including uneconomical inputs. When attempting to spend the entire
balance, the scripting should detect and avoid this. (Remove smallest
input, see if output amount grows. Repeat until it doesn't, then back
up one. Keep in mind, this might result in no inputs, if all are
tiny.)

# Process 4: Signing the PSBT using a quarantined laptop

*Use case: user has completed Process 3 and needs to sign the PSBT
 using their paper keys.*

## Discussion

There are several safety and sanity checks that the offline scripting
should perform before signing any withdrawal.

1. Must have exactly one output whose value equals inputs plus fee; or
must have exactly two outputs, one of which is the change back to
same origin.

2. All inputs must be from same address (so we can assume it's ours
without having to ask user to type in cold storage address)

3. Confirm cold storage address, destination, amount, and fee with
user, before they type in privkeys.

4. After walletprocesspsbt, ensure "complete":true (unless doing
sequential signing)

5. After getting the raw transaction, calculate the fee in sat/vbyte
and display for user.


# Process 5: Finalizing and transmitting the signed transaction using online node

*Use case: user has completed Process 4 and needs to verify and
 transmit the completely signed transaction.*

## Discussion

One final sanity and safety check should be performed on the online
node before broadcasting the transaction to the network.


# Outstanding Issues and Questions

* PSBTs can get large. It might require multiple QR codes to transmit
  back and forth. The QR standard has a mechanism for this
  ("structured QR codes") but it is not well-supported by the popular
  QR apps, especially phone apps.

# References

* Official [PSBT docs](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)

* My questions on Stack Exchange:
  [PSBT](https://bitcoin.stackexchange.com/questions/83070/expected-use-model-for-psbt),
  [importmulti](https://bitcoin.stackexchange.com/questions/83102/how-to-import-p2wsh-in-p2sh-multisig-as-watch-only),
  [unloadwallet](https://bitcoin.stackexchange.com/questions/83121/confusion-about-unloadwallet),
  and [loadwallet
  rescans](https://bitcoin.stackexchange.com/questions/83236/how-can-i-tell-when-rescanblockchain-is-finished).
