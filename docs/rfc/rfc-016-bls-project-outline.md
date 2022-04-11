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
* **BLS Signature**:
* **Interactive**: Cryptographic scheme where parties need to perform one or
more request-response cycles to produce the cryptographic material.
* **Non-interactive**: Cryptographic scheme where parties do not need to
perform any request-response cycles to produce the cryptographic material.


### What systems may be affected by adding aggregated signatures?
* All systems that rely on verifying commits:
  * Light client
  * IBC
  * Consensus
  * Block Sync
  * State Sync ? 

* All systems that rely on producing commits:
  * Light client
  * IBC
  * Consensus
  * Block Sync
  * State Sync ? 

#### Gossip

Gossip could be updated to aggregate vote signatures during a consensus round.
This appears to be of frankly little utility. If a validator has seen 
a subset of votes

Additionally, each validator will still need to receive vote extension data
from the peer validators in order for consensus to proceed. As a result, any
advantage gained by aggregating signatures across the vote message will be
nullified as a result of the addition of vote extensions.

While there may be gains to be made in the gossip layer, there are likely to
be many more gains in improving the structure of our gossip code and improving
the interactions between the gossip code and the consensus state machine.

#### Block Creation

When creating a block, the proposer may create a multi-signature and attach
this to the block instead of including one signature per validator.

#### Block Verification

Verification of blocks would no verify a set of many signatures. Verification
would instead check the single multi-signature.

* where do we keep the 'public-key' in this system?

#### IBC Verification

IBC would no longer need to transmit a large set of signatures and would 
instead just transmit the aggregated signature across. I think this is true.

Where would it store/fetch the 'public key'?

## Discussion

### What are the proposed benefits to aggregated signatures?

#### Reduce Commit Size
* How big are commits now per validator? Well, it scales linearly with num vals
	BlockIdFlag      ValidatorAddress Timestamp        Signature
	32      20 * 8 bit + 64+32+ 512
	(both secp and ed are 512 bits long)
* How large would sig be without duplicated information?
	512 * NUM_VALIDATORS
	* Hub this is 150 * 512 = 9.6 kb per block
	* 10045594 * 9.6 = 96.45 GB
* How long would single sig be? 
	512 x 2 + encoding of S (150 bits) = 1174
	(1174 * 10045594) = 1.47 GB for the whole chain.
$ .026 per GB on GCP = a savings of $ 2.46 a month

#### Reduce Gossip Bandwidth

* Allow for smaller IBC Packets in Cosmos-> Tendermint headers will only require
one signature Perform signature aggregation during gossip to reduce total
bandwidth. 

* Speed of signature verification
* What speed is claimed by signature verification for by our signature aggregation library?
* What speed is claimed for signature verification and production by BLST?


FROM IETF DOC ON BLS
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

#### Heterogeneous key types cannot be aggregated

#### Do common HSMs support BLS signatures?
* yubikey: no
* Ledger: G1 but not G2, no Ledger is not supported really
* Cloud HSM: no

### Can aggregated signatures be added as soft-upgrades?

### Implementing vote-time and block-time signature aggregation separately

#### Separable implementation

#### Simultaneous implementation

### References

[line-ostracton-repo]: https://github.com/line/ostracon 
[line-ostracton-pr]: https://github.com/line/ostracon/pull/117 
[mit-BLS-lecture]: https://youtu.be/BFwc2XA8rSk?t=2521
[gcp-storage-pricing]: https://cloud.google.com/storage/pricing#north-america_2
[yubi-key-bls-support]: https://github.com/Yubico/yubihsm-shell/issues/66
[cloud-hsm-support]: https://docs.aws.amazon.com/cloudhsm/latest/userguide/pkcs11-key-types.html
[bls-ietf]: https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bls-signature-04
