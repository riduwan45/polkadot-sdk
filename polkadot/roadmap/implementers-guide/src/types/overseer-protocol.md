# Overseer Protocol

This chapter contains message types sent to and from the overseer, and the underlying subsystem message types that are
transmitted using these.

## Overseer Signal

Signals from the overseer to a subsystem to request change in execution that has to be obeyed by the subsystem.

```rust
enum OverseerSignal {
  /// Signal about a change in active leaves.
  ActiveLeavesUpdate(ActiveLeavesUpdate),
  /// Signal about a new best finalized block.
  BlockFinalized(Hash),
  /// Conclude all operation.
  Conclude,
}
```

All subsystems have their own message types; all of them need to be able to listen for overseer signals as well. There
are currently two proposals for how to handle that with unified communication channels:

1. Retaining the `OverseerSignal` definition above, add `enum FromOrchestra<T> {Signal(OverseerSignal), Message(T)}`.
1. Add a generic variant to `OverseerSignal`: `Message(T)`.

Either way, there will be some top-level type encapsulating messages from the overseer to each subsystem.

## Active Leaves Update

Indicates a change in active leaves. Activated leaves should have jobs, whereas deactivated leaves should lead to
winding-down of work based on those leaves.

```rust
struct ActiveLeavesUpdate {
    activated: [(Hash, Number)],
    deactivated: [Hash],
}
```

## All Messages

A message type tying together all message types that are used across Subsystems.

```rust
enum AllMessages {
    CandidateValidation(CandidateValidationMessage),
    CandidateBacking(CandidateBackingMessage),
    ChainApi(ChainApiMessage),
    CollatorProtocol(CollatorProtocolMessage),
    StatementDistribution(StatementDistributionMessage),
    AvailabilityDistribution(AvailabilityDistributionMessage),
    AvailabilityRecovery(AvailabilityRecoveryMessage),
    BitfieldDistribution(BitfieldDistributionMessage),
    BitfieldSigning(BitfieldSigningMessage),
    Provisioner(ProvisionerMessage),
    RuntimeApi(RuntimeApiMessage),
    AvailabilityStore(AvailabilityStoreMessage),
    NetworkBridge(NetworkBridgeMessage),
    CollationGeneration(CollationGenerationMessage),
    ApprovalVoting(ApprovalVotingMessage),
    ApprovalDistribution(ApprovalDistributionMessage),
    GossipSupport(GossipSupportMessage),
    DisputeCoordinator(DisputeCoordinatorMessage),
    ChainSelection(ChainSelectionMessage),
    PvfChecker(PvfCheckerMessage),
}
```

## Approval Voting Message

Messages received by the approval voting subsystem.

```rust
enum AssignmentCheckResult {
    // The vote was accepted and should be propagated onwards.
    Accepted,
    // The vote was valid but duplicate and should not be propagated onwards.
    AcceptedDuplicate,
    // The vote was valid but too far in the future to accept right now.
    TooFarInFuture,
    // The vote was bad and should be ignored, reporting the peer who propagated it.
    Bad(AssignmentCheckError),
}

pub enum AssignmentCheckError {
    UnknownBlock(Hash),
    UnknownSessionIndex(SessionIndex),
    InvalidCandidateIndex(CandidateIndex),
    InvalidCandidate(CandidateIndex, CandidateHash),
    InvalidCert(ValidatorIndex),
    Internal(Hash, CandidateHash),
}

enum ApprovalCheckResult {
    // The vote was accepted and should be propagated onwards.
    Accepted,
    // The vote was bad and should be ignored, reporting the peer who propagated it.
    Bad(ApprovalCheckError),
}

pub enum ApprovalCheckError {
    UnknownBlock(Hash),
    UnknownSessionIndex(SessionIndex),
    InvalidCandidateIndex(CandidateIndex),
    InvalidValidatorIndex(ValidatorIndex),
    InvalidCandidate(CandidateIndex, CandidateHash),
    InvalidSignature(ValidatorIndex),
    NoAssignment(ValidatorIndex),
    Internal(Hash, CandidateHash),
}

enum ApprovalVotingMessage {
	/// Import an assignment into the approval-voting database.
	///
	/// Should not be sent unless the block hash is known and the VRF assignment checks out.
	ImportAssignment(CheckedIndirectAssignment, Option<oneshot::Sender<AssignmentCheckResult>>),
	/// Import an approval vote into approval-voting database
	///
	/// Should not be sent unless the block hash within the indirect vote is known, vote is
	/// correctly signed and we had a previous assignment for the candidate.
	ImportApproval(CheckedIndirectSignedApprovalVote, Option<oneshot::Sender<ApprovalCheckResult>>),
    /// Returns the highest possible ancestor hash of the provided block hash which is
    /// acceptable to vote on finality for. Along with that, return the lists of candidate hashes
    /// which appear in every block from the (non-inclusive) base number up to (inclusive) the specified
    /// approved ancestor.
    /// This list starts from the highest block (the approved ancestor itself) and moves backwards
    /// towards the base number.
    ///
    /// The base number is typically the number of the last finalized block, but in GRANDPA it is
    /// possible for the base to be slightly higher than the last finalized block.
    ///
    /// The `BlockNumber` provided is the number of the block's ancestor which is the
    /// earliest possible vote.
    ///
    /// It can also return the same block hash, if that is acceptable to vote upon.
    /// Return `None` if the input hash is unrecognized.
    ApprovedAncestor {
        target_hash: Hash,
        base_number: BlockNumber,
        rx: ResponseChannel<Option<(Hash, BlockNumber, Vec<(Hash, Vec<CandidateHash>)>)>>
    },
}
```

