# Basis of Lightning Technology RFC 1 #

Inter-node Encryption and Authentication

# Status #

Initial draft

# Author #

Rusty Russell, Blockstream <mailto:rusty@rustcorp.com.au>

# Abstract #

All communications between lightning nodes should be encrypted to make
analysis more difficult, and authenticated to avoid malicious
interference.  Each node has a known identifier which is a unique
bitcoin-style public key[1].

## Initial Handshake ##

The first message sent by a node is of form:

1. `length`: 4 byte little-endian
2. `sessionpubkey`: 33 byte DER-encoded compressed public EC-key

The `length` field is the length after the field itself, and MUST be
33 or greater.  `length` MUST NOT exceed 1MB (1048576 bytes).

The `sessionpubkey` field is a compressed public key corresponding to
a `sessionsecretkey`.  The receiver MUST check that `sessionpubkey` is
a valid point.

The `sessionsecretkey` MUST be is unguessable, MUST BE unique for this
session, MUST NOT be zero and MUST BE a valid EC key.

Additional fields MAY be added, and MUST be included in the `length` field.  These MUST be ignored by implementations which do not understand them.

### Derivation of the Shared Secret and Encryption Keys ###

Once a node has received the initial handshake, it can derive the
shared secret using the received `sessionpubkey` point and its own
`sessionsecretkey` scalar using EC Diffie-Hellman.

Now both nodes have obtained the shared secret, all messages are
encrypted using keys derived from the shared secret.  Keys are derived
as follows:

* sending-key: SHA256(shared-secret || sending-node-session-pubkey)
* receiving-key: SHA256(shared-secret || receiving-node-session-pubkey)

ie. each node combines the secret with its session public key to produce the key
to encrypt data it sends.

## Encryption of Messages ##

The protocol uses Authenticated Encryption with Additional Data using
ChaCha20-Poly1305[2].

Each message contains a length and a body.  `length` is a 4-byte little-endian field indicating the size of the unencrypted body.

The 4-byte length for each message is encrypted separately (resulting
in a 20 byte header when the authentication tag is appended), to
offer additional protection from traffic analysis.

The body also has a 16-byte authentication tag appended.

Nonces are 64-bit little-endian numbers, which MUST begin at 0 and
MUST be incremented after each encryption (ie. twice per message), such
that headers are encrypted with even nonces and the message bodies
encrypted with odd nonces.

## Authentication Message ##

Once the shared secret has been exchanged, the identity of the peer
has still not been authenticated.  The first message sent MUST be an
`authenticate` message:

	message authenticate {
	  // Which node this is.
	  required bitcoin_pubkey node_id = 1;
	  // Signature of your session key.
	  required signature session_sig = 2;
      // How many update_commit and update_revocation messages already received
      optional uint64 ack = 3 [ default = 0 ];
	};

The receiving node MUST check that:

1. `node_id` is the expected value for the sending node.
2. `session_sig` is a valid secp256k1 ECDSA signature encoded as a
32-byte big endian R value, followed by a 32-byte big endian S value.
3. `session_sig` is the signature of the SHA256 of SHA256 of the its
   own sessionpubkey, using the secret key corresponding to the sender's `node_id`.

The `ack` field allows reconnection of a previously-established connection.

The receiver MUST NOT examine the `ack` value until after the
authentication fields have been successfully validated.  The
`ack` field MUST BE set to the number of `update_commit`, `open_commit_sig`
and `update_revocation` messages received and processed.

Additional fields MAY be included, and MUST BE ignored if not
understood (to allow for future extensions).

## Rationale ##

Multiple choices are possible for symmetric encryption; AES256-GSM is
the other obvious choice but it is slower if there is no hardware
acceleration, and the well-supported libsodium[3] doesn't support it
on non-accelerated platforms.

The header encryption could use a different key for encryption and
eschew 16-bytes for the authentication tag, but modern APIs tend to
shy away from offering unauthenticated encryption.

While multiple choices are possible for public-key cryptography, the
bitcoin protocol already relies on the secp256k1 elliptic curve, so
reusing it here avoids additional dependencies.

The authentication signature ensures that the node owning the
`node_id` has specifically initiated this session; using double-sha256
is done because bitcoin transaction signatures use that scheme.

# Security Considerations #

It is strongly recommended that existing, commonly-used, validated
libraries be used for encryption and decryption, to avoid the many
implementation pitfalls possible.

# References #

1. https://en.bitcoin.it/wiki/Secp256k1
2. https://tools.ietf.org/html/rfc7539
3. https://download.libsodium.org/doc/index.html

# Acknowledgements #

Thanks to Olaoluwa Osuntokun for pointing out the idea of encrypting
the length, and noting that it needed a separate key if it didn't
include the authentication tag.

Thanks to Mats Jerratsch and Anthony Towns for feedback on initial
handshake design.

# Feedback #

Feedback is welcome on the [lightning-dev list](https://lists.linuxfoundation.org/mailman/listinfo/lightning-dev).
