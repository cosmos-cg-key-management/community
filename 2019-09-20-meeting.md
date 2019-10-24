# Cosmos Community Group on Key Management

## Agenda 2019-09-20

* Talk about proposed roadmap - finish fee delegation, generalized delegation, first, then groups and additional signature types like NIST P-256
* Review status of fee delegation https://github.com/cosmos/cosmos-sdk/pull/4616
* Review design of generalized delegation
* Discuss desired `FeeAllowance` and `Capability` implementations
* Discuss possible WASM hook points for `FeeAllowance` and `Capability`
* Who has bandwidth to work on any pieces?

## Minutes 2019-09-20

* next meeting scheduled for 8am PT/11am ET/5pm CET on Wed Oct 9th
* in the interim we will work on:
    * wrapping up https://github.com/cosmos/cosmos-sdk/pull/4616
    * upgrading the specification for groups to review at the next meeting
    * working on the PR for generalized delegation as time allows

# Current Specification

## Fee delegation

The `delegation` module also allows for fee delegation via some
changes to the `AnteHandler` and `StdTx`. The behavior is similar
to that described above for `Msg` delegations except using
the interface `FeeAllowance` instead of `Capability`:

```go
// FeeAllowance defines a permission for one account to use another account's balance
// to pay fees
type FeeAllowance interface {
	// Accept checks whether this allowance allows the provided fees to be spent,
	// and optionally updates the allowance or deletes it entirely
	Accept(fee sdk.Coins, block abci.Header) (allow bool, updated FeeAllowance, delete bool)
}
```

An example `FeeAllowance` could be created that simply sets a `SpendLimit`:

```go
type BasicFeeAllowance struct {
	// SpendLimit specifies the maximum amount of tokens that can be spent
	// by this capability and will be updated as tokens are spent. If it is
	// empty, there is no spend limit and any amount of coins can be spent.
	SpendLimit sdk.Coins
}

func (cap BasicFeeAllowance) Accept(fee sdk.Coins, block abci.Header) (allow bool, updated FeeAllowance, delete bool) {
	left, invalid := cap.SpendLimit.SafeSub(fee)
	if invalid {
		return false, nil, false
	}
	if left.IsZero() {
		return true, nil, true
	}
	return true, BasicFeeAllowance{SpendLimit: left}, false
}

```

Other `FeeAllowance` types could be created such as a daily spend limit.

## `StdTx` and `AnteHandler` changes

In order to support delegated fees `StdTx` and the `AnteHandler` needed to be changed.

The field `FeeAccount` was added to `StdTx`.

```go
type StdTx struct {
	Msgs       []sdk.Msg      `json:"msg"`
	Fee        StdFee         `json:"fee"`
	Signatures []StdSignature `json:"signatures"`
	Memo       string         `json:"memo"`
	// FeeAccount is an optional account that fees can be spent from if such
	// delegation is enabled
	FeeAccount sdk.AccAddress `json:"fee_account"`
}
```

An interface `FeeDelegationHandler` (which is implemented by the `delegation` module) was created and a parameter for it was added to the default `AnteHandler`:

```go
type FeeDelegationHandler interface {
	// AllowDelegatedFees checks if the grantee can use the granter's account to spend the specified fees, updating
	// any fee allowance in accordance with the provided fees
	AllowDelegatedFees(ctx sdk.Context, grantee sdk.AccAddress, granter sdk.AccAddress, fee sdk.Coins) bool
}

// NewAnteHandler returns an AnteHandler that checks and increments sequence
// numbers, checks signatures & account numbers, and deducts fees from the first
// signer.
func NewAnteHandler(ak AccountKeeper, fck FeeCollectionKeeper, feeDelegationHandler FeeDelegationHandler, sigGasConsumer SignatureVerificationGasConsumer) sdk.AnteHandler {
```

Basically if someone sets `FeeAccount` on `StdTx`, the `AnteHandler` will call into the `delegation` module via its `FeeDelegationHandler` and check if the tx's fees have been delegated by that `FeeAccount` to the key actually signing the transaction.

## Core `FeeAllowance` types

