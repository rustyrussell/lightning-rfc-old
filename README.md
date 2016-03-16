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

# Lightning Early Access Developments, Evaluation and Review (LEADERs)

LEADER | Year | Title                               | Author
------:|------|-------------------------------------|-----------
(none) | 2016 | (none)                              | (none)

LEADERs are ideas being considered by implementors that might or might
not ever see actual use.  (In the weather phenomenon, cloud to ground
lightning bolts are usually initiated by a stepped "leader" moving down
from the cloud to ground)

LEADERs are identified by a self-assigned 8 character nickname. LEADERs
that are over a year old and have not been implemented and promoted to
a BOLT, will generally be deleted.

