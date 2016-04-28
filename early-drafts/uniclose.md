# Lightning: When things go sideways

Lightning Networks allows for two parties to make transactions off-chain. How does this work? If A and B have a payment channel open together, they both hold a cross-signed *commitment tx*, which describes the current state of the channel (basically the current balance). This *commitment tx* is updated everytime a new payment is made, and is spendable at all time.

There are three ways a channel can end:
* the good way: at some point A and B mutually agree on closing the channel, they generate a *final tx* (which is similar to a *commitment tx* without any pending payments), and publish it on the blockchain
* the bad way: something goes wrong, without necessarily any evil intent on either side (maybe one of the party crashed, for instance). Anyway, one side publishes its *commitment tx*, we call this an unilateral close
* the ugly way: one of the parties deliberately tries to cheat by publishing an outdated version of the *commitment tx* (presumably one that was more in her favor), we call this a cheating attempt

Because Lightning is designed to be trustless, there is no risk of loss of funds in any of these 3 cases, provided that the situation is properly handled. The goal of this article is to explain exactly how to react to an unilateral close or a cheating attempt.

## Commitment tx

A *commitment tx* between A and B has 3 types of outputs:
* A's main output
* B's main output
* all the pending payments, otherwise referred to as *HTLCs* (they can go to A or B)

As an incentive for A and B to cooperate, an `OP_CSV` relative timeout encumbers the first two outputs. If A publishes its commitment tx, she won't be able to get her funds immediately but B will. As a consequence, A and B's *commitment txes* are not identical, they are symmetrical.

A typical *commitment tx* (from the point of view of A) looks like that:

TODO


## Unilateral close handling

Let's suppose that we have the following network, and see what happens **from the point of view of B** when the current *commitment tx* of channel B-C is published on the blockchain.

Let's see what the possible pending payments can be:
```
A                    B                    C
+--------------------+--------- X --------+

                      ------------------->  [1] payment to C initiated by B
                      
 -------------------> ------------------->  [2] payment to C forwarded by B
 
 <-------------------                       [3] payment to A initiated by B
 
 <------------------- <-------------------  [4] payment to A forwarded by B                      
```

We can immediately disregard `[3]` as it won't be impacted by something hapenning between B and C.

### `1`: payment to C initiated by B

In this case, B initated a payment to some node E, relayed by C.

There are two outcomes possible from the pov of B, depending on whether C got the R value from E and use it in time:
* C has the R value: C can spend the HTLC. There is no prejudice to B, because the fact that C has R implies that B's payment reached its destination
* C does not have the R value or does not use it in time: B will be able to get her money back after the HTLC timeout expires.

Note that if C just dies and never reappears, it may lose money, for example in this scenario:
```
    B                   C                   D                   E
    +-------------------+-------------------+-------------------+

  |  ---------h-------->                                           B initiates a payment to E
  |                      ---------h-------->                       C forwards to D
  |                                          ---------h-------->   D forwards to E (E is paid!)
t |                                          <--------r---------   E gives r to D
i |                      ? <------r---------                       *C dies*
m |                                                                D publishes the commit tx C-D and pulls money from C using r
e |                                                                B publishes the commit tx B-C
  |                                                                B waits for the HTLC to time out
  |                                                                B gets her money back
  V        
```

### `2`: payment to C forwarded by B

There are two outcomes possible from the pov of B, depending on whether C got the R value from X and use it in time:
* C has the R value and spends the HTLC. B needs to monitor the blockchain and extract the R value, which appears in clear in the scriptsig. She can then fulfill the HTLC in the B-C channel
* C does not have the R value or does not use it in time: B will be able to get her money back after the HTLC timeout expires. She better does it, otherwise she risks losing money because A will probably do the exact same thing in channel A-B

### `4`: payment to A forwarded by B

The only thing to do is wait for the R value on the A-B channel. Upon reception B needs to spend the HTLC as soon as possible on channel B-C (before it times out).


## Revoked tx handling