```go
type BasicFeeAllowance struct {
	// SpendLimit specifies the maximum amount of tokens that can be spent
	// by this capability and will be updated as tokens are spent. If it is
	// empty, there is no spend limit and any amount of coins can be spent.
	SpendLimit sdk.Coins
	// Expiration specifies an optional time when this allowance expires
	Expiration time.Time
}

type PeriodicFeeAllowance struct {
	BasicFeeAllowance
	// Period specifies the time duration in which PeriodSpendLimit coins can
	// be spent before that allowance is reset
	Period time.Duration
	// PeriodSpendLimit specifies the maximum number of coins that can be spent
	// in the Period
	PeriodSpendLimit sdk.Coins
	// PeriodCanSpend is the number of coins left to be spend before the PeriodReset time
	PeriodCanSpend sdk.Coins
	// PeriodReset is the time at which this period resets and a new one begins,
	// it is calculated from the start time of the first transaction after the
	// last period ended
	PeriodReset time.Time
}
```

# `delegation` module

The `delegation` module provides for generic delegation of blockchain actions
and fees to other accounts.

## `sdk.Msg` delegation

Delegations can be granted or revoked using the following messages:

```go
type MsgDelegate struct {
	Granter    sdk.AccAddress `json:"granter"`
	Grantee    sdk.AccAddress `json:"grantee"`
	Capability Capability     `json:"capability"`
	Expiration time.Time      `json:"expiration"`
}

type MsgRevoke struct {
	Granter sdk.AccAddress `json:"granter"`
	Grantee sdk.AccAddress `json:"grantee"`
	MsgType sdk.Msg        `json:"msg_type"`
}
```

Capabilities determine exactly what action is delegated. They are extensible
and can be defined for any `sdk.Msg` type even outside of the module
where the `Msg` is defined.

```go
type Capability interface {
	// MsgType returns the type of Msg's that this capability can accept
	MsgType() sdk.Msg
	// Accept determines whether this grant allows the provided action, and if
	// so provides an upgraded capability grant
	Accept(msg sdk.Msg, block abci.Header) (allow bool, updated Capability, delete bool)
}
```

For example a `SendCapability` like this is defined for `MsgSend` that takes
a `SpendLimit` and updates it down to zero:

```go
type SendCapability struct {
	// SpendLimit specifies the maximum amount of tokens that can be spent
	// by this capability and will be updated as tokens are spent. If it is
	// empty, there is no spend limit and any amount of coins can be spent.
	SpendLimit sdk.Coins
}

func (cap SendCapability) MsgType() sdk.Msg {
	return bank.MsgSend{}
}

func (cap SendCapability) Accept(msg sdk.Msg, block abci.Header) (allow bool, updated Capability, delete bool) {
	switch msg := msg.(type) {
	case bank.MsgSend:
		left, invalid := cap.SpendLimit.SafeSub(msg.Amount)
		if invalid {
			return false, nil, false
		}
		if left.IsZero() {
			return true, nil, true
		}
		return true, SendCapability{SpendLimit: left}, false
	}
	return false, nil, false
}
```

A different type of capability for `MsgSend` could be implemented
using the `Capability` interface with new need to change the underlying
`bank` module.

For simplicity a `Granter` can grant a `Grantee` exactly one `Capability`
for a given `sdk.Msg` type at a time. Grants can be given with an
optional `Expiration` time.

The `delegation` keeper takes a reference to the `BaseApp` `Router` and
provides a `DispatchActions(ctx sdk.Context, sender sdk.AccAddress, msgs []sdk.Msg) sdk.Result`
that can be called from another module (such as the `group` module or a
contracts module) to safely dispatch actions back the the router
in a way that respects the authentication behavior managed by the `delegation`
module. By default if there are no delegations the `sender` to `DispatchActions`
must be equal to value of `GetSigners()` for an `sdk.Msg`.

To execute a delegated action `MsgExecDelegatedAction` can be used:

```go
type MsgExecDelegatedAction struct {
	Signer sdk.AccAddress `json:"signer"`
	Msgs   []sdk.Msg      `json:"msg"`
}
```

# `group` module