## Approval Distribution Message

Messages received by the approval distribution subsystem.

```rust
/// Metadata about a block which is now live in the approval protocol.
struct BlockApprovalMeta {
    /// The hash of the block.
    hash: Hash,
    /// The number of the block.
    number: BlockNumber,
    /// The candidates included by the block. Note that these are not the same as the candidates that appear within the
    /// block body.
    parent_hash: Hash,
    /// The candidates included by the block. Note that these are not the same as the candidates that appear within the
    /// block body.
    candidates: Vec<CandidateHash>,
    /// The consensus slot of the block.
    slot: Slot,
    /// The session of the block.
    session: SessionIndex,
}

enum ApprovalDistributionMessage {
    /// Notify the `ApprovalDistribution` subsystem about new blocks and the candidates contained within
    /// them.
    NewBlocks(Vec<BlockApprovalMeta>),
    /// Distribute an assignment cert from the local validator. The cert is assumed
    /// to be valid, relevant, and for the given relay-parent and validator index.
    ///
    /// The `u32` param is the candidate index in the fully-included list.
    DistributeAssignment(IndirectAssignmentCert, u32),
    /// Distribute an approval vote for the local validator. The approval vote is assumed to be
    /// valid, relevant, and the corresponding approval already issued. If not, the subsystem is free to drop
    /// the message.
    DistributeApproval(IndirectSignedApprovalVote),
    /// An update from the network bridge.
    NetworkBridgeUpdate(NetworkBridgeEvent<ApprovalDistributionV1Message>),
}
```

## Availability Distribution Message

Messages received by the availability distribution subsystem.

This is a network protocol that receives messages of type
[`AvailabilityDistributionV1Message`][AvailabilityDistributionV1NetworkMessage].

```rust
enum AvailabilityDistributionMessage {
      /// Incoming network request for an availability chunk.
      ChunkFetchingRequest(IncomingRequest<req_res_v1::ChunkFetchingRequest>),
      /// Incoming network request for a seconded PoV.
      PoVFetchingRequest(IncomingRequest<req_res_v1::PoVFetchingRequest>),
      /// Instruct availability distribution to fetch a remote PoV.
      ///
      /// NOTE: The result of this fetch is not yet locally validated and could be bogus.
      FetchPoV {
          /// The relay parent giving the necessary context.
          relay_parent: Hash,
          /// Validator to fetch the PoV from.
          from_validator: ValidatorIndex,
          /// Candidate hash to fetch the PoV for.
          candidate_hash: CandidateHash,
          /// Expected hash of the PoV, a PoV not matching this hash will be rejected.
          pov_hash: Hash,
          /// Sender for getting back the result of this fetch.
          ///
          /// The sender will be canceled if the fetching failed for some reason.
          tx: oneshot::Sender<PoV>,
      },
}
```

## Availability Recovery Message

Messages received by the availability recovery subsystem.

```rust
enum RecoveryError {
    Invalid,
    Unavailable,
}
enum AvailabilityRecoveryMessage {
    /// Recover available data from validators on the network.
    RecoverAvailableData(
        CandidateReceipt,
        SessionIndex,
        Option<GroupIndex>, // Backing validator group to request the data directly from.
        Option<CoreIndex>, /* A `CoreIndex` needs to be specified for the recovery process to
		                    * prefer systematic chunk recovery. This is the core that the candidate
                            * was occupying while pending availability. */
        ResponseChannel<Result<AvailableData, RecoveryError>>,
    ),
}
```

## Availability Store Message

Messages to and from the availability store.

