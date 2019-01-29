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

* The user has a Bitcoin Core full node with a complete blockchain.

* The user is familiar with using
  [`bitcoin-cli`](https://bitcoin.org/en/developer-examples).

# Process 1: Importing Glacier address to online node

*Use case: the user has constructed a new Glacier address using the
quarantined laptops.*

*Use case: the user has a new PC and new Bitcoin Core installation.*

*Use case: the user has inherited Bitcoins stored using Glacier.*

## Discussion

Newly-created Glacier addresses must be imported into the full node as
a watch-only address so that (1) the balance can be displayed; and (2)
withdrawal transactions can be constructed in the future.

If the address is new, we can assume no deposits have been made to
this address, and avoid a blockchain rescan. If the address is old
(user has a new PC with a new Bitcoin Core installation for a
years-old address; an heir is attempting to claim their deceased
relative's Glacier coins) then a rescan is required, at least back to
the date the address was created.

Fortunately the Glacier instructions have always required the user to
print the date on the cold storage information page.


# Process 2: Constructing an unsigned PSBT using online node

*Use case: user wishes to withdraw bitcoins from cold storage.*

## Discussion



# Process 3: Signing the PSBT using a quarantined laptop

*Use case: user has completed Process 2 and needs to sign the PSBT
 using their paper keys.*

## Discussion



# Process 4: Finalizing and transmitting the signed transaction using online node

*Use case: user has completed Process 3 and needs to verify and
 transmit the completely signed transaction.*

## Discussion



# Outstanding Issues and Questions

# References
