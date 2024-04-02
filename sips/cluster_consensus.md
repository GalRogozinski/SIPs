|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Matheus Franco, Gal Rogozinski | Cluster consensus          | Core       | open-for-discussion | 2024-03-05 |

## Summary

Aggregate `Attestation` and `Sync Committee` duties based on the cluster of operators and the duties' slot. 

## Motivation

With the current design, a cluster of operators associated with several validators may end up performing more than one attestation or sync committee duties on equivalent data. This proposal helps to decrease the number of messages exchanged in the network and the processing cost.

## Improvement

According to Monte Carlo simulations using a dataset based on the Mainnet, this proposal reduces to $21.60$% the current number of messages exchanged in the network. Note that this result includes aggregating the post-consensus messages into a single message.

Regarding the number of bits exchanged, we estimate that this proposal will reduce the current value to, at least, $52.96$%. Notice that this reduction is not as significant as the number of messages reduction due to the larger post-consensus messages.

Again with Monte Carlo simulations using the Mainnet dataset, the number of attestation duties aggregated presented the following distribution.

<p align="center">
<img src="./images/cluster_consensus/aggregated_duties.png"  width="50%" height="10%">
</p>


## Rationale

The aggregation of duties is possible because the data that must be agreed on is independent of the validator.

For example, take a look at the `AttestationData` type.
```go
type AttestationData struct {
	Slot            Slot
	Index           CommitteeIndex
	BeaconBlockRoot Root `ssz-size:"32"`
	Source          *Checkpoint
	Target          *Checkpoint
}
```
The only validator-dependent field is `CommitteeIndex` and it does not have to be agreed on.

For the `Sync Committee` duty, operators agree on a `phase0.Root` data which is also independent of the validator.


## Spec changes

### Design

Under the new design we will have a `Cluster` object that will be the top level object in charge of processing consensus messages and partial signature messages for the attestation and sync committee roles.

The `Cluster` will hold a single `ConsensusRunner` and multiple `PartialSigRunners` for each Validator managed by the cluster.

For other duty roles the old design will remain.

#### Code

```go
type Cluster interface {
	// startDuties starts the duties for the given slot
	// Starts the CosnensusRunner and registers the proper SSVRunners
	startDuties(duties []types.Duty, slot spec.Slot) error
}

// Cluster is a cluster of a unique set of operators that run the same validator set
type Cluster struct {
	ConsensusRunner ConsensusRunner
	SSVRunners      SSVRunners
	Network         Network
	Beacon          BeaconNode
}

// ConsensusRunner is in charge of processing consensus messages and managing the consensus instance
type ConsensusRunner interface {
	GetBeaconNode() BeaconNode
	GetValCheckF() qbft.ProposedValueCheckF
	GetNetwork() Network

	// StartNewConsensus starts a new consensus instance for the given roles and slot
	StartNewConsensus(roles []types.BeaconRole, slot spec.Slot) error
	// ProcessConsensus processes a consensus message
	ProcessConsensus(msg *qbft.SignedMessage) (decided bool, cd *ConsensusData, error)
	// HasRunningInstance returns true if there is a running consensus instance
	HasRunningInstance() bool
}

type ConsensusRunner struct {
	Share          *types.Share
	QBFTController *qbft.Controller
	BeaconNetwork  types.BeaconNetwork

	// highestDecidedSlot holds the highest decided duty slot and gets updated after each decided is reached
	highestDecidedSlot spec.Slot

}

type PartialSigRunner interface {
	GetBeaconNode() BeaconNode
	GetValCheckF() qbft.ProposedValueCheckF
	GetSigner() types.BeaconSigner
	GetNetwork() Network

	//TODO should add local duty? or use the one in cd? 
	UponDecided(cd *ConsesusData) error
	ProcessPostConsensus(signedMsg *types.SignedPartialSignatureMessage) error
}

type PartialSigRunner struct
	State          *State
	Share          *types.Share
	BeaconNetwork  types.BeaconNetwork
	BeaconRoleType types.BeaconRole
```