```rust
pub enum AvailabilityStoreMessage {
	/// Query a `AvailableData` from the AV store.
	QueryAvailableData(CandidateHash, oneshot::Sender<Option<AvailableData>>),

	/// Query whether a `AvailableData` exists within the AV Store.
	///
	/// This is useful in cases when existence
	/// matters, but we don't want to necessarily pass around multiple
	/// megabytes of data to get a single bit of information.
	QueryDataAvailability(CandidateHash, oneshot::Sender<bool>),

	/// Query an `ErasureChunk` from the AV store by the candidate hash and validator index.
	QueryChunk(CandidateHash, ValidatorIndex, oneshot::Sender<Option<ErasureChunk>>),

	/// Get the size of an `ErasureChunk` from the AV store by the candidate hash.
	QueryChunkSize(CandidateHash, oneshot::Sender<Option<usize>>),

	/// Query all chunks that we have for the given candidate hash.
	QueryAllChunks(CandidateHash, oneshot::Sender<Vec<ErasureChunk>>),

	/// Query whether an `ErasureChunk` exists within the AV Store.
	///
	/// This is useful in cases like bitfield signing, when existence
	/// matters, but we don't want to necessarily pass around large
	/// quantities of data to get a single bit of information.
	QueryChunkAvailability(CandidateHash, ValidatorIndex, oneshot::Sender<bool>),

	/// Store an `ErasureChunk` in the AV store.
	///
	/// Return `Ok(())` if the store operation succeeded, `Err(())` if it failed.
	StoreChunk {
		/// A hash of the candidate this chunk belongs to.
		candidate_hash: CandidateHash,
		/// The chunk itself.
		chunk: ErasureChunk,
		/// Sending side of the channel to send result to.
		tx: oneshot::Sender<Result<(), ()>>,
	},

	/// Computes and checks the erasure root of `AvailableData` before storing all of its chunks in
	/// the AV store.
	///
	/// Return `Ok(())` if the store operation succeeded, `Err(StoreAvailableData)` if it failed.
	StoreAvailableData {
		/// A hash of the candidate this `available_data` belongs to.
		candidate_hash: CandidateHash,
		/// The number of validators in the session.
		n_validators: u32,
		/// The `AvailableData` itself.
		available_data: AvailableData,
		/// Erasure root we expect to get after chunking.
		expected_erasure_root: Hash,
		/// Sending side of the channel to send result to.
		tx: oneshot::Sender<Result<(), StoreAvailableDataError>>,
	},
}

/// The error result type of a [`AvailabilityStoreMessage::StoreAvailableData`] request.
pub enum StoreAvailableDataError {
	InvalidErasureRoot,
}
```

## Bitfield Distribution Message

Messages received by the bitfield distribution subsystem. This is a network protocol that receives messages of type
[`BitfieldDistributionV1Message`][BitfieldDistributionV1NetworkMessage].

```rust
enum BitfieldDistributionMessage {
    /// Distribute a bitfield signed by a validator to other validators.
    /// The bitfield distribution subsystem will assume this is indeed correctly signed.
    DistributeBitfield(relay_parent, SignedAvailabilityBitfield),
    /// Receive a network bridge update.
    NetworkBridgeUpdate(NetworkBridgeEvent<BitfieldDistributionV1Message>),
}
```

## Bitfield Signing Message

Currently, the bitfield signing subsystem receives no specific messages.

```rust
/// Non-instantiable message type
enum BitfieldSigningMessage { }
```

## Candidate Backing Message

```rust
enum CandidateBackingMessage {
  /// Requests a set of backable candidates attested by the subsystem.
  /// The order of candidates of the same para must be preserved in the response.
  /// If a backed candidate of a para cannot be retrieved, the response should not contain any
  /// candidates of the same para that follow it in the input vector. In other words, assuming
  /// candidates are supplied in dependency order, we must ensure that this dependency order is
  /// preserved.
  GetBackedCandidates(
    HashMap<ParaId, Vec<(CandidateHash, Hash)>>,
    oneshot::Sender<HashMap<ParaId, Vec<BackedCandidate>>>,
  ),
  /// Note that the Candidate Backing subsystem should second the given candidate in the context of the
  /// given relay-parent (ref. by hash). This candidate must be validated using the provided PoV.
  /// The PoV is expected to match the `pov_hash` in the descriptor.
  Second(Hash, CandidateReceipt, PoV),
  /// Note a peer validator's statement about a particular candidate. Disagreements about validity must be escalated
  /// to a broader check by the Disputes Subsystem, though that escalation is deferred until the approval voting
  /// stage to guarantee availability. Agreements are simply tallied until a quorum is reached.
  Statement(Statement),
}
```

## Chain API Message

The Chain API subsystem is responsible for providing an interface to chain data.

