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

# Table of Contents
* 1. General Protocol
   * 1.1. pkt message format
   * 1.2. Persistence and Retransmission
* 2. Channel Establishment
   * 2.1. The Initial `open_channel` message
   * 2.2. Describing the anchor transaction: `open_anchor`
   * 2.3. Accepting the Anchor Transaction: `open_commit_sig`
   * 2.4. Waiting for the Anchor: `open_complete`
* 3. Normal Operation
   * 3.1. Risks With HTLC Timeouts
   * 3.2. Adding an HTLC
   * 3.3. Removing an HTLC: `update_fulfill_htlc` and `update_fail_htlc`
   * 3.4. Updating Fees: `update_fee`
   * 3.5. Signing HTLCs So Far: `update_commit` and `update_revocation`
   * 3.6. Fee Calculation
* 4. Channel Close
   * 4.1. Closing initiation: `close_shutdown`
   * 4.2. Closing negotiation: `close_signature`
* 5. Revocation Hashes
* 6. On The Blockchain
   * 6.1. Monitoring the Blockchain
   * 6.2. On-chain HTLCs
   * 6.3. Failing The Connection
* 7. Security Considerations
* 8. References
* Appendix A. Acknowledgements
* Appendix B. Feedback

# 1. General Protocol

The wire protocol used is protobufs[1], passed over an encrypted
channel set up and used as specified in BOLT#1[2].

