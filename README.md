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
  (testnet) blockchain.

* The user is familiar with using
  [`bitcoin-cli`](https://bitcoin.org/en/developer-examples).

# Process 1: Importing Glacier address to online node

*Use case: the user has constructed a new Glacier address using the
quarantined laptops.*

*Use case: the user has a new PC and new Bitcoin Core installation.*

*Use case: the user has inherited Bitcoins stored using Glacier.*

## Command line

```
### Create new wallet for cold storage address 2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC

### Wallet creation
bitcoin-cli -testnet createwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC" true

### Ugly: need to decodescript to get witness script, in case this is p2sh-segwit. Won't hurt in case it's legacy.
### See https://bitcoin.stackexchange.com/questions/83102/how-to-import-p2wsh-in-p2sh-multisig-as-watch-only
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC decodescript \
    "5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae"

## Then take results["segwit"]["hex"] as a second redeemscript.

## Address import: once per address. Script will need to calculate timestamp based on user-entered creation date.
## I calculated this timestamp as 24 hours ago, since that's before I sent some coins to it
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC importmulti \
    '[
      {
        "scriptPubKey": { "address": "2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC" },
        "timestamp":1545413762,
        "redeemscript":"5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae",
        "watchonly":true
      },
      {
        "scriptPubKey": { "address": "2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC" },
        "timestamp":1545413762,
        "redeemscript":"0020dabea3445c14e4a08d6705db4373bef467d4c64e7c8ddf149be50670de6878ae",
        "watchonly":true
      }
    ]'

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

## Command line

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

## Command line

Spending the entire balance:
```
bitcoin-cli -testnet loadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC walletcreatefundedpsbt \
    '[]' \
    '[{"2MtPcPXMrGxhprqSyLU8zDsbuMyESxdPpb2":0.17400000}]' \
    0 \
    '{
        "includeWatching":true,
        "subtractFeeFromOutputs":[0],
        "replaceable":true,
        "changeAddress":"2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
     }' \
     false
bitcoin-cli -testnet unloadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
```

Spending less than the entire balance:
```
bitcoin-cli -testnet loadwallet "Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
bitcoin-cli -testnet -rpcwallet=Glacier-2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC walletcreatefundedpsbt \
    '[]' \
    '[{"2MtPcPXMrGxhprqSyLU8zDsbuMyESxdPpb2":0.00400000}]' \
    0 \
    '{
        "includeWatching":true,
        "replaceable":true,
        "changeAddress":"2MzqiaZzpLT2SSBfsFqqo3FpZsP8g6WTvyC"
    }' \
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

## Command line

```
### Remember, this is on quarantined laptop, running bitcoind like so:
bitcoind -testnet -daemon -connect=0.0.0.0

### First show us what we have here:
bitcoin-cli -testnet decodepsbt "cHNidP8BAFMCAAAAASX9+/r5FLgd8fsd2thLtfeNA0Ou4+bIcigceMcydMqXAAAAAAD9////AQGACQEAAAAAF6kUDI4nEy+SWSrEWcce4KKL6im4O9WHAAAAAAABASDAgAkBAAAAABepFFNO484huPgFEo92udl341G7vqINhwEEIgAg2r6jRFwU5KCNZwXbQ3O+9GfUxk58jd8Um+UGcN5oeK4BBYtSIQKfUxUD+s2sJJb1CkRtm9KYRqBqBKReOEW2VrtHHfQi/CEC4weHcDqZDkAVosuQcfz9HH1GQfspTktMP19rRQoZJRMhAtooCIqAImURccTxNCm5hwnavhO8baUmU3/dLQcw3S27IQMobJbsqoUKa6Q8xF+7U5wfsdZcI8wPPNCfz5dlgm/53lSuAAA="

### User has confirmed details and entered privkeys
bitcoin-cli -testnet importprivkey cVb7UNXC7nXzBLEtQZMzJcoRYNsiGCTjgAiibJqtEpLM1gimrhW2
bitcoin-cli -testnet importprivkey cUxC3su6U61NsGspvcsD7JxSDUiRY7YaybTHGDgF5e8T5tpk7UD1

### Sign:
bitcoin-cli -testnet walletprocesspsbt "cHNidP8BAFMCAAAAASX9+/r5FLgd8fsd2thLtfeNA0Ou4+bIcigceMcydMqXAAAAAAD9////AQGACQEAAAAAF6kUDI4nEy+SWSrEWcce4KKL6im4O9WHAAAAAAABASDAgAkBAAAAABepFFNO484huPgFEo92udl341G7vqINhwEEIgAg2r6jRFwU5KCNZwXbQ3O+9GfUxk58jd8Um+UGcN5oeK4BBYtSIQKfUxUD+s2sJJb1CkRtm9KYRqBqBKReOEW2VrtHHfQi/CEC4weHcDqZDkAVosuQcfz9HH1GQfspTktMP19rRQoZJRMhAtooCIqAImURccTxNCm5hwnavhO8baUmU3/dLQcw3S27IQMobJbsqoUKa6Q8xF+7U5wfsdZcI8wPPNCfz5dlgm/53lSuAAA="

bitcoin-cli -testnet stop
```

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

## Command line