```rust
enum ChainApiMessage {
    /// Get the block number by hash.
    /// Returns `None` if a block with the given hash is not present in the db.
    BlockNumber(Hash, ResponseChannel<Result<Option<BlockNumber>, Error>>),
    /// Request the block header by hash.
    /// Returns `None` if a block with the given hash is not present in the db.
    BlockHeader(Hash, ResponseChannel<Result<Option<BlockHeader>, Error>>),
    /// Get the cumulative weight of the given block, by hash.
    /// If the block or weight is unknown, this returns `None`.
    ///
    /// Weight is used for comparing blocks in a fork-choice rule.
    BlockWeight(Hash, ResponseChannel<Result<Option<Weight>, Error>>),
    /// Get the finalized block hash by number.
    /// Returns `None` if a block with the given number is not present in the db.
    /// Note: the caller must ensure the block is finalized.
    FinalizedBlockHash(BlockNumber, ResponseChannel<Result<Option<Hash>, Error>>),
    /// Get the last finalized block number.
    /// This request always succeeds.
    FinalizedBlockNumber(ResponseChannel<Result<BlockNumber, Error>>),
    /// Request the `k` ancestors block hashes of a block with the given hash.
    /// The response channel may return a `Vec` of size up to `k`
    /// filled with ancestors hashes with the following order:
    /// `parent`, `grandparent`, ...
    Ancestors {
        /// The hash of the block in question.
        hash: Hash,
        /// The number of ancestors to request.
        k: usize,
        /// The response channel.
        response_channel: ResponseChannel<Result<Vec<Hash>, Error>>,
    }
}
```

## Chain Selection Message

Messages received by the [Chain Selection subsystem](../node/utility/chain-selection.md)

```rust
enum ChainSelectionMessage {
    /// Signal to the chain selection subsystem that a specific block has been approved.
    Approved(Hash),
    /// Request the leaves in descending order by score.
    Leaves(ResponseChannel<Vec<Hash>>),
    /// Request the best leaf containing the given block in its ancestry. Return `None` if
    /// there is no such leaf.
    BestLeafContaining(Hash, ResponseChannel<Option<Hash>>),

}
```

## Collator Protocol Message

Messages received by the [Collator Protocol subsystem](../node/collators/collator-protocol.md)

This is a network protocol that receives messages of type
[`CollatorProtocolV1Message`][CollatorProtocolV1NetworkMessage].

```rust
enum CollatorProtocolMessage {
    /// Signal to the collator protocol that it should connect to validators with the expectation
    /// of collating on the given para. This is only expected to be called once, early on, if at all,
    /// and only by the Collation Generation subsystem. As such, it will overwrite the value of
    /// the previous signal.
    ///
    /// This should be sent before any `DistributeCollation` message.
    CollateOn(ParaId),
    /// Provide a collation to distribute to validators with an optional result sender.
    ///
    /// The result sender should be informed when at least one parachain validator seconded the collation. It is also
    /// completely okay to just drop the sender.
    DistributeCollation(CandidateReceipt, PoV, Option<oneshot::Sender<CollationSecondedSignal>>),
    /// Fetch a collation under the given relay-parent for the given ParaId.
    FetchCollation(Hash, ParaId, ResponseChannel<(CandidateReceipt, PoV)>),
    /// Report a collator as having provided an invalid collation. This should lead to disconnect
    /// and blacklist of the collator.
    ReportCollator(CollatorId),
    /// Note a collator as having provided a good collation.
    NoteGoodCollation(CollatorId, SignedFullStatement),
    /// Notify a collator that its collation was seconded.
    NotifyCollationSeconded(CollatorId, Hash, SignedFullStatement),
}
```

## Collation Generation Message

Messages received by the [Collation Generation subsystem](../node/collators/collation-generation.md)

This is the core interface by which collators built on top of a Polkadot node submit collations to validators. As such,
these messages are not sent by any subsystem but are instead sent from outside of the overseer.

```rust
/// A function provided to the subsystem which it uses to pull new collations.
///
/// This mode of querying collations is obsoleted by `CollationGenerationMessages::SubmitCollation`
///
/// The response channel, if present, is meant to receive a `Seconded` statement as a
/// form of authentication, for collation mechanisms which rely on this for anti-spam.
type CollatorFn = Fn(Hash, PersistedValidationData) -> Future<Output = (Collation, Option<ResponseChannel<SignedStatement>>)>;

/// Configuration for the collation generator
struct CollationGenerationConfig {
    /// Collator's authentication key, so it can sign things.
    key: CollatorPair,
    /// Collation function. See [`CollatorFn`] for more details.
    collator: CollatorFn,
    /// The parachain that this collator collates for
    para_id: ParaId,
}

/// Parameters for submitting a collation
struct SubmitCollationParams {
    /// The relay-parent the collation is built against.
    relay_parent: Hash,
    /// The collation itself (PoV and commitments)
    collation: Collation,
    /// The parent block's head-data.
    parent_head: HeadData,
    /// The hash of the validation code the collation was created against.
    validation_code_hash: ValidationCodeHash,
    /// A response channel for receiving a `Seconded` message about the candidate
    /// once produced by a validator. This is not guaranteed to provide anything.
    result_sender: Option<ResponseChannel<SignedStatement>>,
}

enum CollationGenerationMessage {
    /// Initialize the collation generation subsystem
	Initialize(CollationGenerationConfig),
    /// Submit a collation to the subsystem. This will package it into a signed
    /// [`CommittedCandidateReceipt`] and distribute along the network to validators.
    ///
    /// If sent before `Initialize`, this will be ignored.
    SubmitCollation(SubmitCollationParams),
}
```

