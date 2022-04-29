# 📡 State Connector

## Introduction

Piece of the Flare network that keeps track of the state of other networks, facilitating the implementation of advanced mechanisms like the FAssets. The State Connector uses several independent Attestation Providers that are rewarded for providing correct information.

1. Anybody can request an attestation of a specific external event from the State Connector contract running on the Flare Network.

   - Requests are yes/no questions regarding things that happened outside the Flare Network, for example, "Has transaction 0xABC been confirmed on the Bitcoin network enough times?"

   - Available request types are strictly decidable by design (answers cannot be argued).

   - Requesters use the requestAttestations method.

   - Compare with other bridges where the complete state of the connected chain is available.

2. The State Connector signals all Attestation Providers about this request.

   - This is done through very gas-efficient EVM events.

3. Interested Attestation Providers fetch the requested data by means that depend on the type of attestation.

   - E.g., retrieving data from another blockchain or, in the future, reading a physical sensor.

4. Attestation providers submit their results to the State Connector in a [Commit and Reveal](#rcr-protocol) fashion so they cannot peek at each other's submissions until the round is over.

   - Results are submitted in a closed envelope. When the round is over all envelopes are opened.

   - For performance reasons, all requests collected during a 90s round are answered at once, using a hash (Merkle tree root) that summarizes all of them (See [Attestation Provider](#attestation-providers-architecture)).

   - Providers must answer all queries in the round or none at all, otherwise, the Merkle tree root will not match other providers.

   - Providers use the submitAttestation method in the State Connector.

5. If more than 50% of the providers agree, Merkle roots (called Attestation Proofs) are made public.

   - Otherwise (no consensus achieved) requests remain unanswered and must be issued again.

   - Proofs are stored for a week. Anybody can recover the Merkle root of a previous round and extract an answer for their request (See [Attestation Proof Unpacking](#proof-unpacking) for details).

6. Rewards.

## Architecture

<figure markdown>
  ![State Connector architecture](SC-architecture.png){ loading=lazy }
  <figcaption>The State Connector architecture.</figcaption>
</figure>

## RCR Protocol

<figure markdown>
  ![Request-Commit-Reveal protocol](SC-RCR.png){ loading=lazy }
  <figcaption>The Request-Commit-Reveal protocol.</figcaption>
</figure>

<figure markdown>
  ![Overlapped Request-Commit-Reveal protocol](SC-RCR-overlapped.png){ loading=lazy }
  <figcaption>The Request-Commit-Reveal protocol with overlapped phases.</figcaption>
</figure>

## Branching Protocol

<figure markdown>
  ![Branching](SC-branching.png){ loading=lazy }
  <figcaption>Branching.</figcaption>
</figure>

<figure markdown>
  ![Branching resolution 1](SC-branching-resolution-1.png){ loading=lazy }
  <figcaption>Branching resolution 1.</figcaption>
</figure>

<figure markdown>
  ![Branching resolution 2](SC-branching-resolution-2.png){ loading=lazy }
  <figcaption>Branching resolution 2.</figcaption>
</figure>

## Attestation Providers Architecture

<figure markdown>
  ![Attestation Provider architecture](SC-attestation-provider.png){ loading=lazy }
  <figcaption>The Attestation Provider architecture.</figcaption>
</figure>

## Default Attestation Providers

## Local Attestation Providers

<figure markdown>
  ![Local attestation providers](SC-local-AP.png){ loading=lazy }
  <figcaption>Local attestation providers.</figcaption>
</figure>

## Proof Unpacking

<figure markdown>
  ![Proof unpacking](SC-proof-unpacking.png){ loading=lazy }
  <figcaption>Proof unpacking.</figcaption>
</figure>

??? "Old page"

    The State Connector on Flare enables proving real-world events to any contract on Flare. Uniquely, the State Connector can also handle any notional value represented by the events it proves. This property of the system is achieved by automated branching of the Flare Network state to correct event outcomes, without degrading the time-to-finality or throughput metrics of the network.

    ## Voting Protocol

    There are three phases of the State Connector voting protocol, delineated based on the current timestamp on Flare, and overlapping such that multiple sets of requests for event proofs can be worked on in parallel.

    <figure markdown>
      ![The three phases of the State Connector voting protocol](../assets/SCVotingProtocol.svg){ loading=lazy }
      <figcaption>The three phases of the State Connector voting protocol.</figcaption>
    </figure>

    #### Request Phase

    At any point in time, any user can submit a request to the State Connector contract to have an event proven. The window in time that this request enters the network state is known as the _request_ phase from its perspective.

    #### Commit Phase

    During the next window of time, _attestation providers_ have the opportunity to _commit_ a hidden vote regarding their belief in the outcome of the events requested in the previous phase. Anyone may operate as an attestation provider without any capital requirement, but a default incentivized set is used as the minimal requirement for passing a vote about the events in the previous set.

    #### Reveal Phase

    Finally in the next window of time, attestation providers _reveal_ their votes that they committed to in the previous round. Once this reveal phase concludes and the next phase begins, the revealed votes are automatically counted and all valid events become immediately available to all contracts on Flare.

    ## Branching Protocol

    The State Connector branching protocol protects Flare against incorrect interpretation of real-world events, _proactively_, such that there are never any rollbacks on the Flare blockchain state. Instead of having rollbacks, contention on state correctness is handled via automatic state branching into a correct and incorrect path. The security assumption is that if you as an independent node operator are following along with the correct real-world state, then you will always end up on the correct branch of the blockchain state.

    <figure markdown>
      ![The State Connector branching protocol.](../assets/SCBranchingProtocol.svg){ loading=lazy }
      <figcaption>The State Connector branching protocol.</figcaption>
    </figure>

    #### Default Attestation Providers

    The minimum requirement to confirm the existence and validity of a blockchain transaction is for it to be confirmed by a majority of the default set of Attestation Providers.

    #### Local Attestation Providers

    Anyone may also operate their own _local_ attestation provider(s) without any capital requirement. Every Flare node operator, no matter how prominently they feature in the overall network, defines which local attestation provider(s) they wish to use for the State Connector branching protocol.

    A Flare node will only pass a State Connector vote if both the default set and their locally-defined set of attestation providers pass the vote:

    * If a Flare node's locally-defined set of attestation providers disagrees with the vote made by the default set, then:
      1. The Flare node will automatically create a backup of the blockchain state at the last point that it will have in common with the default set.
      2. The Flare node will then proceed along the branch that it locally believes is correct.
    * Else if the default set fails to pass a vote, then:
      1. No branching occurs.

    ## Scalability

    Below are examples of design considerations in the State Connector that make it highly scalable.

    #### Overlapped Voting Protocol

    <figure markdown>
      ![Overlapped voting protocol.](../assets/SCOverlappedVoting.svg){ loading=lazy }
      <figcaption>Overlapped voting protocol.</figcaption>
    </figure>

    Every window of time during the State Connector voting protocol is an opportunity to _request_ event proofs, meaning that while a new event is being requested, prior events can be voted on in both the commit and reveal phase. This multiplies the throughput of the state connector by a factor of three.

    #### No Storage of Requests

    When requests for new events are submitted to the State Connector contract, storage is not invoked. Instead, a Solidity event is emitted. This enables the total cost of the event request transaction to be below 2k gas, i.e. less than 0.1x the cost of a simple payment.

    #### Merkle-Tree Root Voting

    The gas usage of attestation providers is always constant, despite the number of event proof requests they handle, because they construct the valid events into a merkle tree and simply vote on the merkle tree root hash. The merkle tree algorithm can also be swapped out over time to more efficient algorithms without impacting the core State Connector voting protocol which always just votes on the root hash.

    ## New Event-Type Integrations

    New real-world event-type integrations are introduced to the State Connector via acceptance by the default attestation providers, and without requiring any changes to the core voting or branching protocols described above. This enables rapid deployment of new use-cases without any validator-level code changes.
