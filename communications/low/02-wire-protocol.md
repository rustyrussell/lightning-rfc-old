# Basis of Lightning Technology RFC 2 #

# Status #

Initial draft

# Author #

Rusty Russell, Blockstream <mailto:rusty@rustcorp.com.au>

# Abstract #

Pairs of lightning nodes cooperate to establish a bitcoin transaction
(called a "channel") which requires the two of them to spend, and
manage updates which spend that transaction.  This occurs in three
phases:

1.  Channel Establishment
2.  Normal Operation
3.  Channel Close

This document describes the protocol used between nodes for each of
these phases.

# General Protocol #

The wire protocol used is protobufs[1], passed over an encrypted
channel set up and used as specified in BOLT#1[2].

In general, if a node received a field it does not understand, if the
field number is odd it MUST ignore it, if it is even it MUST treat it
as an error and fail the connection (also known as "it's OK to be
odd").  A node MUST NOT send out an even-numbered field not listed
here without prior negotiation for its acceptance.

## pkt message format ##

    // This is the union which defines all of them
    message pkt {
      oneof pkt {
	  FIXME
      }
    }

## Persistence and Retransmission ##

Because communication transports are unreliable and may need to be
re-established from time to time, the design of the transport has been
explicitly separated from the protocol.

A node MUST handle continuing a previous channel on a new encrypted
transport.  A node MUST set the `acknowledge` field in the header of
the `authenticate` message to the the number of previously-processed
non-authenticate messages.  A node MUST NOT send out a message with an
`acknowledge` field smaller than a previous `acknowledge` field.

A node MUST retransmit messages which may not have been included in the
`authenticate` header's `acknowledge` field.

A node retransmitting previous messages MUST ensure they are bitwise
identical after decryption.

# Channel Establishment #

Channel establishment begins immediately after authentication, and consists of each node sending an `open_channel` message, followed by one node sending `open_anchor`, the other providing its `open_commit_sig` then both sides waiting for the anchor transaction to enter the blockchain and reach their specified depth, at which point they send `open_complete`.  After both sides have sent `open_complete` the channel is established and can begin normal operation.

If this fails at any stage, or a node decides that the channel terms offered by the other node are not suitable, see "Failing The Connection".

## The Initial open_channel message ##

The first message for a new connection after authentication is the
`open_channel` message.  This contains information about each node, and
its requirements to set up a channel.

Informational fields are:

* `revocation_hash`, `next_revocation_hash`: the revocation hashes for the first two commitment transactions.  See "Revocation Hashes".
* `commit_key`: the bitcoin pubkey which this node will use to sign commitment transactions and redeem the anchor transaction output.
* `final_key`: the bitcoin pubkey to use for a mutual close transaction.
* `anch`: whether the node will create the anchor transaction to fund the channel or not.  Currently this MUST be set to `WILL_CREATE_ANCHOR` by the node which created the connection, and `WONT_CREATE_ANCHOR` by the node which received the connection request.

The negotiation fields which place requirements on the receiver are:

* `delay`: the `OP_CHECKSEQUENCEVERIFY` value the other node should use to delay payments to itself.  The sender SHOULD set this to a value sufficient to ensure it can irreversibly spend a commitment transaction output in case of misbehavior by the receiver.  This is effectively a demand on how long the receiver could have their funds withheld, thus the receiver MUST reject the delay if it considers it unreasonably large.
* `min_depth`: the minimum block depth before the anchor transaction is considered irreversible and Normal Operation can begin.  The receiver MAY reject the delay if it considers it unreasonably large; the sender which is not creating the anchor SHOULD set this to a value sufficient to ensure the anchor cannot be unspent.
* `initial_fee_rate`: the fee-per-kilobyte to use on commitment transactions, in satoshi.  The receiver MUST fail the connection if considers this unnecessarily large or too small for timely processing.  The sender SHOULD set this to at least the rate it estimates would cause the transaction to be immediately included in a block.

### open_channel message format ###

    // Set channel params.
    message open_channel {
      // Relative locktime for outputs going to us.
      required locktime delay = 1;
      // Hash for revoking first commitment transaction.
      required sha256_hash revocation_hash = 2;
      // Hash for revoking second commitment transaction.
      required sha256_hash next_revocation_hash = 8;
      // Pubkey for anchor to pay into commitment tx.
      required bitcoin_pubkey commit_key = 3;
      // How to pay money to us from commit_tx.
      required bitcoin_pubkey final_key = 4;
    
      enum anchor_offer {
        // I will create the anchor
        WILL_CREATE_ANCHOR = 1;
        // I won't create the anchor
        WONT_CREATE_ANCHOR = 2;
      }
      required anchor_offer anch = 5;
    
      // How far must anchor be buried before we consider channel live?
      optional uint32 min_depth = 6 [ default = 0 ];
    
      // How much fee would I like on commitment tx?
      required uint32 initial_fee_rate = 7;
    }

## Describing the anchor transaction: open_anchor ##

Whichever node offered the anchor (`WILL_CREATE_ANCHOR`) will initially fund the channel.  This node will create a transaction with an output redeemable by the `commit_key`s from both nodes (see [3]), which it MUST NOT broadcast.  It then sends an `open_anchor` message which allows the recipient to calculate the signature for the initial commitment transaction.

The fields of this message are:

* `txid`: the transaction ID of the anchor transaction.
* `output_index`: the index of the output which is to fund the channel.
* `amount`: the amount in satoshis of the `output_index` output of `txid`.
* `commit_sig`: the signature for the receiver's initial commitment transaction.

The receiver MAY fail the connection if `amount` is too low; the sender MUST offer an `amount` sufficient to cover the fees of both initial commitment transactions.  The receiver MUST fail the connection if the `commit_sig` does not sign its initial commit transaction.

### open_anchor message format ###

    // Whoever is supplying anchor sends this.
    message open_anchor {
      // Transaction ID of anchor.
      required sha256_hash txid = 1;
      // Which output is going to the 2 of 2.
      required uint32 output_index = 2;
      // Amount of anchor output.
      required uint64 amount = 3;
    
      // Signature for your initial commitment tx.
      required signature commit_sig = 4;
    }

## Accepting the Anchor Transaction: open_commit_sig ##

Upon accepting the `open_anchor` message, the node creates a signature for the anchor creator's initial commitment transaction, and sends it in an `open_commit_sig` message.

The receiver (ie. anchor creator) MUST fail the connection if the `commit_sig` does not sign its initial commitment transaction.  The receiver SHOULD broadcast the anchor transaction upon receipt of the signature; this ensures that it can use that signature on our initial commitment transaction to redeem the anchor funds in case of failure.

### open_commit_sig message format ###

    // Reply: signature for your initial commitment tx
    message open_commit_sig {
      required signature sig = 1;
    }

## Waiting for the Anchor: open_complete ##

Once the anchor has reached `min_depth` in the blockchain, the node sends `open_complete` to indicate it is ready to transition to normal operating mode.  A node MUST NOT begin normal operation until it has both sent and received `open_complete`.  A node MAY fail the connection if it does not receive `open_complete` in a timely manner after the other's `min_depth` is reached.

### open_complete message format ###

    // Indicates we've seen anchor reach min-depth.
    message open_complete {
      // FIXME: add a merkle proof plus block headers here?
    }

# Normal Operation #

Once both nodes have exchanged `open_complete`, the channel can be
used to make payments via Hash TimeLocked Contracts.  Each node
stores:

1. Other node's previous obsoleted commitment transactions
(or at least enough information to spend them, see [3]),
2. The current HTLCs.
3. The other node's signature on the commitment transaction created from current HTLCs.
4. A list of staged changes proposed by the other node.
5. A list of staged changes proposed by this node.

Note that updates are asynchronous, which means that the two
commitment transactions may be out of sync in intermediate stages.

Each node can send messages offering new HTLCs or closing HTLCs
offered by the other node, and then a signature to commit to those
changes and any it has received: this message indicates the highest
update number being applied, as there is a possible ambiguity for
in-flight updates.  Once a node has received a signature for a new
commitment transaction, it sends the revocation preimage which
invalides the old one.

Here's an example:

    NODE A                NODE B
	Committed: []         Committed: []
	Staged:    []         Staged:    []

A decides to offer a new HTLC X:

    Committed: []
    Staged:    [X]
         ADD HTLC X ----->
	                      Committed: []
	                      Staged:    [X]

B decides to offer a new HTLC Y:

	                      Committed: []
	                      Staged:    [X Y]
            <---------- ADD HTLC Y
    Committed: []
    Staged:    [X Y]

A decides to commit its update, and the 1 update it got from B:

         SIG 1 ----->
	                      Committed: [X Y]
	                      Staged:    []

B replies with the revocation preimage which invalidates its old
commitment transaction, and a signature for the new commitment
transaction which includes one update from A:

             <--------- REVOCATION
             <--------- SIG 1

A receives this signature and now commits and sends its revocation
preimage for its old commitment transaction:

    Committed: [X Y]
    Staged:    []
             REVOCATION --------->

There are several alternate timing scenarios: if B decided to commit
in parallel to A, before receiving A's update:

           NODE A                            NODE B
                                              Committed: []
                                              Staged:    [Y]
                           <----------- ADD HTLC Y
    Committed: []                                  
    Staged:    [Y]
    
    
    Committed: []                             Committed: [Y]
    Staged:    [Y X]                          Staged:    []
                ADD HTLC X -------  --- SIG 0
                                  \/
    Committed: [Y X]              /\
    Staged:    []                /  -->
                     SIG 1 ---  /             Committed: [Y]
                              \/              Staged:    [X]               
                              /\   
                           <--  ------> 
    Committed: [Y]
    Staged:    [X]
    
                REVOCATION ----------->

                           <----------- REVOCATION
                                              Committed: [Y X]     
                                              Staged:    []
                           <----------- SIG 1
    Committed: [Y X]
    Staged:    []
                REVOCATION ----------->

Here, A received the signature and commits B's update but not its own
(because B didn't acknowledge it); later B responds with a signature
including A's update and A commits that update too.

## Risks With HTLC Timeouts ##

HTLCs tend to be chained across the network.  For example, a node A
might offer node B an HTLC with a timeout of 3 days, and node B might
offer node C the same HTLC with a timeout of 2 days.

This difference in timeouts is important: after 2 days B can try to
remove the offer to C even if C is unresponsive, by broadcasting the
commitment transaction it has with C and spending the HTLC output.
Even though C might race to try to use its R preimage at that point to
also spend the HTLC, it should be resolved well before the 3 day
deadline so B can either redeem the HTLC off A or close it.

If the timing is too close, there is a risk of "one-sided redemption",
where the R preimage received from an offered HTLC is too late to be
used for an incoming HTLC, leaving the node with unexpected liability.

However, there is an additional relative delay which needs to be
considered; if the connection fails, the node is forced to broadcast
the latest commitment transaction to the blockchain.  It will not be
able to reclaim timed-out HTLC funds until `delay` (as specified by
the other node's `open_message`) has passed.  Thus the actual timeout
of the HTLC is the greater of `expiry`, and the current time plus
`delay`.  In addition, there will be some additional delay for the
transaction which redeems the HTLC output to be irreversibly committed
to the blockchain.

Thus a node MUST estimate the deadline for successful redemption for
each HTLC it offers.  A node MUST NOT offer a HTLC after this
deadline, and MUST fail the connection if an HTLC which it offered is
in either node's current commitment transaction past this deadline.

## Adding an HTLC ##

Either node can offer a HTLC to the other, which is redeemable in
return for a hash preimage (sometimes referred to as R).  Amounts are
in millisatoshi, though on-chain enforcement is only possible for
whole satoshi amounts: in commitment transactions these are rounded
down as specified in [3].

A node MUST NOT offer `amount_msat` it cannot pay for in both
commitment transactions at the current `fee_rate` (see "Fee
Calculation" ).  A node SHOULD fail the connection if this occurs.
`amount_msat` MUST BE greater than 0.

A node MUST NOT add a HTLC if it would result in it offering more than
1500 HTLCs in either commitment transaction.  At 32 bytes per HTLC
output, this is comfortably under the 100k soft-limit for standard
transaction relay.

A node SHOULD NOT offer a HTLC with a timeout less than `delay` in the
future.  See also "Risks With HTLC Timeouts".

### update_add_htlc message format ###

    message update_add_htlc {
      // Amount for htlc (millisatoshi)
      required uint32 amount_msat = 2;
      // Hash for HTLC R value.
      required sha256_hash r_hash = 3;
      // Time at which HTLC expires (absolute)
      required locktime expiry = 4;
	  // Onion-wrapped routing information.
	  required routing = 5;
    }

## Removing an HTLC: update_fulfill_htlc and update_fail_htlc ##

For simplicity, a node can only remove HTLCs added by the other node.
There are three reasons for removing an HTLC: it has timed out, it has
failed to route, or the R preimage is supplied.

A node SHOULD remove an HTLC as soon as it can; in particular, a node
SHOULD fail an HTLC which has timed out, otherwise it risks connection
failure (see "Risks With HTLC Timeouts").

A node MUST check that the `index` is less than the number of HTLCs it
has offered in the current commitment transaction, and MUST fail the
connection if it does not.  The `index` refers to the current HTLCs in
the order they were offered.

A node MUST check that the `r` value in `update_fulfill_htlc` hashes
to the corresponding HTLC, and MUST fail the connection if it does not.

The `reason` field is an opaque encrypted blob for the benefit of the
original HTLC initiator as defined in [4].  A node which closes an
incoming HTLC in response to an `update_fail_htlc` message on an
offered HTLC MUST copy this field to the outgoing `update_fail_htlc`.

### update_fulfill_htlc message format ###

    // Complete your HTLC: I have the R value, pay me!
    message update_fulfill_htlc {
      // Which HTLC (index into current HTLCs in the order offered)
      required uint32 index = 1;
      // HTLC R value.
      required sha256_hash r = 2;
    }
    
### update_fail_htlc message format ###
	
    message update_fail_htlc {
      // Which HTLC (index into current HTLCs in the order offered)
      required int32 index = 1;
	  // Reason for failure (for relay to initial node)
	  required fail_reason reason = 2;
    }

## Updating Fees: update_fee ##

An `update_fee` message is used for a node to specify the fee for its
next commitment transaction.

A node SHOULD track bitcoin fees independently, and SHOULD send an
`update_fee` message whenever they change significantly.  It MAY
simply send an `update_fee` message on every new bitcoin block.

A node MUST update bitcoin fees if it estimates that the current
commitment transaction will not be processed in a timely manner (see
"Risks With HTLC Timeouts").

The receiving node MAY fail the connection if the `update_fee` is too
high.

As the commitment transaction is only used in failure cases, it is
suggested that `fee_rate` be twice the amount estimated to allow entry
into the next block, and that nodes accept a `fee_rate` up to ten
times that same estimate.

### update_fee message format ###
	
    message update_fee {
      required uint32 fee_rate = 1;
    }

## Signing HTLCs So Far: update_commit and update_revocation ##

When a node wants to update the commitment transaction to include the
staged changes, it generates the other node's commitment transaction with those changes, signs it and sends an `update_commit` message:

* `sig`: the signature using the private key corresponding to `commit_key` for the receiving node's commitment transaction.

A node MUST NOT send an `update_commit` message which does not include any updates.  Note that a node MAY send an `update_commit` message which only alters the fee.

The receiving node creates its own new commitment transaction
with the last `fee_rate` it has acknowledged, all the other node's staged changes and its own
staged changes up to and including the message acknowledged by the `acknowledge` header.  The receiver MUST
check the signature is valid for that transaction.

The receiver then responds with an `update_revocation` message which
contains the preimage for its old commitment transaction.

* `revocation_preimage`: the SHA256() of this value is the revocation hash for the sender's old commitment transaction.
* `next_revocation_hash`: the hash of the revocation for this node's next commitment transaction.

The receiver of `update_revocation` MUST check that the SHA256 hash of
`revocation_preimage` matches the previous commitment transaction, and
MUST fail if it does not.

Nodes MUST NOT broadcast old (revoked) commitment transactions; doing
so will allow the other node to seize all the funds.  Nodes SHOULD NOT
sign commitment transactions unless it is about to broadcast them (due
to a failed connection), to reduce this risk.

### update_commit message format ###

    // Commit all the staged HTLCs.
    message update_commit {
      // Signature for your new commitment tx.
      required signature sig = 1;
    }

### update_revocation message format ###

    // Complete the update.
    message update_revocation {
      // Hash preimage which revokes old commitment tx.
      required sha256_hash revocation_preimage = 1;
      // Revocation hash for my next commit transaction
      required sha256_hash next_revocation_hash = 2;
    }

## Fee Calculation ##

Each node maintains a separate current fee rate for its own commitment
transaction; if it needs to use the commitment transaction the fee
will ensure timely inclusion in a block.

For simplicity and fairness, the nodes split the payment of the
commitment transaction fee evenly where possible; for robustness, one
node will make up the difference if necessary.

As the core commitment transaction size can vary depending on the
number of outputs and the DER-encoding of signatures, we use the
following worst case values:

1. The input script is 221 bytes long:
    * redeemscript is 71 bytes ("2" `key1` `key2` "2" OP_CHECKMULTISIG)
    * signatures are 73 bytes each (the worst case)
	* extra 0 push op for the multisig is 1 byte
    * Overhead for pushing two signatures and the redeemscript is 3 bytes

2. The input is 41 bytes long:
    * transaction id is 32 bytes long.
    * input index is 4 bytes long.
	* script length is 1 byte long.
	* sequence number is 4 bytes long

3. The core of the transaction is 12 bytes long:
    * 4 byte version
    * 1 byte input count
    * 3 byte output count (worst case)
    * 4 byte lock time

4. Each output is a p2sh, which is 32 bytes long:
    * 8 byte amount
    * 1 byte script length
    * 23 byte p2sh script

Summing the first three gives 274 bytes, and it is also assumed that
there is an output for each node's `final_key`, even if the amount
would be dust (and thus not actually be constructed in the commitment
transaction, as detailed in [3]).  This results in an assumed fixed
overhead of 338 bytes.

A node MUST use the formula 338 + 32 bytes for every non-dust HTLC as
the bytecount for calculating commitment transaction fees.  Note that
the fee requirement is unchanged, even if the elimination of dust HTLC
outputs has caused a non-zero fee already.

The fee for a commitment transaction MUST be calculated by the
multiplying this bytescount by the fee rate, dividing by 1000 and
truncating (rounding down) the result to an even number of satoshis.

eg.  A 402-byte transaction with a `fee_rate` of 1112 has a fee of:

	402 * 1112 / 1000 = 447.024 = 446 satoshis

Note that one or both nodes may be unable to pay its share of fees
even with best of intentions, when a fee increase via `update_fee` is
acknowledged after one or more `update_add_htlc` messages.  At worst
case, the fee rate will be equal to the previous commitment
transaction.  In any case, nodes which cannot pay their fees will be
unable to add any additional HTLCs until fees change again or HTLCs
are fulfilled or failed.

Fees are divided as following:

1. If each nodes can afford half the fee from their to-`final_key`
   output, reduce the two to-`final_key` outputs accordingly.

2. Otherwise, reduce the to-`final_key` output of one node which
   cannot afford the fee to zero (resulting in that entire output
   paying fees).  If the remaining to-`final_key` output is greater
   than the fee remaining, reduce it accordingly, otherwise reduce
   it to zero to pay as much fee as possible.

3. If either to-`final_key` output is dust, eliminate it from the
   transaction.

# Channel Close #

Nodes can negotiate a mutual close for the connection, which unlike a
unilateral close, allows them to access their funds immediately and
can be negotiated with lower fees.

Closing happens in two stages: the first is by one side indicating
that it wants to clear the channel (and thus will accept no new
HTLCs), and once all HTLCs are resolved, the final channel close
negotiation begins.

## Closing initiation: close_clearing ##

Either node (or both) can send a `close_clearing` message to initiate closing.

A node MUST NOT send a `update_add_htlc` after a `close_clearing`, and
must not send more than one `close_clearing`.  A node MUST NOT send an
`update_add_htlc` after sending an `update_commit` with an
`acknowledge` header field acknowledging the other node's `close_clearing`.

A node MUST respond with `update_fail_htlc` to any HTLC received after it sent `close_clearing`.

### close_clearing message format ###

    // Start clearing out the channel HTLCs so we can close it
    message close_clearing {
    }

## Closing negotiation: close_signature ##

Once clearing is complete the final current commitment transactions
will have no HTLCs, and fee negotiation begins.  Each node chooses a
fee and signs the close transaction with that fee, and sends the
signature.  The process terminates when both agree on a fee, or one
side fails the connection.

Nodes SHOULD send a `close_signature` message after `close_clearing` has
been received or acknowledged, and no HTLCs remain in either
commitment transaction:

* `close_fee`: the fee to offer for the close transaction (in satoshis).
* `sig`: the signature for the close transaction with that fee.

The sender MUST set `close_fee` lower than or equal to the fee of the
final commitment transaction, and MUST set `close_fee` to an even
number of satoshis.

The sender SHOULD set the initial `close_fee` according to its
estimate of cost of inclusion in a block.  Note that there is no
security issue if the closing transaction is delayed, and it will be
broadcast very soon, so there is usually no reason to pay a premium
for rapid processing.

The receiver MUST check `sig` is valid for the close transaction, and
MUST fail the connection if it is not.

If the receiver agrees with the fee, it SHOULD reply with a
`close_signature` with the same `close_fee` value and sign and
broadcast that closing transaction, otherwise it SHOULD propose a
value strictly between the received `close_fee` and its
previously-sent `close_fee`.

Once a node has sent or received a `close_signature` with matching
`close_fee` it SHOULD close the connection.

### close_signature message format ###

    message close_signature {
		// Fee in satoshis.
		required uint64 close_fee = 1;
		// Signature on the close transaction.
		required signature sig = 2;
    }

# Revocation Hashes #

Revocation hashes are used to allow invalidation of old commitment
transactions after a new one has been negotiated: the output scripts
of a commitment transaction allow the other node to immediately spend
the output if they have the revocation preimage.

For efficiency, the series of revocation preimages are generated from
a single seed, which allows the receiver to compactly store them (see
[5]).

A node MUST select an unguessable 256-bit seed for each connection,
and MUST NOT reveal the seed.  Up to 2^64-1 preimages can be generated;
the first preimage used MUST be index 18446744073709551615, and then
the index decremented.

The preimages P for index N MUST match the output of this algorithm:

    generate_from_seed(seed, N):
        P = seed
        for B in 0 to 63:
            if B set in N:
                flip(B) in P
                P = SHA256(P)
        return P

Where "flip(B)" alternates the B'th least significant bit in the value P.

The receiving node MAY store all previous R values, or MAY calculate
it from a compact representation as described in [5].

# On The Blockchain #

The blockchain is used to enforce commitments in the case where
communication or cooperation breaks down between two nodes.  This is
slower and more expensive that using off-chain signature exchanges,
but vitally important for correct operation.

## Monitoring the Blockchain ##

Once the anchor transaction is broadcast, a node MUST monitor for
transactions which spend the anchor transaction, and if seen it MUST
fail the connection if not already failing or closed.

If a node sees a revoked commitment transaction, it MUST use the
revocation preimage to spend every output well before `delay` as
specified in the `open_channel`.  A node MAY need to use multiple
transactions to spend the outputs, and MUST ensure that these
transactions are each less than 100,000 bytes.

A node MUST continue to monitor for (and react to) additional
transactions until one transaction is deeply buried (usually
considered to be between 6 and 100 blocks).

## On-chain HTLCs ##

When a valid commitment transaction is broadcast, any HTLC outputs
must be monitored and handled as follows:

For each HTLC output this node offered:

1. If it is spent (by the other node), the spending transaction
reveals the R value which this node MUST use to redeem the corresponding
incoming HTLC, if any.
2. If it is timed out, this node MUST spend the output to itself (possibly after a delay).

For each HTLC output offered by the other node:

1. If we know the R value, the node MUST spend the output to itself (possibly after a delay).
2. Otherwise, if this node has no outgoing HTLC using the same the R value, it SHOULD ignore the output.

Note that for a node broadcasting its own commitment transaction,
there is an additional OP_CHECKSEQUENCEVERIFY delay (correponding to
the other node's `open_channel` `delay` field) before it can spend the
output.

## Failing The Connection ##

Failure can happen under various circumstances, including protocol
failures, unreachability, timeouts or deliberate abort decisions.

A node MUST NOT fail a connection simply because communication is lost
or corrupted, thought it MAY fail after a timeout.

A node MUST fail the connection if it receives an `err` message, and
MUST NOT send an `err` message in this case.  For other connection
failures, a node SHOULD send an informative `err` message.

The behaviour when failing a connection depends on the state:

1. If no anchor has been broadcast, nothing need be done.
2. If no HTLC was ever created, the latest commitment transaction
SHOULD be broadcast.
3. If a valid `close_signature` was received, the node SHOULD use
`sig` to create a close transaction, which SHOULD be broadcast.  (The last `close_signature` is closest to our desired fee).
4. Otherwise, the node SHOULD sign and broadcast the latest commitment
   transaction.

In the last two cases, the node MUST continue to monitor the
blockchain for invalidated commitment transactions as in "Monitoring
The Blockchain".

### err message format ###

    // This means we're going to hang up; it's to help diagnose only! 
    message error {
      optional string problem = 1;
    }

# Security Considerations #

Many.  Try not to Gox anyone.

# References #

[1] https://github.com/google/protobuf

[2] BOLT #1: Inter-node Encryption and Authentication

[3] BOLT #3: Transaction formats

[4] BOLT #4: Onion Routing And Error Formats

[5] shachain design: https://github.com/rustyrussell/ccan/blob/master/ccan/crypto/shachain/design.txt

# Acknowledgements #

FIXME: Too many to list.

# Feedback #

Feedback is welcome on the [lightning-dev list](https://lists.linuxfoundation.org/mailman/listinfo/lightning-dev).