## Dispute Coordinator Message

Messages received by the [Dispute Coordinator subsystem](../node/disputes/dispute-coordinator.md)

This subsystem coordinates participation in disputes, tracks live disputes, and observed statements of validators from
subsystems.

```rust
enum DisputeCoordinatorMessage {
    /// Import a statement by a validator about a candidate.
    ///
    /// The subsystem will silently discard ancient statements or sets of only dispute-specific statements for
    /// candidates that are previously unknown to the subsystem. The former is simply because ancient
    /// data is not relevant and the latter is as a DoS prevention mechanism. Both backing and approval
    /// statements already undergo anti-DoS procedures in their respective subsystems, but statements
    /// cast specifically for disputes are not necessarily relevant to any candidate the system is
    /// already aware of and thus present a DoS vector. Our expectation is that nodes will notify each
    /// other of disputes over the network by providing (at least) 2 conflicting statements, of which one is either
    /// a backing or validation statement.
    ///
    /// This does not do any checking of the message signature.
    ImportStatements {
        /// The hash of the candidate.
        candidate_hash: CandidateHash,
        /// The candidate receipt itself.
        candidate_receipt: CandidateReceipt,
        /// The session the candidate appears in.
        session: SessionIndex,
        /// Triples containing the following:
        /// - A statement, either indicating validity or invalidity of the candidate.
        /// - The validator index (within the session of the candidate) of the validator casting the vote.
        /// - The signature of the validator casting the vote.
        statements: Vec<(DisputeStatement, ValidatorIndex, ValidatorSignature)>,

        /// Inform the requester once we finished importing.
        ///
        /// This is, we either discarded the votes, just record them because we
        /// casted our vote already or recovered availability for the candidate
        /// successfully.
        pending_confirmation: oneshot::Sender<ImportStatementsResult>
    },
    /// Fetch a list of all recent disputes that the co-ordinator is aware of.
    /// These are disputes which have occurred any time in recent sessions, which may have already concluded.
    RecentDisputes(ResponseChannel<Vec<(SessionIndex, CandidateHash)>>),
    /// Fetch a list of all active disputes that the co-ordinator is aware of.
    /// These disputes are either unconcluded or recently concluded.
    ActiveDisputes(ResponseChannel<Vec<(SessionIndex, CandidateHash)>>),
    /// Get candidate votes for a candidate.
    QueryCandidateVotes(SessionIndex, CandidateHash, ResponseChannel<Option<CandidateVotes>>),
    /// Sign and issue local dispute votes. A value of `true` indicates validity, and `false` invalidity.
    IssueLocalStatement(SessionIndex, CandidateHash, CandidateReceipt, bool),
    /// Determine the highest undisputed block within the given chain, based on where candidates
    /// were included. If even the base block should not be finalized due to a dispute,
    /// then `None` should be returned on the channel.
    ///
    /// The block descriptions begin counting upwards from the block after the given `base_number`. The `base_number`
    /// is typically the number of the last finalized block but may be slightly higher. This block
    /// is inevitably going to be finalized so it is not accounted for by this function.
    DetermineUndisputedChain {
        base_number: BlockNumber,
        block_descriptions: Vec<(BlockHash, SessionIndex, Vec<CandidateHash>)>,
        rx: ResponseSender<Option<(BlockNumber, BlockHash)>>,
    }
}

/// Result of `ImportStatements`.
pub enum ImportStatementsResult {
	/// Import was invalid (candidate was not available)  and the sending peer should get banned.
	InvalidImport,
	/// Import was valid and can be confirmed to peer.
	ValidImport
}
```

## Dispute Distribution Message

Messages received by the [Dispute Distribution subsystem](../node/disputes/dispute-distribution.md). This subsystem is
responsible of distributing explicit dispute statements.

```rust
enum DisputeDistributionMessage {

  /// Tell dispute distribution to distribute an explicit dispute statement to
  /// validators.
  SendDispute((ValidVote, InvalidVote)),

  /// Ask DisputeDistribution to get votes we don't know about.
  /// Fetched votes will be reported via `DisputeCoordinatorMessage::ImportStatements`
  FetchMissingVotes {
    candidate_hash: CandidateHash,
    session: SessionIndex,
    known_valid_votes: Bitfield,
    known_invalid_votes: Bitfield,
    /// Optional validator to query from. `ValidatorIndex` as in the above
    /// referenced session.
    from_validator: Option<ValidatorIndex>,
  }
}
```

