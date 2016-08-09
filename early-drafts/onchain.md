# Lightning On-chain Transaction Handling #

# Status #

Initial draft

# Author #

Pierre-Marie Padiou, ???

Rusty Russell, Blockstream <mailto:rusty@rustcorp.com.au>

# Abstract #

Lightning allows for two parties (A and B) to make transactions off-chain, by both holding a cross-signed *commitment tx*, which describes the current state of the channel (basically the current balance). This *commitment tx* is updated everytime a new payment is made, and is spendable at all times.

There are three ways a channel can end:

1. The good way (*mutual close*): at some point A and B agree on closing the channel, they generate a *final tx* (which is similar to a *commitment tx* without any pending payments), and publish it on the blockchain (see BOLT #2 "4.2: Closing negotiation: close_signature").
2. The bad way (*unilateral close*): something goes wrong, without necessarily any evil intent on either side (maybe one of the party crashed, for instance). Anyway, one side publishes its latest *commitment tx*.
3. The ugly way (*cheating attempt*): one of the parties deliberately tries to cheat by publishing an outdated version of its *commitment tx* (presumably one that was more in her favor).

Because Lightning is designed to be trustless, there is no risk of loss of funds in any of these 3 cases, provided that the situation is properly handled. The goal of this document is to explain exactly how node A should react to seeing any of these on-chain.

# Table of Contents

TODO

# Commitment Transaction

A and B each hold a *commitment tx*, which has 4 types of outputs:

1. _A's main output_: Zero or one outputs which pay to A's `final_key`.
2. _B's main output_: Zero or one outputs which pay to B's `final_key`.
3. _A's offered HTLCs_: Zero or more pending payments (*HTLCs*) to pay B in return for a redemption preimage.
4. _B's offered HTLCs_: Zero or more pending payments (*HTLCs*) to pay A in return for a redemption preimage.

As an incentive for A and B to cooperate, an `OP_CHECKSEQUENCEVERIFY` relative timeout encumbers A's outputs in A's *commitment tx*, and B's outputs in B's *commitment tx*. If A publishes its commitment tx, she won't be able to get her funds immediately but B will. As a consequence, A and B's *commitment txes* are not identical, they are (usually) symmetrical.

See "BOLT #3: Transaction formats" for more details.

## Commitment Transaction: Requirements

All transactions referred to are assumed to be valid, signed transactions:
invalid transactions SHOULD be ignored.  For example, the next paragraph assumes
that the spends are valid, correctly-signed transactions which can actually
spend the output they claim.

After a channel is opened, the funding transaction output is
considered *unresolved*; it is *resolved* by any transaction which
spends that output.  A node MUST monitor the blockchain for
transactions which spend any output which is not *irrevocably
resolved* until all outputs are *irrevocably resolved*.

A node MUST *resolve* all outputs as specified below, and MUST be
prepared to resolve them multiple times in case of blockchain
reorganizations.

Outputs which are *resolved* by a transaction are considered *irrevocably
resolved* once they are included in a block at least 100 deep on the most-work
blockchain.  100 blocks is far greater than the longest known bitcoin fork,
and the same value used to wait for confirmations of miner's rewards[1].

A node SHOULD fail the connection if it is not already closed when it
sees the funding transaction spent.  A node MAY send a descriptive
error packet in this case.

## Mutual Close Handling

A mutual close transaction *resolves* the funding transaction output.

A node doesn't need to do anything else as it has already agreed to the
output, which is sent to its specified scriptpubkey (see BOLT #2 "4.1:
Closing initiation: close_shutdown").

## Unilateral Close Handling

There are two cases to consider here: in the first case, node A sees
its own *commitment tx*, in the second, it sees the node B's unrevoked
*commitment tx*.

Either transaction *resolves* the funding transaction output.

When node A sees its own *commitment tx*:

1. _A's main output_: A node SHOULD spend this output to a convenient address.
   This avoids having to remember the complicated witness script associated
   with that particular channel for later spending.  A node MUST wait until
   the `OP_CHECKSEQUENCEVERIFY` delay has passed (as specified by the other
   node's `open_channel` `delay` field) before spending the output.  If the
   output is spent (as recommended), the output is *resolved* by the spending
   transaction, otherwise it is considered *resolved* by the *commitment tx*.
2. _B's main output_: No action required, this output is considered *resolved*
   by the *commitment tx*.
3. _A's offered HTLCs_: See On-chain HTLC Handling: Our Offers below.
4. _B's offered HTLCs_: See On-chain HTLC Handling: Their Offers below.

Similarly, when node A sees a *commitment tx* from B:

1. _A's main output_: No action is required; this is a simple P2WPKH output.
   This output is considered *resolved* by the *commitment tx*.
2. _B's main output_: No action required, this output is considered *resolved*
   by the *commitment tx*.
3. _A's offered HTLCs_: See On-chain HTLC Handling: Our Offers below.
4. _B's offered HTLCs_: See On-chain HTLC Handling: Their Offers below.

Note that there can be more than one valid, unrevoked *commitment tx*
after a signature has been received via `update_commit` and before the
corresponding `update_revocation`.  Either commitment can serve as B's
*commitment tx* and a node MUST handle both as specified here.

### On-chain HTLC Handling: Our Offers

A node MUST watch for spends of *commitment tx* outputs for HTLCs it offered;
each one must be *resolved* by a timeout transaction (the node pays back to
itself) or redemption transaction (the other node provides the redemption
preimage).

The HTLC is *expired* once the depth of the latest block is equal or greater
than the HTLC `expiry` (if in blocks), or once the median time of the latest
block is greater than HTLC `expiry` (if in seconds).

If the *commitment tx* is the other node's, the output is considered *timed
out* once the HTLC is expired.  If the *commitment tx* is this node's, the
output is considered *timed out* once the HTLC is expired, AND the output's
`OP_CHECKSEQUENCEVERIFY` delay has passed.

If a node sees a redemption transaction, the output is considered *irrevocably
resolved*, and the node MUST extract the preimage from the transaction input
witness.  This is either to prove payment (if this node originated the
payment), or to redeem the corresponding incoming HTLC from another peer.
Note that we don't care about the fate of the redemption transaction itself
once we've extracted the preimage; the knowledge is not revocable.

If the output has *timed out* and not been *resolved*, the node MUST *resolve*
the output by spending it.  The node SHOULD provide sufficient fee that this
transaction spending the output is mined before the incoming HTLC times out,
to avoid the risk of one-sided redemption.

Note that in cases where both resolutions are possible (redemption seen after
timeout, for example), either intepretation is acceptable; it is the
responsibility of the other node to ensure that doesn't occur.

### On-chain HTLC Handling: Their Offers

If the node receives (or already knows) a redemption preimage for an unresolved *commitment tx* output it was
offered, it MUST *resolve* the output by spending it using the preimage.
Without this, the other node could spend it once it as *timed out* as above.

Otherwise, if the output HTLC has expired, it is considered *irrevocably
resolved*.  In theory, time could go backwards due to a blockchain
reorganization, but an implementer SHOULD expend their mental effort dealing
with real problems!

## Cheating Attempt Handling

A node MUST NOT broadcast a *commitment tx* for which it has exposed
the revocation preimage.

If a node sees a *commitment tx* for which it has a revocation
preimage, it *resolves* the funding transaction output:

1. _A's main output_: No action is required; this is a simple P2WPKH output.
   This output is considered *resolved* by the *commitment tx*.
2. _B's main output_: The node MUST *resolve* this by spending using the
   revocation preimage.
3. _A's offered HTLCs_: The node MUST *resolve* this by spending using the
   revocation preimage.
4. _B's offered HTLCs_: The node MUST *resolve* this by spending using the
   revocation preimage.

The node MAY use a single transaction to *resolve* all the outputs;
due to the 300 HTLC-per-party limit (See BOLT #2: 3.2. Adding an HTLC)
this can be done within a standard transaction.

# General Requirements

A node SHOULD report an error to the operator if it sees a transaction 
spend the funding transaction output which does not fall
into one of these categories (mutual close, unilateral close, or
cheating attempt).  Such a transaction implies its private key has
leaked, and funds may be lost.

A node MAY simply watch the contents of the most-work chain for
transactions, or MAY watch for (valid) broadcast transactions a.k.a
mempool.  Considering mempool transactions should cause lower latency
for HTLC redemption, but on-chain HTLCs should be such an unusual case
that speed cannot be considered critical.
