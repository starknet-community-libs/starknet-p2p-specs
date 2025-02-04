# Starknet Consensus Protocol

The consensus network is built using libp2p. It uses a pubsub model with 2 topics:
1. consensus_votes
2. consensus_proposals

Validators are expected to be in the Mempool and Sync networks in addition to the consensus network.

It is an open network, meaning there is no assertion that a node is in the validator set in order to
join. In fact, validators are expected to listen on the network before the height they are scheduled
to join at is reached.

# Votes

This topic uses a broadcast model.

Votes are atomic messages, meaning each network message corresponds to a single logical consensus
vote.

In order to ensure all validators receive enough votes to progress, validators are expected to 
periodically resend their latest prevote and precommit. This is aimed to bring us closer aligned
with the PST model, which is difficult to simply assume in a real network. We do not fear this
overloading the network under the following assumptions:
  1. The number of validators sending these votes is small (<=100)
  2. The message is small (see Vote definition)
  3. The period between resends is large enough (>=1s)

# Streaming

We define here a generic streaming protocol which is used for proposals.

stream_id - field which identifies a stream of messages.
- The primary requirement is that a given sender never reuse the stream_id.
- Receivers identify streams based on (stream_id, sender_id), so distinct senders need not
  coordinate IDs.
- Applications are not expected to derive meaning from the stream ID. The primary reason for being
  generic is to put information which may be useful for humans when debugging.
- This field should be small to avoid unnecessary overhead.

message_id - field which defines the order of messages (0 based).

content - the application level information.

fin - the final message on the stream.
- If a receivers sees a message with `message_id` greater than that of the fin's `message_id` it may
  either ignore such messages or reject the stream.


# Proposals

This topic uses a broadcast model.

Starknet sends proposals as streams of network messages. This enables:
1. Proposals to be larger than a single message.
2. Higher utilization, as Validators can begin re-executing while the Proposer is still building.

Tendermint, though, views Proposals as a single logical event. To that end, the primary consensus
based messages are:
1. ProposalInit - the fields of the Tendermint proposal which can be known before the proposer
   completes block building.
2. ProposalFin - the proposer's signature on the block proposed.

## Order of messages

All proposals must begin with a ProposalInit.

The standard order for a Proposals (following the init):
1. BlockInfo - once, required to execute the transactions.
2. TransactionBatch - multiple
3. ProposalCommitment - once
4. ProposalFin

#### Empty Proposals

A proposer may not be able to offer a valid proposal. If so, the height can be agreed to be empty.
In this case the proposer will send ProposalInit followed by ProposalFin.

#### Known Proposals

In the future we may support KnownProposals, an optimization when a proposer sends out a Proposal
that has already been built. In this case it would send the ProposalFin immediately after the
ProposalInit. This would allow nodes which already know this proposal to ignore re-executing the
content.

### Proposal Commitment

In Starknet validators vote on an execution of a Proposal, not a simple identification of the values
proposed. The primary reason for this is that Starknet is optimizing for the e2e latency between:
1. End user submits a TX
2. The effect of that TX is widely visible; consensus has been reached on the StateDiff including
   it.

While not strictly needed for consensus, the ProposalCommitment field may be useful to debug nodes
which disagree a proposal's commitment `id(v)`.

#### State Root

Calculating the new state by applying the StateDiff in block H takes a long time, and should not
block consensus. Instead we allow this to occur in parallel to consensus. Once this is ready we put
the StateRoot, after applying the StateDiff from block `H`, in the commitment for `H+K`.
- K is defined in the version constants.