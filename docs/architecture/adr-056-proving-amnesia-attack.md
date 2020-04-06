# ADR 056: Proving flip flopping attacks

## Changelog

- 02.04.20: Initial Draft
- 06.04.20: Second Draft

## Context

Whilst most created evidence of malicious behaviour is self evident such that any individual can verify them independently there are types of evidence, known as global evidence, that require further collaboration from the network in order to accumulate enough information to create evidence that is individually verifiable and can therefore be processed through consensus. Fork Accountability as a whole constitutes of detection, proving and punishing. This ADR addresses how to prove infringement from global evidence that is sent to a full node.

The currently only known form of global evidence stems from [flip flopping](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md#flip-flopping) attacks. The schematic below explains one scenario where an amnesia attack, a form of flip flopping, can occur such that C1 and C2 commit different blocks.

![](../imgs/tm-amnesia-attack.png)

This creates a fork on the main chain.  Back to the past, another form of flip flopping, creates a light fork (capable of fooling those not involved in consensus), in a similar way, with F taking the precommits from C1 and forging a commit from them.

## Decision

As the distinction between these two attacks (amnesia and back to the past) can only be distinguished by confirming with all validators (to see if it is a full fork or a light fork), for the purpose of simplicity, these attacks will be treated as the same.

Currently, the evidence reactor is used to simply broadcast and store evidence. Instead of perhaps creating a new reactor for the specific task of verifying these attacks, the current evidence reactor will be extended.

The process begins with a light client receiving conflicting headers (in the future this could also be a full node during fast sync), which it sends to a full node to analyse. As part of [evidence handling](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-047-handling-evidence-from-light-client.md), this could be deduced into potential amnesia evidence

```
type PotentialAmnesiaEvidence struct {
	V1 []*types.Vote
	V2 []*types.Vote

	timestamp time.Time
}
```

*REMARK: We could also use `VoteSet`*

*NOTE: this is almost identical to Duplicate Vote Evidence and although one is for different rounds and the other is for the same round, there is a case that these two could be combined. In addition, I believe batch processing to be more efficient than individual processing and would think all evidence that constitutes multiple attackers should be grouped together.*

The evidence should contain the precommit votes for the intersection of validators that voted for both rounds. The votes should be all valid and the height and time that the infringement was made should be within:

`Bonding period - Amnesia trial period - Gossip safety margin`

With reference to the honest nodes, C1 and C2, in the schematic, C2 will not PRECOMMIT an earlier round, but it is likely, if a node in C1 were to recieve +2/3 PREVOTE's or PRECOMMIT's for a higher round, that it would remove the lock and PREVOTE and PRECOMMIT for the later round. Therefore, unfortunately it is not a case of simply punishing all nodes that have double voted in the `PotentialAmnesiaEvidence`.

Instead we use the Proof of Lock Change (PoLC) referred to in the [consensus spec](https://github.com/tendermint/spec/blob/master/spec/consensus/consensus.md#terms). When an honest node votes again for a later round, in very rare cases, it will generate the PoLC and store it in the evidence reactor for a time equal to the `Bonding Period`

```
type ProofOfLockChange struct {
	Votes []*types.Vote
}
```

This can be either evidence of +2/3 PREVOTES or PRECOMMITS (either warrants the honest node the right to vote) and is valid, among other checks, so long as the PRECOMMIT vote of the node in V2 came after all the votes in the `ProofOfLockChange` i.e. it received +2/3  votes and then voted itself (F is unable to prove this).

In the event that an honest node receives `PotentialAmnesiaEvidence` it will first `Verify()` it and then will check if it is among the suspected nodes in the evidence. If so, it will retrieve the `ProofOfLockChange` and combine it with `PotentialAmensiaEvidence` to form `AmensiaEvidence`:

```
type AmnesiaEvidence struct {
	Evidence *types.PotentialAmnesiaEvidence
	Proofs	 []*types.ProofOfLockChange
}
```

If the node is not required to submit any proof than it will simply broadcast the `PotentialAmnesiaEvidence` .

In either case, in a network where each node has n peers, if n is high enough to easily overcome network problems such as dropout than it should be sufficient that peers only broadcast once per `Amnesia trial period` + `Gossip safety margin` as long as `AmnesiaEvidence` has not yet been committed on the chain.

If a node does not have sufficient peers then it may proportionally send more messages at an appropriate interval.

When a node has successfully validated `PotentialAmnesiaEvidence` it timestamps it and refuses to receive the same form of `PotentialAmnesiaEvidence`. If a node receives `AmnesiaEvidence` it checks it against any current `AmnesiaEvidence` it might have and if so merges the two by adding the proofs, if it doesn't have it yet it run's `Verify()` and stores it.

There can only be one `AmnesiaEvidence` and one `PotentialAmneisaEvidence` stored for each attack.

When, `time.Now() > PotentialAmnesiaEvidence.timestamp + AmnesiaTrialPeriod`, honest validators of the current validator set can begin proposing the block that contains the `AmnesiaEvidence`.

Other validators will vote <nil> if:

- The Amnesia Evidence is not valid
- The Amensia Evidence is not within the validators trial period - the gossip safety margin i.e. too soon.
- The Amensia Evidence is of the same height but is different to the Amnesia Evidence that they have. i.e. is missing proofs.
- Is of an AmnesiaEvidence that has already been committed to the chain.

*NOTE: Although this process only indicates committing `AmensiaEvidence` to the chain, it is also possible that `PotentialAmnesiaEvidence` is committed*

## Status

Proposed

## Consequences

### Positive

Increasing fork detection makes the system more secure

### Negative

Non-responsive but honest nodes that are part of the suspect group that don't produce a proof will be punished

A delay between the detection of a fork and the punishment of one

### Neutral

## References

- [Fork accountability algorithm](https://docs.google.com/document/d/11ZhMsCj3y7zIZz4udO9l25xqb0kl7gmWqNpGVRzOeyY/edit)
- [Fork accountability spec](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md)
