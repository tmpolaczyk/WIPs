<pre>
    WIP: WIP-Gorka-Irazoqui-ARS-merkelization
    Layer: Consensus (soft fork)
    Title: ARS merkelization
    Author: Gorka Irazoqui <gorka@witnet.foundation>
    Discussions-To: `#dev-general` channel on Witnet Comunnity's Discord server
    Status: Draft
    Type: Standards Track
    Created: 2020-06-10
    License: BSD-2-Clause
</pre>

## Abstract

This proposal introduces an upgrade that is key to achieve a full decentralized interacion with smart contract platforms by enabling Proof of Memberships with respect to the Active Reputation Set (ARS), i.e., being able to demonstrate that a particular identity was part of the ARS at a particular point in time.

This feature is achieved by merkelizing the ARS members keys and appending such merkle root in the superblocks that will be relayed to the Block Relay. In order to prove that an identity was part of the ARS, it suffices to provide a merkle path leading to the merkle root stored in ths superblock. This enables to perform selective signature aggregation, in this case, aggregating signatures generated by ARS members.

## Rationale
In Witnet, the consensus is dependent on the reputation that node holds, a metric that quantifies the honest participation (i.e., agreeing with the majority) of the nodes in previous data requests. These nodes will be part of a group of members called the Active Reputation Set (ARS).  Under BFT assumptions we can expect 2/3 of such nodes to be honest, and therefore a block header validity in a smart contract platform will likely depend on the ability to demonstrate that those 2/3 agree on a specific value. This can be ensured by, e.g., making sure these members sign block headers.

Howver, for the smart contract to determine the validity of the block header it first needs to validate that the signature was signed by a legitimate ARS member. At the moment the smart contract platform does not posses access to such information, as it only resides in the Witnet chain.

In other words, the smart contract storing the block headers requires the ability to validate Proof of Memberships. The bridge nodes in this case are in charge of generating the proofs each ARS member they want to add a vote for. Nevertheless, such a scheme requires some kind of information about the ARS to be stored together with the block header.

This proposal describes how to achieve Proof of Memberships by merkelizing the ARS and appending the root to the block header. With such a scheme, demonstrating that a particular identity belongs to the ARS is as easy as providing the merkle path leading to the merkle root. In the following sections we refer to block headers as superblocks, which are simply groups of block headers that are join together in one single superblock block header.

## Specification
### Forming a ARS with public keys in G2
The current implementation of BLS signatures in the node assumes that the verification to be done in Ethereum is *e(S, G<sub>2</sub>) = e(H(M), P)*, where:

- *S* is an individual/aggregated signature in G1
- *G<sub>2</sub>* is the generator in the field G2
- *H(M)* is the output of a hash to point function that transforms a message *M* and outputs a point in G1.
- *P* is an individual/aggregated public key in G2.

Therefore, the public keys to be aggregated are those in G2, and that is exactly the reason why the public keys in G2 need to be committed alongside eligibility proofs. With those public keys in G2, the ARS is composed of the public keys of ARS members in G2.

Each of the public keys is composed of 65 bytes in compressed form, and 128 bytes in uncompressed form. Now the question that follows is, should we use the compressed or the uncompressed form to from the merkle tree? To answer the question, we should take a look at how we can calculate the uncompressed from the compressed form. In the case of G2, the uncompressed form calculation involves complex operations in G2. Note that these public keys will need to be provided to the smart contract verifying the ARS membership. Thus, for the moment we decided that such computations are too complex to perform in the Ethereum EVM.

For the aforementioned reasons, the public keys in G2 will need to be merkelized in **uncompressed form**. These keys can still be passed and committed in compressed form by the ARS members, but need to be uncompressed prior to composing the ARS merkle root.

### Order of the leaves

Two possible ordering scenarios are considered for the leaves before computing the merkle root. These options are:

- 1. **ARS ordered by reputation**: allows us to weight votes from those members being more reputed and to standarize the ordering of the members regardless of the curve/public key  associated with it.

- 2. **Public key based order**: this option allows to perform non-membership proofs. However, in the presence of several curves such a feature would require to have a different root per supported curve.

For the case of the bridge we choose to perform a **reputation based ordering**. The reasons are simple: we do not expect the merkle tree to grow to a point up to which non-membership proofs can give us a clear advantage by, e.g., using optimistic rooll-up schemes. In case of tie in reputation, the keys are ordered by pkh. Note that this has little relevancy with respect to the bridge, as long as all nodes have common ordering instructions.

In this case we choose to utilize the 1024 bit public keys in bytes as the leaves. The reason is the reduction on hash functions that need to be computed in the Ethereum smart contract. While the keys being transfered over commits and blocks are compressed (65 bytes), nodes should convert this to uncompressed form prior to generating the ARS.

