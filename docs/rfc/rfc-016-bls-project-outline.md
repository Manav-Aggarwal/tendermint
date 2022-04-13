# RFC 016: Add Signature Aggregation to Tendermint

## Changelog

- 01-April-2022: Initial draft (@williambanfield).

## Abstract

## Background

### Glossary

The terms that are attached to these types of cryptographic signing systems
become confusing quickly. Different sources appear to use slightly different
meanings of each term and this can certainly add to the confusion. Below is
a brief glossary that may be helpful in 

* **Multi-Signature**: A signature generated over a single message
where, given the message and signature, a verifier is able to determine
that each party signed the message as well as which parties signed the message.
May be short or may vary with number of signers.
* **Aggregated Signature**: A _short_ signature generated over messages with
possibly different content where, given the messages and signature, a verifier
should be able to determine that each party signed the designated messages.
* **Threshold Signature**: A _short_ signature generated from multiple signers
where, given a message and the signature, a verifier is able to determine that
a large enough share of the parties signed the message. The identifies of the
parties that contributed to the signature are not revealed.
* **BLS Signature**: An elliptic-curve pairing based signature system that
has some nice properties for short multi-signatures. May stand for
*Boneh-Lynn-Schacham* or *Barreto-Lynn-Scott* depending on the context.
* **Interactive**: Cryptographic scheme where parties need to perform one or
more request-response cycles to produce the cryptographic material.
* **Non-interactive**: Cryptographic scheme where parties do not need to
perform any request-response cycles to produce the cryptographic material.

### Adoption

* Algorand is working on an implementation.
* [Zcash][zcash-adoption] has adopted BLS12-381 into the protocol.
* [Ethereum 2.0][eth-2-adoption] has adopted BLS12-381 into the protocol.
* Chia Network
https://tools.ietf.org/id/draft-yonezawa-pairing-friendly-curves-02.html#adoption


### What systems may be affected by adding aggregated signatures?

#### Gossip

Gossip could be updated to aggregate vote signatures during a consensus round.
This appears to be of frankly little utility. Creating an aggregated signature
is not a fast operation, so frequently re-aggregating will incur a significant
overhead. [IS THIS TRUE?]

Additionally, each validator will still need to receive vote extension data
from the peer validators in order for consensus to proceed. As a result, any
advantage gained by aggregating signatures across the vote message will be
nullified as a result of the addition of vote extensions.

While there may be gains to be made in the gossip layer, there are likely to
be many more gains in improving the structure of our gossip code and improving
the interactions between the gossip code and the consensus state machine.

#### Block Creation

When creating a block, the proposer may create a small set of short
multi-signatures and attach these to the block instead of including one
signature per validator.

#### Block Verification

Verification of blocks would not verify a set of many signatures. Verification
would instead check the single multi-signature using the public keys stored
by the validator. Currently, we verify each validator signature using
the public key associated with that validator.

#### IBC Packet Relaying

IBC would no longer need to transmit a large set of signatures and would
instead just transmit the aggregated signature across when updating state.
In my understanding, to update the state of a IBC channel, you need to submit
a Tendermint Header, complete with commit signatures, to the chain. Adding
BLS signatures would mean creating a new signature type that could be
understood by the IBC module and relayer.

## Discussion

### What are the proposed benefits to aggregated signatures?

#### Reduce Block Size
* How big are commits now per validator? Well, it scales linearly with num vals
	BlockIdFlag      ValidatorAddress Timestamp        Signature
	32      20 * 8 bit + 64+32+ 512
	(both secp and ed are 512 bits long)
* How large would sig be without duplicated information?
	512 * NUM_VALIDATORS
	* Hub this is 150 * 512 = 9.6 kb per block
	* 10045594 * 9.6 = 96.45 GB
* How long would single sig be? 
* 96 bytes for public keys, and 48 bytes
   for signatures
	384 x 2 + encoding of S (150 bits) = 1174
	// why did I assume I need 2 signatures????
	// One for 'yes', one for 'no'

	(1174 * 10045594) = 1.47 GB for the whole chain.
$ .026 per GB on GCP = a savings of $ 2.46 a month

#### Reduce IBC Packet Size

#### Reduce Light-Client Verification Time

#### Reduce Signature Verification Time

#### Reduce Gossip Bandwidth

##### Vote

It is possible to aggregate signatures during voting and not need to gossip all 
*n* validator signatures to all other validators. Theoretically, subsets of
the signatures could be aggregated during consensus to produce vote messages
carrying aggregated signatures.

*Q*: Can you disaggregate signatures?
*Q*: Can you aggregate a signature twice?

##### Block

* Speed of signature verification
* What speed is claimed by signature verification for by our signature aggregation library?
* What speed is claimed for signature verification and production by BLST?
   BLS takes 370 and 2700 micro-seconds to sign
   and verify a signature. - IETF


FROM [IETF][bls-ietf] DOC ON BLS
   The following comparison assumes BLS signatures with curve BLS12-381,
   targeting 128 bits security.

   For 128 bits security, ECDSA takes 37 and 79 micro-seconds to sign
   and verify a signature on a typical laptop.  In comparison, for the
   same level of security, BLS takes 370 and 2700 micro-seconds to sign
   and verify a signature.

   In terms of sizes, ECDSA uses 32 bytes for public keys and 64 bytes
   for signatures; while BLS uses 96 bytes for public keys, and 48 bytes
   for signatures.  Alternatively, BLS can also be instantiated with 48
   bytes of public keys and 96 bytes of signatures.  BLS also allows for
   signature aggregation.  In other words, a single signature is
   sufficient to authenticate multiple messages and public keys.

### What are the drawbacks to aggregated signatures?

