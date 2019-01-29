# glacier-psbt

## What is this?

This is an investigation into using Partially Signed Bitcoin
Transactions
[(PSBT)](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)
from a [Bitcoin Core](https://bitcoincore.org/) [full
node](https://bitcoin.org/en/full-node) for signing Bitcoin
transactions for [Glacier](https://glacierprotocol.org) cold-storage
addresses.

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

## Discussion

# Process 2: Constructing an unsigned PSBT using online node

## Discussion

# Process 3: Signing the PSBT using a quarantined laptop

## Discussion

# Process 4: Finalizing and transmitting the signed transaction using online node

## Discussion

# Outstanding Issues and Questions