The group module allows "key groups" to be created. Groups are collections of members without
any voting policy or account - just a weighted aggregation of other accounts. A group account
associates a group with a `DecisionPolicy` - which could be a simple threshold or percentage
or some more complex mechanism. Group accounts get their own `sdk.AccAddress` and can own coins.


```go
type GroupID uint64

// Groups get their own GroupID
type MsgCreateGroup struct {
	Signer sdk.AccAddress `json:"signer"`
	// The Owner of the group is allowed to change the group structure. A group account
	// can own a group in order for the group to be able to manage its own members
	Owner  sdk.AccAddress `json:"owner"`
	// The members of the group and their associated weight
	Members []Member `json:"members,omitempty"`
	// TODO maybe make this something more specific to a domain name or a claim on identity? or Info leave it generic
	Memo string `json:"memo,omitempty"`
}

// group accounts get their own sdk.AccAddress
type MsgCreateGroupAccount struct {
	Signer         sdk.AccAddress `json:"signer"`
	// The Owner of a group account is allowed to change the DecisionPolicy. This can be left nil 
	// in order for the group account to "own" itself
	Owner          sdk.AccAddress `json:"owner"`
	Group          GroupID        `json:"group"`
	DecisionPolicy DecisionPolicy `json:"decision_policy"`
	Memo           string         `json:"memo,omitempty"`
}

type Tally struct {
	YesCount sdk.Int
	NoCount sdk.Int
	AbstainCount sdk.Int
	VetoCount sdk.Int
}

// DecisionPolicy allows for flexibility in decision policy based both on
// weights (the tally of yes, no, abstain, and veto votes) and time (via
// the block header proposalSubmitTime)
type DecisionPolicy interface {
	Allow(tally Tally, totalPower sdk.Int, header types.Header, proposalSubmitTime time.Time)
}

type ThresholdDecisionPolicy struct {
	// Specifies the number of votes that must be accumulated in order for a decision to be made by the group.
	// A member gets as many votes as is indicated by their Weight field.
	// A big integer is used here to avoid any potential vulnerabilities from overflow errors
	// where large weight and threshold values are used.
	DecisionThreshold sdk.Int `json:"decision_threshold"`
}

type PercentageDecisionPolicy struct {
	Percent sdk.Dec `json:"percent"`
}

// A member specifies a address and a weight for a group member
type Member struct {
	// The address of a group member. Can be another group or a contract
	Address sdk.AccAddress `json:"address"`
	// The integral weight of this member with respect to other members and the decision threshold
	Weight sdk.Int `json:"weight"`
}
```

Because a group has its own `sdk.AccAddress` group members can also be other
groups so that groups can be nested.

## Proposals

Groups can execute any authorized action on the blockchain using their group
`sdk.AccAddress` by approving proposals.

Proposals have a simple creation, voting, and execution behavior based on the
following messages:

```go
type MsgCreateProposal struct {
	Proposer sdk.AccAddress `json:"proposer"`
	Group    sdk.AccAddress `json:"group"`
	Msgs     []sdk.Msg      `json:"msgs"`
	// Exec can be set to true in order to attempt to execute the proposal immediately
	// with no voting in a single transaction - this is useful for 1/N or 2/N multisig
	// key groups. Every signer of the MsgCreateProposal transaction is considered a yes
	// vote
	Exec bool `json:"exec,omitempty"`
}

type Vote int

const (
	No Vote = iota
	Yes
	Abstain
	Veto
)

type MsgVote struct {
	ProposalID ProposalID     `json:"proposal_id"`
	// Voters must sign this transaction
	Voters     []sdk.AccAddress `json:"voters"`
	Vote       Vote           `json:"vote"`
}

type MsgTryExecuteProposal struct {
	ProposalID ProposalID     `json:"proposal_id"`
	Signer     sdk.AccAddress `json:"signer"`
}
```

As one can see proposal `Msgs` can be any `sdk.Msg`. The execution of these
messages is handled by the `delegation` module which checks whether or not
the group is authorized to execute the provided `Msg` and routes these
messages back to the `BaseApp` `Router` if so.

Groups can use proposals to update their group members and decision policies.