```
## Convert to raw transaction format
bitcoin-cli -testnet finalizepsbt \
    "cHNidP8BAFMCAAAAASX9+/r5FLgd8fsd2thLtfeNA0Ou4+bIcigceMcydMqXAAAAAAD9////AQGACQEAAAAAF6kUDI4nEy+SWSrEWcce4KKL6im4O9WHAAAAAAABASDAgAkBAAAAABepFFNO484huPgFEo92udl341G7vqINhwEHIyIAINq+o0RcFOSgjWcF20NzvvRn1MZOfI3fFJvlBnDeaHiuAQj9HgEEAEcwRAIgVhc8bBMY3Rfn6AKiW+srBcBNKSjhfTbNyZbi+WJH+44CIAM44QgBeZPutarNopHVa8t72NkQomzuN74Q3w5O314iAUcwRAIgO2ydHblxgMA0Pvc14M00qsYXM05X4Ra8dNDXwZiNLmICIDv0HSikT95699H7ftkJP5n3VmnlNMatBPqruGmKh+HrAYtSIQKfUxUD+s2sJJb1CkRtm9KYRqBqBKReOEW2VrtHHfQi/CEC4weHcDqZDkAVosuQcfz9HH1GQfspTktMP19rRQoZJRMhAtooCIqAImURccTxNCm5hwnavhO8baUmU3/dLQcw3S27IQMobJbsqoUKa6Q8xF+7U5wfsdZcI8wPPNCfz5dlgm/53lSuAAA=" \
    true

## Make sure transaction will be accepted, explain to user if not
bitcoin-cli -testnet testmempoolaccept \
    '["0200000000010125fdfbfaf914b81df1fb1ddad84bb5f78d0343aee3e6c872281c78c73274ca970000000023220020dabea3445c14e4a08d6705db4373bef467d4c64e7c8ddf149be50670de6878aefdffffff01018009010000000017a9140c8e27132f92592ac459c71ee0a28bea29b83bd5870400473044022056173c6c1318dd17e7e802a25beb2b05c04d2928e17d36cdc996e2f96247fb8e02200338e108017993eeb5aacda291d56bcb7bd8d910a26cee37be10df0e4edf5e220147304402203b6c9d1db97180c0343ef735e0cd34aac617334e57e116bc74d0d7c1988d2e6202203bf41d28a44fde7af7d1fb7ed9093f99f75669e534c6ad04faabb8698a87e1eb018b5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae00000000"]'

## Decode raw transaction so we can confirm details with user before transmitting
bitcoin-cli -testnet decoderawtransaction \
    "0200000000010125fdfbfaf914b81df1fb1ddad84bb5f78d0343aee3e6c872281c78c73274ca970000000023220020dabea3445c14e4a08d6705db4373bef467d4c64e7c8ddf149be50670de6878aefdffffff01018009010000000017a9140c8e27132f92592ac459c71ee0a28bea29b83bd5870400473044022056173c6c1318dd17e7e802a25beb2b05c04d2928e17d36cdc996e2f96247fb8e02200338e108017993eeb5aacda291d56bcb7bd8d910a26cee37be10df0e4edf5e220147304402203b6c9d1db97180c0343ef735e0cd34aac617334e57e116bc74d0d7c1988d2e6202203bf41d28a44fde7af7d1fb7ed9093f99f75669e534c6ad04faabb8698a87e1eb018b5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae00000000"

## After confirmation, send:
bitcoin-cli -testnet sendrawtransaction \
    "0200000000010125fdfbfaf914b81df1fb1ddad84bb5f78d0343aee3e6c872281c78c73274ca970000000023220020dabea3445c14e4a08d6705db4373bef467d4c64e7c8ddf149be50670de6878aefdffffff01018009010000000017a9140c8e27132f92592ac459c71ee0a28bea29b83bd5870400473044022056173c6c1318dd17e7e802a25beb2b05c04d2928e17d36cdc996e2f96247fb8e02200338e108017993eeb5aacda291d56bcb7bd8d910a26cee37be10df0e4edf5e220147304402203b6c9d1db97180c0343ef735e0cd34aac617334e57e116bc74d0d7c1988d2e6202203bf41d28a44fde7af7d1fb7ed9093f99f75669e534c6ad04faabb8698a87e1eb018b5221029f531503facdac2496f50a446d9bd29846a06a04a45e3845b656bb471df422fc2102e30787703a990e4015a2cb9071fcfd1c7d4641fb294e4b4c3f5f6b450a1925132102da28088a8022651171c4f13429b98709dabe13bc6da526537fdd2d0730dd2dbb2103286c96ecaa850a6ba43cc45fbb539c1fb1d65c23cc0f3cd09fcf9765826ff9de54ae00000000"
```

## Discussion

One final sanity and safety check should be performed on the online
node before broadcasting the transaction to the network.

Scripting this portion has serious security issues; see discussion
below. Perhaps this portion should be done on an entirely different
computer that has never run any Glacier code. (Still using coinb.in?)

# Security Analysis

The PSBT (BIP174) flow was designed to be secure provided the offline
node is secure. This includes any scripting provided by Glacier.

Unfortunately this flow, as envisioned, is less secure than today's
Glacier, due to the requirement to run GlacierScript code on both the
online and the offline computers. If Glacier itself were compromised,
the two scripts could coordinate to exfiltrate keys and steal
funds. Today's Glacier uses only public web services on the online
node, requiring an attacker to compromise both GlacierScript and those
online services at the same time.

For example, if process 5 were scripted and GlacierScript itself was
compromised, an attacker could use such scripting to exfiltrate
private keys (by hiding them inside the base64-encoded PSBT data). By
making process 5 steps manual, we prevent that attack vector.

But GlacierScript code has already run previously on the online node
-- if GlacierScript itself was compromised, anything is possible. It
might have installed malware that intercepts the copy/pasted PSBT,
extracting the privkeys before placing the legitimate PSBT in the
clipboard.

One workaround is to use an entirely different online computer for
decoding and transmitting the raw transaction, which probably means
using coinb.in (instead of having the user install a second copy of
Bitcoin Core with a second copy of the blockchain). This would also
complicate sequential signing.

Another option would be to accept and understand that a compromised
Glacier would lead to loss of funds -- but this is a serious step down
from today's Glacier security.

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