In general, if a node received a field it does not understand, if the
field number is odd it MUST ignore it, if it is even it MUST treat it
as an error and fail the connection (also known as "it's OK to be
odd").  A node MUST NOT send out an even-numbered field not listed
here without prior negotiation for its acceptance.

## 1.1. pkt message format

    // This is the union which defines all of them
    message pkt {
      oneof pkt {
	  FIXME
      }
    }

## 1.2. Persistence and Retransmission

Because communication transports are unreliable and may need to be
re-established from time to time, the design of the transport has been
explicitly separated from the protocol.

A node MUST handle continuing a previous channel on a new encrypted
transport.  A node reconnecting after receiving or sending an
`open_channel` message SHOULD send a `reconnect` message on the new
connection immediately after it has validated the `authenticate`
message.  A node MAY forget old nodes which have not sent or received
an `open_commit_sig` message.

On disconnection, a node MUST reverse any uncommitted changes sent by
the other side (ie. `update_add_htlc`, `update_fee`,
`update_fail_htlc` and `update_fulfill_htlc` for which no
`update_commit` has been received).  A node SHOULD retain the `r`
value from the `update_fulfill_htlc`, however.

A node MUST set the `ack` field in the `reconnect` message to the the
sum of previously-processed messages of types `open_commit_sig`,
`update_commit`, `update_revocation`, `close_shutdown` and
`close_signature`.

A node MAY assume that only one of each type of message need be
retransmitted.  A node SHOULD retransmit the last of each message type
which was not counted by the `ack` field.  Before retransmitting
`update_commit`, the node MUST send appropriate `update_add_htlc`,
`update_fee`, `update_fail_htlc` or `update_fulfill_htlc` messages
(the other node will have forgotten them, as required above).

A node MAY simply retransmit messages which are identical to the
previous transmission.  A node MUST not assume that
previously-transmitted messages were lost: in particular, if there is
a previous `update_commit` message, a node MUST handle the case where
the corresponding commitment transaction is broadcast by the other
side at any time.  This is particularly important if a node does not
simply retransmit the exact same `update_commit` as previously sent.

### 1.2.1. reconnect message format.

	// We're reconnecting, here's what we've received already.
	message reconnect {
		// open_commit_sig + update_commit + update_revocation + close_shutdown + close_signature received.
		required uint64 ack = 1;
	}

# 2. Channel Establishment

Channel establishment begins immediately after authentication, and consists of each node sending an `open_channel` message, followed by one node sending `open_anchor`, the other providing its `open_commit_sig` then both sides waiting for the anchor transaction to enter the blockchain and reach their specified depth, at which point they send `open_complete`.  After both sides have sent `open_complete` the channel is established and can begin normal operation.

    +-------+                            +-------+
    |       |--(1)--  open_channel  ---->|       |
    |       |<-(2)--  open_channel  -----|       |
    |       |                            |       |
    |       |--(3)--  open_anchor   ---->|       |
    |   A   |                            |   B   |
    |       |<-(4)-- open_commit_sig ----|       |
    |       |                            |       |
    |       |--(5)-- open_complete  ---->|       |
    |       |<-(6)-- open_complete  -----|       |
    +-------+                            +-------+

If this fails at any stage, or a node decides that the channel terms offered by the other node are not suitable, see "Failing The Connection".

## 2.1. The Initial `open_channel` message

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

Each `bitcoin_pubkey` field MUST be a valid compressed 33-byte
DER-encoded bitcoin public key.

### 2.1.1. `open_channel` message format

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

## 2.2. Describing the anchor transaction: `open_anchor`

Whichever node offered the anchor (`WILL_CREATE_ANCHOR`) will initially fund the channel.  This node will create a transaction with an output redeemable by the `commit_key`s from both nodes (see [3]), which it MUST NOT broadcast.  It then sends an `open_anchor` message which allows the recipient to calculate the signature for the initial commitment transaction.

The fields of this message are:

* `txid`: the transaction ID of the anchor transaction.
* `output_index`: the index of the output which is to fund the channel.
* `amount`: the amount in satoshis of the `output_index` output of `txid`.

The receiver MAY fail the connection if `amount` is too low; the sender MUST offer an `amount` sufficient to cover the fees of both initial commitment transactions.

The sender MUST NOT offer an `amount` in excess of 4294967 satoshis;
the receiver MAY fail the connection if `amount` is greater than this
amount.  This ensures that any possible HTLC amounts (in millisatoshi)
can be represented by 32 bits.

### 2.2.1. `open_anchor` message format

    // Whoever is supplying anchor sends this.
    message open_anchor {
      // Transaction ID of anchor.
      required sha256_hash txid = 1;
      // Which output is going to the 2 of 2.
      required uint32 output_index = 2;
      // Amount of anchor output.
      required uint64 amount = 3;
    }

## 2.3. Accepting the Anchor Transaction: `open_commit_sig`

Upon accepting the `open_anchor` message, the node creates a signature for the anchor creator's initial commitment transaction, and sends it in an `open_commit_sig` message.

The receiver (ie. anchor creator) MUST fail the connection if the `commit_sig` does not sign its initial commitment transaction.  The receiver SHOULD broadcast the anchor transaction upon receipt of the signature; this ensures that it can use that signature on our initial commitment transaction to redeem the anchor funds in case of failure.

### 2.3.1. `open_commit_sig` message format

    // Reply: signature for your initial commitment tx
    message open_commit_sig {
      required signature sig = 1;
    }

## 2.4. Waiting for the Anchor: `open_complete`

Once the anchor has reached `min_depth` in the blockchain, the node sends `open_complete` to indicate it is ready to transition to normal operating mode.  A node MUST NOT begin normal operation until it has both sent and received `open_complete`.  A node MAY fail the connection if it does not receive `open_complete` in a timely manner after the other's `min_depth` is reached.

### 2.4.1. `open_complete` message format

    // Indicates we've seen anchor reach min-depth.
    message open_complete {
      // FIXME: add a merkle proof plus block headers here?
    }

# 3. Normal Operation

Once both nodes have exchanged `open_complete`, the channel can be
used to make payments via Hash TimeLocked Contracts.

Changes are sent in batches: one or more messages are sent before an
`update_commit` message, as in the following diagram:

    +-------+                            +-------+
    |       |--(1)---- add_htlc   ------>|       |
    |       |--(2)---- add_htlc   ------>|       |
    |       |                            |       |
    |       |--(3)----   commit   ------>|       |
    |   A   |                            |   B   |
    |       |<-(4)---- revocation -------|       |
    |       |<-(5)----   commit   -------|       |
    |       |                            |       |
    |       |--(6)---- revocation ------>|       |
    +-------+                            +-------+

Counterintuitively, these changes apply to the *other node's*
commitment transaction; the node only adds those changes to its own
commitment transaction when the remote node responds to the
`update_commit`.  Thus each node (conceptually) tracks:

1. *local commitment*: The current (unrevoked) commitment transaction for itself.
2. *remote commitment*: The current (unrevoked) commitment transaction for the other node.
3. Two *unacked changesets*: one for the local commitment (their proposals) and one for the remote (our proposals)
4. Two *acked changesets*: one for the local commitment (our proposals, acknowledged) and one for the remote (their proposals, acknowledged).

(Note that an implementation MAY optimize this internally, for
example, pre-applying the changesets in some cases, or simply keeping a state
associated with each HTLC for the latest commitment transactions).

As the two nodes updates are independent, the two commitment
transactions may be out of sync indefinitely.  This is not concerning:
what matters is whether both sides have committed to a particular HTLC
or not.

Here's a full example of simultaneous proposals, showing internal state:

    NODE A                     NODE B
	local: []                  local: []
	remote:[]                  remote:[]

A decides to offer a new HTLC X.  It adds it to the remote unacked
changeset, and upon receipt B adds it to its local, unacked
changeset.

	local: []                        local: []
	remote:[] unacked:+X             remote:[]
	
           ADD HTLC X --------------->
	
	                                 local:  [] unacked:+X
	                                 remote: []

B decides to offer its own new HTLC Y; adding it to the remote
unacked changeset.  A adds it to its local, unacked changeset:

	local: []                        local: [] unacked:+X
	remote:[] unacked:+X             remote:[] unacked:+Y
	
            <-------------------- ADD HTLC Y
	
	local: [] unacked:+Y
	remote:[] unacked:+X

A decides to commit its update; it applies all pending changes to the
remote commitment (remembering the unacked ones for later), and sends
the signature:

	local: [] unacked:+Y             local: [] unacked:+X
	remote:[X] (unacked:+X)          remote:[] unacked:+Y
	
         COMMIT SIG[X] -------------->
	
	                                 local:  [X] (unacked:+X)
	                                 remote: [] unacked:+Y

As shown, B applies the changes to its local commitment transaction.
It now replies with the revocation preimage; this also acknowledges
the changes up until that commit message, so it adds the old unacked
local changeset to its remote, acked changeset:

	local: [] unacked:+Y             local: [X]
	remote:[X] (unacked:+X)          remote:[] unacked:+Y acked:+X
	
           <----------------------- REVOCATION
	
	local: [] unacked:+Y acked:+X
	remote:[X]

Receiving the revocation causes A to add the local unacked changeset
(which it remembered when it send the commit message) to its remote,
acked changeset.

Now B has two outstanding changes for the remote commitment; the
unacked HTLC Y it proposed, and the acked HTLC X that A proposed.  It
can now commit those, remembering the unacked ones:

	local: [] unacked:+Y acked:+X    local: [X]
	remote:[X]                       remote:[Y X] (unacked:+Y)
	
	        <---------------- COMMIT SIG[Y X]
	
	local: [Y X] (unacked:+Y)
	remote:[X]

As shown, A applies the changes queued for its local commitment.  It
now replies with the revocation preimage which invalidates its old
commitment transaction, and the unacked local changes become the
remote acked changes:

	local: [Y X]
	remote:[X] acked:+Y
	
           REVOCATION ----------------->
	
	                                 local: [X] acked:+Y
	                                 remote:[Y X]

As shown, when B receives the revocation response it adds the unacked
remote changeset it had applied above to the acked local changeset.

Now, node A has uncommitted changes to the remote commitment, so it
sends the final commit.  As there are no unacked changes, the local
commitment is not changed.  B applies the changes to its local
commitment and responds, but again there are no unacked changes so
neither the local nor remote change:

	local: [Y X]
	remote:[X Y]
	
           COMMIT ------------------->
	
	                                 local: [X Y]
	                                 remote:[Y X]
	
           <------------------- REVOCATION

## 3.1. Risks With HTLC Timeouts

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

## 3.2. Adding an HTLC

Either node can send `update_add_htlc` to offer a HTLC to the other, which is redeemable in
return for a hash preimage (sometimes referred to as R).  Amounts are
in millisatoshi, though on-chain enforcement is only possible for
whole satoshi amounts: in commitment transactions these are rounded
down as specified in [3].

A node MUST NOT offer `amount_msat` it cannot pay for in the remote
commitment transaction at the current `fee_rate` (see "Fee
Calculation" ).  A node SHOULD fail the connection if this occurs.
`amount_msat` MUST BE greater than 0.

A node MUST NOT add a HTLC if it would result in it offering more than
300 HTLCs in the remote commitment transaction.  A node SHOULD fail the
connection if this occurs.  At 32 bytes per HTLC output, this is
comfortably under the 100k soft-limit for standard transaction relay,
and at a per-input BIP141 cost of 572[3], a transaction which spends
all inputs is comfortably under the 400k cost limit, such as a steal
transaction.

A node SHOULD NOT offer a HTLC with a timeout less than `delay` in the
future.  See also "Risks With HTLC Timeouts".

A node MUST set `id` to a unique identifier for this HTLC amongst
all past or future `update_add_htlc` messages.  A node MAY do this simply by incrementing a counter and
assuming there will never be 2^64 messages.

The sending node MUST add the HTLC addition to the unacked changeset
for its remote commitment, and the receiving node MUST add the HTLC
addition to the unacked changeset for its local commitment.

### 3.2.1. `update_add_htlc` message format

	message update_add_htlc {
	  // Unique identifier for this HTLC.
	  required uint64 id;
      // Amount for htlc (millisatoshi)
      required uint32 amount_msat = 2;
      // Hash for HTLC R value.
      required sha256_hash r_hash = 3;
      // Time at which HTLC expires (absolute)
      required locktime expiry = 4;
	  // Onion-wrapped routing information.
	  required routing route = 5;
    }

## 3.3. Removing an HTLC: `update_fulfill_htlc` and `update_fail_htlc`

For simplicity, a node can only remove HTLCs added by the other node.
There are three reasons for removing an HTLC: it has timed out, it has
failed to route, or the R preimage is supplied.

A node SHOULD remove an HTLC as soon as it can; in particular, a node
SHOULD fail an HTLC which has timed out, otherwise it risks connection
failure (see "Risks With HTLC Timeouts").

A node MUST check that `id` corresponds to an HTLC in its current
commitment transaction, and MUST fail the connection if it does not.

A node MUST check that the `r` value in `update_fulfill_htlc` hashes
to the corresponding HTLC, and MUST fail the connection if it does not.

The `reason` field is an opaque encrypted blob for the benefit of the
original HTLC initiator as defined in [4].  A node which closes an
incoming HTLC in response to an `update_fail_htlc` message on an
offered HTLC MUST copy this field to the outgoing `update_fail_htlc`.

The sending node MUST add the HTLC fulfill/fail to the unacked
changeset for its remote commitment, and the receiving node MUST add
the HTLC fulfill/fail to the unacked changeset for its local commitment.

### 3.3.1. `update_fulfill_htlc` message format

    // Complete your HTLC: I have the R value, pay me!
    message update_fulfill_htlc {
      // Which HTLC
      required uint64 id = 1;
      // HTLC R value.
      required sha256_hash r = 2;
    }
    
### 3.3.2. `update_fail_htlc` message format
	
    message update_fail_htlc {
      // Which HTLC
      required uint64 id = 1;
	  // Reason for failure (for relay to initial node)
	  required fail_reason reason = 2;
    }

## 3.4. Updating Fees: `update_fee`

An `update_fee` message is used for a node to specify the fee for *its
own* next commitment transaction, but use the same mechanism as other
updates: apply to the remote commitment first, then the local
commitment upon acknowledgement.  Thus fee changes only have an effect
when they are applied from the unacked queue.

A node SHOULD track bitcoin fees independently, and SHOULD send an
`update_fee` message whenever they change significantly.  It MAY
simply send an `update_fee` message on every new bitcoin block.

A node MUST update bitcoin fees if it estimates that the current
commitment transaction will not be processed in a timely manner (see
"Risks With HTLC Timeouts").

The sending node MUST NOT send a `fee_rate` which it could not afford
(see "Fee Calculation), were it applied to the receiving node's
commitment transaction.  The receiving node SHOULD fail the connection
if this occurs.

Note the unusual check here: the fee rate sent by node A to B will be
applied at some future point to the A's commitment transaction, but we
check the fee against B's commitment transaction for validity.  This
is because B might be adding HTLCs which haven't been received by A,
which thus doesn't know that the fee rate will be unaffortable.  This check is both simple, and deterministic.

The receiving node MAY fail the connection if the `update_fee` is too
high.

As the commitment transaction is only used in failure cases, it is
suggested that `fee_rate` be twice the amount estimated to allow entry
into the next block, and that nodes accept a `fee_rate` up to ten
times that same estimate.

The sending node MUST add the fee change to the unacked changeset for
its remote commitment, and the receiving node MUST add the fee change
to the unacked changeset for its local commitment.

### 3.4.1. `update_fee` message format
	
    message update_fee {
      required uint32 fee_rate = 1;
    }

## 3.5. Signing HTLCs So Far: `update_commit` and `update_revocation`

When a node has changes for the remote commitment, it can apply them,
sign the resulting transaction and send an `update_commit` message.
In the special case where the remote node can redeem no outputs (there
are no HTLC outputs, and no output to itself), it does not send a
signature.

* `sig`: the signature using the private key corresponding to `commit_key` for the receiving node's local commitment transaction (ie. sender's remote commitment transaction).

If the commitment transaction has only a single output which pays
to the other node, `sig` MUST be unset.  Otherwise, a sending node
MUST apply all remote acked and unacked changes except
unacked fee changes to the remote commitment before generating `sig`.

A node MUST NOT send an `update_commit` message which does not include any updates.  Note that a node MAY send an `update_commit` message which only alters the fee, and MAY send an `update_commit` message which doesn't change the commitment transaction other than the new revocation hash (due to dust, identical HTLC replacement, or insignificant or multiple fee changes).

A receiving node MUST apply all local acked and unacked changes except
unacked fee changes to the local commitment, then it MUST check `sig`
is valid for that transaction.

The receiver MUST respond with an `update_revocation` message which
contains the preimage for its old commitment transaction:

* `revocation_preimage`: the SHA256() of this value is the revocation hash for the sender's old commitment transaction.
* `next_revocation_hash`: the hash of the revocation for this node's next commitment transaction.

The node sending `update_revocation` MUST add the local unacked
changes to the set of remote acked changes.

The receiver of `update_revocation` MUST check that the SHA256 hash of
`revocation_preimage` matches the previous commitment transaction, and
MUST fail if it does not.  That node MUST add the remote unacked
changes to the set of local acked changes.

Nodes MUST NOT broadcast old (revoked) commitment transactions; doing
so will allow the other node to seize all the funds.  Nodes SHOULD NOT
sign commitment transactions unless it is about to broadcast them (due
to a failed connection), to reduce this risk.

An implementation MAY choose not to send an `update_commit` until it
receives the `update_revocation` response to the previous
`update_commit`, so there is only ever one unrevoked local commitment.

### 3.5.1. `update_commit` message format

    // Commit all the staged HTLCs.
    message update_commit {
      // Signature for your new commitment tx.
      required signature sig = 1;
    }

### 3.5.2. `update_revocation` message format

    // Complete the update.
    message update_revocation {
      // Hash preimage which revokes old commitment tx.
      required sha256_hash revocation_preimage = 1;
      // Revocation hash for my next commit transaction
      required sha256_hash next_revocation_hash = 2;
    }

## 3.6. Fee Calculation

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

The fee for a transaction MUST be calculated by multiplying this
bytecount by the fee rate, dividing by 1000 and truncating (rounding
down) the result to an even number of satoshis.

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

# 4. Channel Close

Nodes can negotiate a mutual close for the connection, which unlike a
unilateral close, allows them to access their funds immediately and
can be negotiated with lower fees.

Closing happens in two stages: the first is by one side indicating
that it wants to clear the channel (and thus will accept no new
HTLCs), and once all HTLCs are resolved, the final channel close
negotiation begins.

    +-------+                              +-------+
    |       |--(1)--  close_shutdown  ---->|       |
    |       |<-(2)--  close_shutdown  -----|       |
    |       |                              |       |
    |       | <complete all pending htlcs> |       |
    |   A   |             ...              |   B   |
    |       |                              |       |
    |       |<-(3)-- close_signature F1----|       |
    |       |--(4)-- close_signature F2--->|       |
    |       |             ...              |       |
    |       |--(?)-- close_signature Fn--->|       |
    |       |<-(?)-- close_signature Fn----|       |
    +-------+                              +-------+

## 4.1. Closing initiation: `close_shutdown`

Either node (or both) can send a `close_shutdown` message to initiate closing:

* `script_pubkey`: the output script for the close transaction.

A node MUST NOT send a `update_add_htlc` after a `close_shutdown`, and
MUST NOT send more than one `close_shutdown`.  A node SHOULD send a `close_shutdown` (if it has not already) after receiving `close_shutdown`.

A node MUST fail to route any HTLC added received after it sent `close_shutdown`.

A node MUST set `script_pubkey` to one of the following forms:

1. `OP_DUP` `OP_HASH160` `20` 20-bytes `OP_EQUALVERIFY` `OP_CHECKSIG`
   (pay to pubkey hash), OR
2. `OP_HASH160` `20` 20-bytes `OP_EQUAL` (pay to script hash), OR
3. `OP_0` `20` 20-bytes (version 0 pay to witness pubkey), OR
4. `OP_0` `32` 32-bytes (version 0 pay to witness script hash)

A node receiving `close_shutdown` SHOULD fail the connection
`script_pubkey` is not one of those forms.  This avoids the risk that
the mutual close transaction will be nonstandard.

### 4.1.1. `close_shutdown` message format

    // Start clearing out the channel HTLCs so we can close it
    message close_shutdown {
	  // Output script for mutual close tx.
      required bytes script_pubkey = 1;
    }

## 4.2. Closing negotiation: `close_signature`

Once shutdown is complete the final current commitment transactions
will have no HTLCs, and fee negotiation begins.  Each node chooses a
fee and signs the close transaction the `script_pubkey` fields from
the `close_celearing` messages and that fee, and sends the signature.
The process terminates when both agree on a fee, or one side fails the
connection.

Nodes SHOULD send a `close_signature` message after `close_shutdown` has
been received and no HTLCs remain in either commitment transaction:

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

The sender MUST set signature to the bitcoin signature of the close
transaction with `close_fee` divided as described in "Fee
Calculation".

The receiver MUST check `sig` is valid for the close transaction with
the given `close_fee`, and MUST fail the connection if it is not.

If the receiver agrees with the fee, it SHOULD reply with a
`close_signature` with the same `close_fee` value, otherwise it SHOULD
propose a value strictly between the received `close_fee` and its
previously-sent `close_fee`.

Once a node has sent or received a `close_signature` with matching
`close_fee` it SHOULD close the connection and SHOULD sign and
broadcast the final closing transaction.

### 4.2.1. `close_signature` message format

    message close_signature {
		// Fee in satoshis.
		required uint64 close_fee = 1;
		// Signature on the close transaction.
		required signature sig = 2;
    }

# 5. Revocation Hashes

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

# 6. On The Blockchain

The blockchain is used to enforce commitments in the case where
communication or cooperation breaks down between two nodes.  This is
slower and more expensive that using off-chain signature exchanges,
but vitally important for correct operation.

## 6.1. Monitoring the Blockchain

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

## 6.2. On-chain HTLCs

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

## 6.3. Failing The Connection

Failure can happen under various circumstances, including protocol
failures, unreachability, timeouts or deliberate abort decisions.

A node MUST NOT fail a connection simply because communication is lost
or corrupted, thought it MAY fail after a timeout.

A node MUST fail the connection if it receives an `err` message, and
MUST NOT send an `err` message in this case.  For other connection
failures, a node SHOULD send an informative `err` message.

The behaviour when failing a connection depends on the state:

1. If no anchor has been broadcast, nothing need be done.
2. If no commitment signature have ever been received (ie. non-funding
   side before the first HTLC), nothing need be done.
3. If a valid `close_signature` was received, the node SHOULD use
`sig` to create a close transaction, which SHOULD be broadcast.  (The last `close_signature` is closest to our desired fee).
4. Otherwise, the node SHOULD sign and broadcast the latest commitment
   transaction.

In the last two cases, the node MUST continue to monitor the
blockchain for invalidated commitment transactions as in "Monitoring
The Blockchain".

### 6.3.1. `error` message format

    // This means we're going to hang up; it's to help diagnose only! 
    message error {
      optional string problem = 1;
    }

# 7. Security Considerations

Many.  Try not to Gox anyone.

# 8. References

[1] https://github.com/google/protobuf

[2] BOLT #1: Inter-node Encryption and Authentication

[3] BOLT #3: Transaction formats

[4] BOLT #4: Onion Routing And Error Formats

[5] shachain design: https://github.com/rustyrussell/ccan/blob/master/ccan/crypto/shachain/design.txt

# Appendix A. Acknowledgements

FIXME: Too many to list.

# Appendix B. Feedback

Feedback is welcome on the [lightning-dev list](https://lists.linuxfoundation.org/mailman/listinfo/lightning-dev).
