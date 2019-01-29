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

## Discussion

Since each Glacier address is in its own wallet, it is not simple or
obvious, via either GUI or CLI, how to view the balances.

Since the addresses are watch-only, `getwalletinfo` shows 0.

# Process 3: Constructing an unsigned PSBT using online node

*Use case: user wishes to withdraw bitcoins from cold storage.*

## Discussion

In order to simplify both the scripting and the explanation of the
process, we restrict the withdrawal UTXOs to a single Glacier address
at a time. (There might still be multiple inputs, since Glacier reuses
addresses.)

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

# References

* Official [PSBT docs](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)

* My questions on Stack Exchange:
  [PSBT](https://bitcoin.stackexchange.com/questions/83070/expected-use-model-for-psbt),
  [importmulti](https://bitcoin.stackexchange.com/questions/83102/how-to-import-p2wsh-in-p2sh-multisig-as-watch-only),
  [unloadwallet](https://bitcoin.stackexchange.com/questions/83121/confusion-about-unloadwallet),
  and [loadwallet
  rescans](https://bitcoin.stackexchange.com/questions/83236/how-can-i-tell-when-rescanblockchain-is-finished).
