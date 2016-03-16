# lightning

This repository is for discussing and setting the exact requirement a node has to fulfill to successfully participate within the 'Bitcoin Lightning Network'

# Basis of Lightning Technology (BOLTs)

BOLT | Status     | Title                                    | Author        |
----:|------------|------------------------------------------|---------------|
 [1] | Draft      | Inter-node Encryption and Authentication | Rusty Russell |
 [2] | Draft      | Inter-node Wire Protocol                 | Rusty Russell |
 
[1]: bolts/01-encryption.md
[2]: bolts/02-wire-protocol.md

BOLTs are proposals on the standards track, that are being actively implemented or used.

BOLTs go through the following phases:

 * Draft: at least one implementation is actively implementing this BOLT
 * Proposal: proposal has been implemented and able to be used
 * Active: implemented and usable in at least two independent implementations of the Lightning Network, specification fixed
 * Deprecated: implemented and usable in at least one implementation, but recommended to be avoided
 * Obsolete: not in use nor recommended

The text of a BOLT may be updated to remove ambiguities or otherwise
improve explanation at any stage in their lifecycle, but deliberate
changes to requirements should only be made while the the BOLT is in
the Draft or Proposal stages.

BOLTs should list up to five of the Lightning Network implementations
that support them or are working on supporting them.

# Early Drafts

Draft ID   | Year | Title                                     | Author
-----------|------|-------------------------------------------|-----------
[shachain] | 2016 | Efficient Chains Of Unpredictable Numbers | Rusty Russell

[shachain]: early-drafts/shachain.txt

Early drafts of proposed BOLTs can be added to the repo as well. These
are ideas being considered by implementors that might or might not ever
see actual use.

Early drafts are identified by a self-assigned 8 character nickname. If
they have not been implemented and promoted to a BOLT within a year,
they will generally be deleted.

# Lightning Implementations

There are no known functional implementations of the Lightning Network
yet. There are a number of partial implementations and prototypes,
however:

Nickname     | Contact(s)                  | Companies   | Status    | Notes
-------------|-----------------------------|-------------|-----------|-------
[Amiko-Pay]  | CJP                         |             | Prototype | Python, GPLv3+
[Eclair]     | Pierre-Marie PADIOU         | ACINQ       | Prototype | SCALA, Apache 2.0
[lightningd] | Rusty Russell               | Blockstream | Prototype | C, MIT
[lnd]        | Thaddeus Dryja, Joseph Poon |             | Prototype | Go, MIT
[Thunder]    | matsjj                      |             | Prototype | Java, AGPLv3

[Amiko-Pay]: https://github.com/cornwarecjp/amiko-pay
[Eclair]: https://github.com/ACINQ/eclair
[lightningd]: https://github.com/ElementsProject/lightning
[lnd]: https://github.com/LightningNetwork/lnd
[Thunder]: https://github.com/matsjj/thundernetwork