#### Happy Flow

1. `Cluster` receives all duties that match a certain slot, starts consensus for the relevant roles, and initializes the `PartialSigRunners` for the relevant Validators.
2. `Cluster` receives consensus messages and hands them over to the `QBFTController` that has unchanged logic.
3. Once `ProcessConsensus` returns `decided = true`, the `Cluster` will call `UponDecided(cd)` for each `PartialSigRunner` that has been initialized. This will cause an ommision of a partialSigMessage for each validator.
4. `PartialSigRunner` will process post-consensus partial signature messages as before.



### ClusterID

An identifier for cluster must be added to `MessageID`.

```go
[48]byte ClusterID

// Return a 48 bytes ID for the cluster of operators
func getClusterID(operatorIDs []OperatorID) ClusterID {
	// Create a 16 bytes constant prefix for cluters
	const prefix = []byte{0x00}

	// return the sha256 of the sortedIDs
	return ClusterID(prefix + sha256.Sum256(bytes(sorted(operatorIDs))))
}
```

In order to route consensus messages to the correct consensus runner, the `ClusterID` field will be included in the `MessageID` replacing `ValidatorPublicKey`.


#### Prefix Rationale

The 16 bytes prefix we are creating elongates the `ClusterID` to 48 bytes. The same length as `ValidatorPublicKey`. It may seem like a waste of space, but due to SSZ encoding it will actually save 16 bytes when compared to using a variable size array.

### MessageID

`ValidatorPublicKey` and `ClusterID` will be used interchangeably in `MessageID`. `Role` will change location because it is used to determine ID type.

```go
const (
	domainSize       = 4
	domainStartPos   = 0
	roleTypeSize     = 4
	// CHANGE IN Positions
	roleTypeStartPos =  domainStartPos + domainSize
	receiverIDSize   = 48
	receiverIDPos   = roleTypePos + roleTypeSize
)

// MessageID is used to identify and route messages to the right validator and Runner
type MessageID [56]byte

func (msg MessageID) GetDomain() []byte {
	return msg[domainStartPos : domainStartPos+domainSize]
}

func (msg MessageID) GetRoleType() BeaconRole {
	roleByts := msg[roleTypeStartPos : roleTypeStartPos+roleTypeSize]
	return BeaconRole(binary.LittleEndian.Uint32(roleByts))
}

func (msg MessageID) GetRecipientID() []byte {
	return msg[pubKeyStartPos : pubKeyStartPos+pubKeySize]
}
```


### Consensus Data

We note that the data needed for a consensus execution is the same for all validators. Thus, we can create a `ConsensusData` object that will hold the data for all validators. The data needed for the `SyncCommittee` role is a subset of the data needed for the `Attestation` role. Thus we can always query the beacon node for attestation data and pass this data to relevant runners.

`ConsensusData` currently holds the `duty` field to make sure that all committee members agree on committee information. However, if there is a difference between committee members there is no guarantee that a run will be triggered on the first place. Therefore we can rely on local view of `duty`.

### Consensus Message Validation

## P2P Message Validation

This duties transformation requires similar changes in message validation, namely:
  - Different consensus executions are tagged by the `MessageID`. This change would be propagated with no further issues. However, the `MessageID` is used to get the validator's public key and the duty's role which are used as an ID to store the consensus state. This must be changed to use the operators' committee and the duty's role, or even simply the `MessageID`.
- Message validation limits the number of attestation duties per validator by using the validator's public key contained in the `MessageID`. This is no longer possible. A new limitation can be accomplished by checking the number of validators a cluster of operators is assigned to. If this number is less than 32 (the number of slots in an epoch), then we can limit the number of attestation duties of such cluster per epoch. The only exception would be if such a cluster is assigned to a sync committee duty (considering that we will indeed merge attestations and sync committee duties altogether in the same consensus execution).

## Pre-requisites

- SIP #45