## Network Bridge Message

Messages received by the network bridge. This subsystem is invoked by others to manipulate access to the low-level
networking code.

```rust
/// Peer-sets handled by the network bridge.
enum PeerSet {
    /// The collation peer-set is used to distribute collations from collators to validators.
    Collation,
    /// The validation peer-set is used to distribute information relevant to parachain
    /// validation among validators. This may include nodes which are not validators,
    /// as some protocols on this peer-set are expected to be gossip.
    Validation,
}

enum NetworkBridgeMessage {
    /// Report a cost or benefit of a peer. Negative values are costs, positive are benefits.
    ReportPeer(PeerId, cost_benefit: i32),
    /// Disconnect a peer from the given peer-set without affecting their reputation.
    DisconnectPeer(PeerId, PeerSet),
    /// Send a message to one or more peers on the validation peerset.
    SendValidationMessage([PeerId], ValidationProtocolV1),
    /// Send a message to one or more peers on the collation peerset.
    SendCollationMessage([PeerId], ValidationProtocolV1),
    /// Send multiple validation messages.
    SendValidationMessages([([PeerId, ValidationProtocolV1])]),
    /// Send multiple collation messages.
    SendCollationMessages([([PeerId, ValidationProtocolV1])]),
    /// Connect to peers who represent the given `validator_ids`.
    ///
    /// Also ask the network to stay connected to these peers at least
    /// until a new request is issued.
    ///
    /// Because it overrides the previous request, it must be ensured
    /// that `validator_ids` include all peers the subsystems
    /// are interested in (per `PeerSet`).
    ///
    /// A caller can learn about validator connections by listening to the
    /// `PeerConnected` events from the network bridge.
    ConnectToValidators {
        /// Ids of the validators to connect to.
        validator_ids: HashSet<AuthorityDiscoveryId>,
        /// The underlying protocol to use for this request.
        peer_set: PeerSet,
        /// Sends back the number of `AuthorityDiscoveryId`s which
        /// authority discovery has failed to resolve.
        failed: oneshot::Sender<usize>,
    },
    /// Inform the distribution subsystems about the new
    /// gossip network topology formed.
    NewGossipTopology {
		/// The session info this gossip topology is concerned with.
		session: SessionIndex,
		/// Our validator index in the session, if any.
		local_index: Option<ValidatorIndex>,
		/// The canonical shuffling of validators for the session.
		canonical_shuffling: Vec<(AuthorityDiscoveryId, ValidatorIndex)>,
		/// The reverse mapping of `canonical_shuffling`: from validator index
		/// to the index in `canonical_shuffling`
		shuffled_indices: Vec<usize>,
    }
}
```

## Misbehavior Report

```rust
pub type Misbehavior = generic::Misbehavior<
    CommittedCandidateReceipt,
    CandidateHash,
    ValidatorIndex,
    ValidatorSignature,
>;

mod generic {
    /// Misbehavior: voting more than one way on candidate validity.
    ///
    /// Since there are three possible ways to vote, a double vote is possible in
    /// three possible combinations (unordered)
    pub enum ValidityDoubleVote<Candidate, Digest, Signature> {
        /// Implicit vote by issuing and explicitly voting validity.
        IssuedAndValidity((Candidate, Signature), (Digest, Signature)),
        /// Implicit vote by issuing and explicitly voting invalidity
        IssuedAndInvalidity((Candidate, Signature), (Digest, Signature)),
        /// Direct votes for validity and invalidity
        ValidityAndInvalidity(Candidate, Signature, Signature),
    }

    /// Misbehavior: multiple signatures on same statement.
    pub enum DoubleSign<Candidate, Digest, Signature> {
        /// On candidate.
        Candidate(Candidate, Signature, Signature),
        /// On validity.
        Validity(Digest, Signature, Signature),
        /// On invalidity.
        Invalidity(Digest, Signature, Signature),
    }

    /// Misbehavior: submitted statement for wrong group.
    pub struct UnauthorizedStatement<Candidate, Digest, AuthorityId, Signature> {
        /// A signed statement which was submitted without proper authority.
        pub statement: SignedStatement<Candidate, Digest, AuthorityId, Signature>,
    }

    pub enum Misbehavior<Candidate, Digest, AuthorityId, Signature> {
        /// Voted invalid and valid on validity.
        ValidityDoubleVote(ValidityDoubleVote<Candidate, Digest, Signature>),
        /// Submitted a message that was unauthorized.
        UnauthorizedStatement(UnauthorizedStatement<Candidate, Digest, AuthorityId, Signature>),
        /// Submitted two valid signatures for the same message.
        DoubleSign(DoubleSign<Candidate, Digest, Signature>),
    }
}
```

