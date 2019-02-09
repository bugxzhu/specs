# Aggregating sectors

- Author: Nicola

## Context

**Sector:** A `sector` is a unit of storage over which a miner performs a *Proof-of-Replication* to convince the network and their clients that they are dedicating physical storage to the data they promised to store.

**Mining Cycle:** A miner stores data from the Storage Market in a sector. When the miner fills the sector, they *seal* the sector sector. The miner then posts to the chain the the cryptographic commitments (1) of the original data, `commD`, and (2) of the sealed data, `commR`. In addition, the miner submits a convincing *proof* that the data behind `commR` is a correct encoding of the data behind `commD`. This process is called `Sector Commitment`.

**Throughput issue with current mining cycle:** For each sector sealed, the miner must submit a minimum of ~300bytes composed by the proof, `commD` and `commR`. This is an issue for smaller sector size:

- *Bottleneck to storage onboarding*: According to our [Filecoin throughput calculator](https://beta.observablehq.com/d/37ff2d55942d1354), this is a bottleneck to the amount of storage we can onboard through time.
- *Expensive to add storage*: Miners would have to pay transaction fees for each storage submission (which might be expensive, due to their size of data needed to be added).
- TODO: there might be other important issues I am not surfacing here, please add.

## Proposed solutions

There are two main strategies to go around storage onboarding throughput:

- **Proposal 1: Increasing sector size**
- **Proposal 2: Aggregating sector commitments**
- **Proposal 3: Aggregating sector commitments via an "aggregator node"**
- **Proposal 4: One seal for the entire storage**
- **Proposal 5: Make PoSts prove the seal encoding**
- **Proposal 6: Seal proofs on successful block mining**

### Proposal 1: Increasing sector size

The strategy is to increase the storage onboarding througput by a constant `T`, by increasing the sector size by `T`. For example, in order to have 20x more storage onboarded, we can have 20x larger sectors, hence one storage commitment instead of 20.

#### How does it work?

*Construction:* Simple parameter change in PoRep.

*Proof size:* `commD` (32bytes) + `commR` (32bytes)+proof(192bytes)

#### Pros & Cons & Unkowns

**Pros:**

- *Simple protocol:* Keeping the protocol as it is (it's just a param change!).
- *Small circuits:* The proving circuit for `PoRep` grows logarithmically in the size of the sector. In other words, larger sectors won't impact the proving circuit.
- *Single process*: Sealing just requires a single processing unit (e.g. CPU) to seal the large sector

**Cons:**

- *Waiting time*: The miner will have to wait to fill the sector (which is `T` times larger than the original one) with data from the storage market.

**Unknown:**

- *Slow replication time:* the minimum replication time for a sector depends on the size of the data. We might require a sector size whose minimum replication time is unacceptable for Filecoin.

### Proposal 2: Aggregating sector commitments

The strategy is to increase the storage onboarding throughput by a constant `T`, by submitting a single sector commitment for `T` sectors. For example, in order to have a 20x more storage onboard, we can seal 20 different sectors and aggregate their storage commitment into a single commitment instead of 20.

#### How does it work?

Miner runs `T` sealing in parallel, when done, uses one of the strategies below, to generate a single sector commitment. The `commDAgg` and the `commRAgg` posted on chain are vector commitments of all the aggregated `commD`s and `commR`s.

##### Strategy 1: One single proving circuit

*Construction:*

- The sealing proving circuit is repeated `T` times in the circuit. 
- Prove that `commRAgg` is constructed correctly.

*Proof size:* `commD` (32bytes) + `commR` (32bytes)+proof(192bytes)

Note1: Since a single proving circuit is already pretty large, if `T` is too large, this strategy may not be practical. Large circuits could be addressed by using systems such as Hyrax or DIZK.

##### Strategy 2: Bounded bootstrap with CommR hidden

*Construction:* We have two proving circuits:

- Circuit 1: A circuit that proves a single sealed sector.
- Circuit 2: A circuit that proves that
  - `T` sealed sectors proofs from Circuit 1 were correct.
  - Check that all the individual `commR`s to be aggregated are different and never aggregated before and were generated by the correct `minerID`
  - Check that `commRAgg` is constructed correctly

Note that if `T` is too large, Circuit 2 might be too large (hence either slow or impractical to execute).

Proof size:* `commD` (32bytes) + `commR` (32bytes)+proof(192bytes)

##### Strategy 3: Bounded bootstrap with CommR exposed

The second check in circuit 2 might be complicated, or expensive. This strategy is equivalent to strategy 2, except that all the `commR`s are public and they are checked for uniqueness outside of the circuit

Proof size:* `commD` (32bytes) + `T` * `commR` (32bytes)+proof(192bytes)

#### Pros & Cons & Unkowns

**Pros:**

- *Parallelizable:* The encoding of each sealing can be parallelized (hence we could achieve really fast sealing time)

**Cons:**

- *Processing power:* This proposal performs exactly as the status quo (1GB sector), but requires more processing power than Proposal 1.
- *Complicated proof:* Depending on the construction (in particular strategy 2 and 3) require more complicated circuits and more complicated SNARKs.
- *Might not scale as well*: The most practical strategy (Strategy 3) requires `T` `commR`s to be posted on chain.
- *Waiting time*: The miner will have to wait to fill `T` sectors before posting the commitments on chain.

### Proposal 3: Aggregating sector commitments via an "Aggregator node"

This proposal is similar to proposal 2: instead of a storage miner aggregating their own sectors, different storage miners send their commitments to an "aggregator node" which aggregates them and post them on chain. This solution solves the problem of the "waiting time".

There are two strategies:

- Strategy 1: One single proving circuit
- Strategy 2: Bounded bootstrap with  CommR exposed

#### Pros & Cons & Unkowns

**Pros:**

- *Miners do not run aggregation*
- *No SEAL SNARKs for miners*: Using Strategy 1 (and with some tricks Strategy 2), miners won't have to run the SEAL SNARK, since they could delegate it to the SNARK prover

**Cons:**

- *Complicated proof:* Depending on the construction (in particular strategy 2) require more complicated circuits and more complicated SNARKs.
- *Might not scale as well*: The most practical strategy (Strategy 3) requires `T` `commR`s to be posted on chain.
- *Incentivizing aggregator nodes*: Why would a node be the "aggregator node"?

### Proposal 4: One seal proof for the entire storage

The idea is to generate a single SEAL proof for the entire storage, without aggregation.

There are two strategies:

- Strategy 1: One big sector (TO BE DOCUMENTED)
- Strategy 2: One proof for multiple sectors (TO BE DOCUMENTED)

#### Security notes

*Idea*: if a prover generates a SEAL Proof for each sector, then, there is a small percentage (e.g. 1%) of data in that sector that the malicious participant is not encoding or faking and they can be spot with something (e.g. 10%).

- *Case 1 (what we have today):* if we require the miner to seal 1GB sectors and then to generate one seal proof per sector, then, it's 1% of each sector, which is fine.
- *Case 2 (@why suggestion-ish):*, if we require the miner to seal 1GB sectors and to generate one single seal proof for all the storage, then the prover might not have encoded 1% of all the storage (e.g. if they have 100 sector, they lie about 1 sector with 10% prob of being spotted), which with good crypto economics would be fine.
- *Case 3 (@nicola suggestion):* if we require the miner to seal all of their storage as one single sector, then they would generate one single seal proof for all the storage, which keeps the security guarantees as case 1.

### Proposal 5: Make PoSts prove the seal encoding

The strategy here is to not post seal commitments at all, but simply have miners post the commD and commR for each sector as its sealed. Miners do not receive any power updates for a sector until they submit a PoSt including that sector. Then, we change the PoSts to also prove that in addition to the data still being there over time, the data is also a correct encoding of the input data (commD maps to commR).

#### How does it work?

*Construction:* Fairly major change to PoSt, removal of seal proof.

- Prover runs seal and generates a proof (a snark proof)
- During Proof-of-Spacetime, the prover proves that: 
  - they have the data behind all of their on-chain posted commR (as we do now)
  - (new) they have know of a snark proof for each commD to commR
- Profit!

#### Pros & Cons & Unkowns

**Pros:**

- *Super Scalabe:* Seal commitment size goes down from three hashes and a SNARK to just being three (maybe two?) hashes, dramatically increasing the scalability of the protocol.
- *Only one proof:* We would go from needing the seal proofs *and* the PoSts to just needing the seal proof.

**Cons:**

- *Very complicated PoSts*: Requires bounded bootstrap (what they are doing in ZEXE) and makes the proof more expensive
- *Large PoSt circuits:* Unless we have one proof for all the storage, the circuit for proof of spacetime is linear in the number of sectors (which could get easily really big and very expensive)

**Unknown:**

- *Security Model of 'new' PoSt unknown:* It's not yet clear how the security guarantees around the seal being correct would change, and what parameters would need to be tweaked.

### Proposal 6: Seal proofs on successful block mining

The strategy is to avoid posting Seal proofs on chain during sector commitment, instead seal proofs are only posted on-chain when the miner wins the block reward.

#### How does it work?

- Miner fills a sector
  - Miner generates a seal proof
  - Miner sends the seal proof to the clients, as well as commD and commR
  - Miner post the sector commitment including commR (and probably commD)
- Miner posts PoSt proofs for sectors (since commR is on-chain, this can be done)
- When winning the block reward, the miner posts:
  - **Strategy 0:** the SEAL proof for all of their storage (if we have a way to do so, see Proposal 4)
  - **Strategy 1:** One SEAL proof chosen at random

#### Pros & Cons

**Pros**

- *Fewer proofs on chain:* No SEAL proofs on chain (but 1 seal per block)

**Cons**

- *Offchain proofs:* Miner has to send offchain SEAL proofs to their clients
- *Different security model:* Nodes can now claim to have much more storage than the storage they have. The key idea here is to separate consensus from the storage market. Consensus is handled on-chain, storage market proofs off chain:
  - **Fix 0: (radical change) Remove power table*: Run in the spacemint model: nodes are not sample on power table (in this setting the power table is broken), but based on the quality of their proof.
  - *Fix 1: (simple change) Power table is based on stake* The power table is stake-backed with storage, when elected (based on stake), then miners will need to post the SEAL proof for that particular stake.
  - Fix 2: Every X time a proof is generated, post a random SEAL, this mitigates the amount of claiming of storing fake data.
    - The security of this fix is unclear: a miner can split their identity in multiple miner identities.

#### Notes

**Security note**: If the verifier (whether is the chain or the client) doesn't have a SEAL proof, then the miner can do a generation attack and regenerate/retrieve the data they need to prove as they please. However:

- if the data come from the market, the client is sure that the miner did not run a generation attack because they are still posting PoSts on chain. since the PoSt must be put on chain
- if the miner is generating fake data, they can successfully submit PoSts, however, they will not be able to claim the block reward.
- If a miner is not storing the data, then they will be spot with a PoSt.

**Why is the power table broken?**: In this model, miners can claim more than the storage than they have, however they won't be able to win any block ( note: with Fix 2, there is a limit to the amount of spam that a node could do). Why is this a problem? This is a problem in the current Filecoin Leader Election construction, where they run ` H(ticket) < my power/total power`. If bad miners, claim to have A LOT OF POWER~!!, then the total power will be really big and no one will be able to win.

------

## Note on current plan

The current plan is to plan for Proposal 1.

We are looking into Proposal 3 for research.