### How do we, currently, learn about a PK?

PubKeys are returned from `FinalizeBlock` in the [validator updates][validator-updates]
or contained in the [GenesisValidators][genesis-validators] list. It would be
incredibly straightforward to modify this validator data type to also include
an additional proof-of-possession.

### Why POP?
Need to ensure that every public key is accompanied by a known private key.
Otherwise, you are vulnerable to an attacker constructing an aggregated
key for which it does not know all private keys but can perform actions
that will verify against a pubkey for which it does not have the private key


3.3.  Proof of possession

   A proof of possession scheme uses a separate public key validation
   step, called a proof of possession, to defend against rogue key
   attacks.  This enables an optimization to aggregate signature
   verification for the case that all signatures are on the same
   message.

   The Sign, Verify, and AggregateVerify functions are identical to
   CoreSign, CoreVerify, and CoreAggregateVerify (Section 2),
   respectively.  In addition, a proof of possession scheme defines
   three functions beyond the standard API (Section 1.4):

   *  PopProve(SK) -> proof: an algorithm that generates a proof of
      possession for the public key corresponding to secret key SK.

   *  PopVerify(PK, proof) -> VALID or INVALID: an algorithm that
      outputs VALID if proof is valid for PK, and INVALID otherwise.

   *  FastAggregateVerify((PK_1, ..., PK_n), message, signature) ->
      VALID or INVALID: a verification algorithm for the aggregate of
      multiple signatures on the same message.  This function is faster
      than AggregateVerify.

   All public keys used by Verify, AggregateVerify, and
   FastAggregateVerify MUST be accompanied by a proof of possession, and

   As an optimization, implementations MAY cache the result of PopVerify
   in order to avoid unnecessarily repeating validation for known keys.

#### Heterogeneous key types cannot be aggregated

#### Do common HSMs support BLS signatures?
* yubikey: no
* Ledger: G1 but not G2, no Ledger is not supported really
	*Q* would our use of SigAg allow ledger to work here? 
* Cloud HSM: no

### Can aggregated signatures be added as soft-upgrades?

In my estimation, yes. With the implementation of proposer-based timestamps, 
all validators now produce signatures on only one of two messages:

1. A [CanonicalVote]() where the BlockID is the hash of the block or
2. A `CanonicalVote` where the `BlockID` is nil.

The block structure can be updated to perform hashing and validation in a new
way as a soft upgrade. This would look like adding a new section to the [Block.Commit][] structure
alongside the current `Commit.Signatures` field. This new field, tentatively named
`AggregatedSignature` would contain the following structure:

```proto
message AggregatedSignature {
  // yays is a BitArray representing which validators in the active validator
  // set issued a 'yay' vote for the block.
  tendermint.libs.bits.BitArray yays = 1;

  // absent is a BitArray representing which validators in the active
  // validator set did not issue votes for the block.
  tendermint.libs.bits.BitArray abstent = 2;

  // yay_signature is an aggregated signature produced from all of the vote
  // signatures for the block.
  repeated bytes yay_signature = 3;

  // yay_signature is an aggregated signature produced from all of the vote
  // signatures from votes for 'nil' for this block.
  // nay_signature should be made from all of the validators that were both not
  // in the 'yays' BitArray and not in the 'absent' BitArray.
  repeated bytes nay_signature = 4;
}
```

Adding this new field as a soft upgrade would mean hashing this data structure
into the blockID along with the old `Commit.Signatures` when both are present
as well as ensuring that the voting power represented in the new
`AggregatedSignature` and `Signatures` field was enough to commit the block
during block validation. One can certainly imagine other possible schemes for
implementing this but the above should serve as a simple enough proof of concept.

### Implementing vote-time and commit-time signature aggregation separately

Implementing aggregated BLS signatures as part of the block structure can easily be
achieved without implementing any 'vote-time' signature aggregation.
The block proposer would gather all of the votes, complete with signatures,
as it does now, and produce a set of aggregate signatures from all of the
individual vote signatures.

Implementing 'vote-time' signature aggregation cannot be achieved without
also implementing commit-time signature aggregation. This is because such
signatures cannot be dis-aggregated into their constituent pieces. Therefore,
in order to implement 'vote-time' signature aggregation, we would need to
either first implement 'commit-time' signature aggregation, or implement both
'vote-time' signature aggregation while also updating the block creation and
verification protocols to allow for aggregated signatures.

*Q*: can we disaggregate signatures?
*Q*: can we re-aggregate signatures?

### References

[line-ostracton-repo]: https://github.com/line/ostracon 
[line-ostracton-pr]: https://github.com/line/ostracon/pull/117 
[mit-BLS-lecture]: https://youtu.be/BFwc2XA8rSk?t=2521
[gcp-storage-pricing]: https://cloud.google.com/storage/pricing#north-america_2
[yubi-key-bls-support]: https://github.com/Yubico/yubihsm-shell/issues/66
[cloud-hsm-support]: https://docs.aws.amazon.com/cloudhsm/latest/userguide/pkcs11-key-types.html
[bls-ietf]: https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bls-signature-04
[validator-updates]: https://github.com/tendermint/tendermint/blob/441db32c8b9e8827eb6f8ee0f13f8013b979152f/internal/state/execution.go#L241
[genesis-validators]: https://github.com/tendermint/tendermint/blob/441db32c8b9e8827eb6f8ee0f13f8013b979152f/types/genesis.go#L31
[multi-signatures-smaller-blockchains]: https://eprint.iacr.org/2018/483.pdf
[ibc-tendermint]: https://github.com/cosmos/ibc/tree/master/spec/client/ics-007-tendermint-client
[zcash-adoption]: https://github.com/zcash/zcash/issues/2502