## PoV Distribution Message

This is a network protocol that receives messages of type [`PoVDistributionV1Message`][PoVDistributionV1NetworkMessage].

```rust
enum PoVDistributionMessage {
    /// Fetch a PoV from the network.
    ///
    /// This `CandidateDescriptor` should correspond to a candidate seconded under the provided
    /// relay-parent hash.
    FetchPoV(Hash, CandidateDescriptor, ResponseChannel<PoV>),
    /// Distribute a PoV for the given relay-parent and CandidateDescriptor.
    /// The PoV should correctly hash to the PoV hash mentioned in the CandidateDescriptor
    DistributePoV(Hash, CandidateDescriptor, PoV),
    /// An update from the network bridge.
    NetworkBridgeUpdate(NetworkBridgeEvent<PoVDistributionV1Message>),
}
```

## Provisioner Message

```rust
/// This data becomes intrinsics or extrinsics which should be included in a future relay chain block.
enum ProvisionableData {
  /// This bitfield indicates the availability of various candidate blocks.
  Bitfield(Hash, SignedAvailabilityBitfield),
  /// The Candidate Backing subsystem believes that this candidate is valid, pending availability.
  BackedCandidate(CandidateReceipt),
  /// Misbehavior reports are self-contained proofs of validator misbehavior.
  MisbehaviorReport(Hash, MisbehaviorReport),
  /// Disputes trigger a broad dispute resolution process.
  Dispute(Hash, Signature),
}

/// Message to the Provisioner.
///
/// In all cases, the Hash is that of the relay parent.
enum ProvisionerMessage {
  /// This message allows external subsystems to request current inherent data that could be used for
  /// advancing the state of parachain consensus in a block building upon the given hash.
  ///
  /// If called at different points in time, this may give different results.
  RequestInherentData(Hash, oneshot::Sender<ParaInherentData>),
  /// This data should become part of a relay chain block
  ProvisionableData(ProvisionableData),
}
```

## Runtime API Message

The Runtime API subsystem is responsible for providing an interface to the state of the chain's runtime.

This is fueled by an auxiliary type encapsulating all request types defined in the [Runtime API section](../runtime-api)
of the guide.

```rust
enum RuntimeApiRequest {
    /// Get the version of the runtime API at the given parent hash, if any.
    Version(ResponseChannel<u32>),
    /// Get the current validator set.
    Validators(ResponseChannel<Vec<ValidatorId>>),
    /// Get the validator groups and rotation info.
    ValidatorGroups(ResponseChannel<(Vec<Vec<ValidatorIndex>>, GroupRotationInfo)>),
    /// Get information about all availability cores.
    AvailabilityCores(ResponseChannel<Vec<CoreState>>),
    /// with the given occupied core assumption.
    PersistedValidationData(
        ParaId,
        OccupiedCoreAssumption,
        ResponseChannel<Option<PersistedValidationData>>,
    ),
    /// Sends back `true` if the commitments pass all acceptance criteria checks.
    CheckValidationOutputs(
        ParaId,
        CandidateCommitments,
        RuntimeApiSender<bool>,
    ),
    /// Get the session index for children of the block. This can be used to construct a signing
    /// context.
    SessionIndexForChild(ResponseChannel<SessionIndex>),
    /// Get the validation code for a specific para, using the given occupied core assumption.
    ValidationCode(ParaId, OccupiedCoreAssumption, ResponseChannel<Option<ValidationCode>>),
    /// Get validation code by its hash, either past, current or future code can be returned,
    /// as long as state is still available.
    ValidationCodeByHash(ValidationCodeHash, RuntimeApiSender<Option<ValidationCode>>),
    /// Get a committed candidate receipt for all candidates pending availability.
    CandidatePendingAvailability(ParaId, ResponseChannel<Option<CommittedCandidateReceipt>>),
    /// Get all events concerning candidates in the last block.
    CandidateEvents(ResponseChannel<Vec<CandidateEvent>>),
    /// Get the session info for the given session, if stored.
    SessionInfo(SessionIndex, ResponseChannel<Option<SessionInfo>>),
    /// Get all the pending inbound messages in the downward message queue for a para.
    DmqContents(ParaId, ResponseChannel<Vec<InboundDownwardMessage<BlockNumber>>>),
    /// Get the contents of all channels addressed to the given recipient. Channels that have no
    /// messages in them are also included.
    InboundHrmpChannelsContents(ParaId, ResponseChannel<BTreeMap<ParaId, Vec<InboundHrmpMessage<BlockNumber>>>>),
    /// Get information about the BABE epoch this block was produced in.
    BabeEpoch(ResponseChannel<BabeEpoch>),
}

enum RuntimeApiMessage {
    /// Make a request of the runtime API against the post-state of the given relay-parent.
    Request(Hash, RuntimeApiRequest),
    /// Get the version of the runtime API at the given parent hash, if any.
    Version(Hash, ResponseChannel<Option<u32>>)
}
```