Therefore our merkle tree looks like this:


                                ARS root
            H1             H2               H3              H4
      pk1       pk2   pk3       pk4   pk5        pk6   pk7       pk8


### Superblock ARS storage
We also need to take into account that the superblock cannot be built taking into account the current ARS. The current Witnet node's finality is 1 epoch and in consequence at least the ARS for the previous block needs to be considered and stored. **Therefore a new superblock will be formed with the merkle root of the prior to last epoch's ARS**

Aditionally the last superblock's ARS also needs to be stored. In Ethereum, signatures will be validated against the previous superblocks ARS, and therefore this needs to be stored for two reasons:

- First, so that ars members from the previous superblock can propose the new superblock.
- Second, such that bridge nodes can compute compact merkle proofs against it.

Currently the previous superblock ARS in form of pkhs is being stored in the superblock state. However this is not sufficient. Note that whenever a previous superblock's ARS member is checked against the previous ARS, the pkh is checked. However, the bridge node needs to compute merkle proof against bn256 keys. The node contains a mapping going from pkh to bn256, but if for some reason some node changes the bn256 key between superblocks, the bridge would compute a different root than the one generated when the superblock was built. Therefore, in Superblockstate we also need to store the **ordered bn256 ARS** based on reputation. The key point to note here is that reputation changes from one epoch to the other, and therefore the order cannot be inferred by the bridge node.


### Changes in the code

In summary, a new field in the superblock state being the ordered bn256 public key:

```rust
{
pub struct SuperBlockState {
    ...
    // The last ARS ordered keys
    previous_ars_ordered_keys: Vec<Bn256PublicKey>,
}
```

Two new fieldS in the chain state, 

```rust
{
pub struct ChainState {
    ...
    /// Last ARS pkhs
    pub last_ars: Vec<PublicKeyHash>,
    /// Last ARS keys vector ordered by reputation
    pub last_ars_ordered_keys: Vec<Bn256PublicKey>,
}

```

A new field in the Superblock structure:

```rust
pub struct SuperBlock {
    /// Number of ars members,
    pub ars_length: u64,
    ...
}
```

A new argument to build_superblock

```rust
pub fn build_superblock(
        ..
        ars_ordered_keys: &[Bn256PublicKey],
        ..
    )
```

Before block condolidation in the chain manager, should store the ARS and ordered alts withing the chain state:

```rust
 // last ars with previous block ars info
                self.chain_state.last_ars = current_ars;
                self.chain_state.last_ars_ordered_keys = ordered_alts;
```

When building the superblock, this information should be passed to the build_superblock function

```rust
let ars_members = self.chain_state.last_ars.clone();
let ars_ordered_keys = self.chain_state.last_ars_ordered_keys.clone();
self.superblock_creating_and_broadcasting(
      ctx,
      current_epoch,
      superblock_period,
      ars_members,
      ars_ordered_keys,
      genesis_hash,
);
```

### Other concerns
This WIP does not resolve the following concerns, which should be addressed in following proposals:

- Rogue-key attacks: The current ARS merkelization and signature aggregation mechanisms are vulnerable to rogue key attacks. A potential fix would be to merkelize Hash_point(P<sub>i</sub>), adding non-linearity to the signature aggregation by P<sub>aggr</sub> =  Hash_point(P<sub>1</sub>)xP_1 + Hash_point(P<sub>2</sub>)xP_2.
- Efficiency in Ethereum: The current proposal implies aggregating keys in G2, which is an expensive operation in the EVM. The proposal can be improved by merkelizing G1 keys, aggregating in G2 and checking in Ethereum whether e(G1, P_aggr) = e(P<sub>1</sub>+P<sub>2</sub>, G2) still holds. The latter is a subsidized operation in Ethereum.
- Additional curves: The current ARS merkelization only covers the merkelization of keys in one particular curve. If additional curves need to be included, one can perform a merkelization of each ARS members public keys and take those roots as leaves for the ARS merkelization.

## Backwards Compatibility Assessment

### Changes in the data structures

This proposal introduces significant changes to existing data structures, all of which are internal and not consensus critical. No validation nor consensus rules are affected after this change.

On the basis of the above, there is more than enough evidence to assert that the content of this proposal is  backwards compatible from a validations and consensus perspective.

#### Breaking changes

The data structures affected by this proposal belong to the Protocol Buffers schema definition, and therefore the changes hereby proposed may break existing libraries and clients.

### Examples and Documentation

The data structures affected by this proposal are internal to the block chain protocol and the peering protocol. Therefore this proposal triggers no update on existing examples or documentation.

## Reference Implementation

A complete reference implementation is available in the pull request [1282](https://github.com/witnet/witnet-rust/pull/1282) of the [witnet-rust] GitHub repository.

## Acknowledgements

This proposal has been cooperatively devised by many individuals from the Witnet development community.