# RFC 016: Adding Signature Aggregation to Tendermint

## Changelog

- 01-April-2022: Initial draft (@williambanfield).

## Abstract

## Background

### What is an aggregated signature?

### What systems would be affected by adding aggregated signatures?
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

#### Block Verification

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

### What are the drawbacks to aggregated signatures?

#### Heterogeneous key types cannot be aggregated

#### Do common HSMs support BLS signatures?
* yubikey: no
* Ledger: G1 but not G2
* 

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