## Statement Distribution Message

The Statement Distribution subsystem distributes signed statements and candidates from validators to other validators.
It does this by distributing full statements, which embed the candidate receipt, as opposed to compact statements which
don't. It receives updates from the network bridge and signed statements to share with other validators.

This is a network protocol that receives messages of type
[`StatementDistributionV1Message`][StatementDistributionV1NetworkMessage].

```rust
enum StatementDistributionMessage {
    /// An update from the network bridge.
    NetworkBridgeUpdate(NetworkBridgeEvent<StatementDistributionV1Message>),
    /// We have validated a candidate and want to share our judgment with our peers.
    /// The hash is the relay parent.
    ///
    /// The statement distribution subsystem assumes that the statement should be correctly
    /// signed.
    Share(Hash, SignedFullStatementWithPVD),
}
```

## Validation Request Type

Various modules request that the [Candidate Validation subsystem](../node/utility/candidate-validation.md) validate a
block with this message. It returns [`ValidationOutputs`](candidate.md#validationoutputs) for successful validation.

```rust

/// The outcome of the candidate-validation's PVF pre-check request.
pub enum PreCheckOutcome {
    /// The PVF has been compiled successfully within the given constraints.
    Valid,
    /// The PVF could not be compiled. This variant is used when the candidate-validation subsystem
    /// can be sure that the PVF is invalid. To give a couple of examples: a PVF that cannot be
    /// decompressed or that does not represent a structurally valid WebAssembly file.
    Invalid,
    /// This variant is used when the PVF cannot be compiled but for other reasons that are not
    /// included into [`PreCheckOutcome::Invalid`]. This variant can indicate that the PVF in
    /// question is invalid, however it is not necessary that PVF that received this judgement
    /// is invalid.
    ///
    /// For example, if during compilation the preparation worker was killed we cannot be sure why
    /// it happened: because the PVF was malicious made the worker to use too much memory or its
    /// because the host machine is under severe memory pressure and it decided to kill the worker.
    Failed,
}

/// Result of the validation of the candidate.
enum ValidationResult {
    /// Candidate is valid, and here are the outputs and the validation data used to form inputs.
    /// In practice, this should be a shared type so that validation caching can be done.
    Valid(CandidateCommitments, PersistedValidationData),
    /// Candidate is invalid.
    Invalid,
}

const BACKING_EXECUTION_TIMEOUT: Duration = 2 seconds;
const APPROVAL_EXECUTION_TIMEOUT: Duration = 6 seconds;

/// Messages received by the Validation subsystem.
///
/// ## Validation Requests
///
/// Validation requests made to the subsystem should return an error only on internal error.
/// Otherwise, they should return either `Ok(ValidationResult::Valid(_))`
/// or `Ok(ValidationResult::Invalid)`.
#[derive(Debug)]
pub enum CandidateValidationMessage {
    /// Validate a candidate with provided, exhaustive parameters for validation.
    ///
    /// Explicitly provide the `PersistedValidationData` and `ValidationCode` so this can do full
    /// validation without needing to access the state of the relay-chain.
    ///
    /// This request doesn't involve acceptance criteria checking, therefore only useful for the
    /// cases where the validity of the candidate is established. This is the case for the typical
    /// use-case: approval checkers would use this request relying on the full prior checks
    /// performed by the relay-chain.
    ValidateFromExhaustive(
        PersistedValidationData,
        ValidationCode,
        CandidateDescriptor,
        Arc<PoV>,
        Duration, // Execution timeout.
        oneshot::Sender<Result<ValidationResult, ValidationFailed>>,
    ),
    /// Try to compile the given validation code and send back
    /// the outcome.
    ///
    /// The validation code is specified by the hash and will be queried from the runtime API at the
    /// given relay-parent.
    PreCheck(
        // Relay-parent
        Hash,
        ValidationCodeHash,
        oneshot::Sender<PreCheckOutcome>,
    ),
}
```

## PVF Pre-checker Message

Currently, the PVF pre-checker subsystem receives no specific messages.

```rust
/// Non-instantiable message type
pub enum PvfCheckerMessage { }
```

[NBE]: ../network.md#network-bridge-event
[AvailabilityDistributionV1NetworkMessage]: network.md#availability-distribution-v1
[BitfieldDistributionV1NetworkMessage]: network.md#bitfield-distribution-v1
[PoVDistributionV1NetworkMessage]: network.md#pov-distribution-v1
[StatementDistributionV1NetworkMessage]: network.md#statement-distribution-v1
[CollatorProtocolV1NetworkMessage]: network.md#collator-protocol-v1
