---
Title: Introduce Agent instructions
Number: 42
Status: Draft
Version: 0
Authors:
 - vstam1
Created: 2023-08-08
Impact: Low
Requires:
Replaces:
---

## Summary

The RFC introduces the addition of agent instructions into XCM, specifically introducing three
new instructions: SetAgent, RemoveAgent, and TransactAsAgent. These instructions allow a
location to designate an agent to operate on its behalf, enabling the agent to dispatch calls
on behalf of the origin. This is particularly beneficial for local accounts, which can act as
agents for remote locations. This reduces the number of Transact instructions the remote
location needs to send. An agent is also able to dispatch calls via XCM using the
TransactAsAgent instruction. 

## Motivation

The introduction of the agent instructions is motivated by the need to allow for the delegation of
actions on the target system. This is particularly useful for setting a local account ID to act
on behalf of another location, which is presumably remote.

By allowing a local account to act as an agent, we can reduce the number of XCM Transact
messages that need to be sent. The agent can perform transactions locally, thereby substituting
numerous individual XCM Transact instructions. An agent can also dispatch calls for the origin
via XCM using the `TransactAsAgent` instruction. By enabling both local execution and execution
via XCM, we enhance the system's flexibility, accommodating a broader range of user needs and
scenarios.

An agent not only simplifies the process of performing actions on the target system, but also
reduces the load on the transport layer and potentially leads to cost savings.

## Specification

The proposed change introduces three new XCM instructions to facilitate the setting of an
agent. These instructions are as follows:

```rust
/// Set an agent for the `origin` of the instruction. 
///
/// - `agent`: The location that can act as an agent for the origin. 
/// - `permission`: Specifies the type of permissions that the agent has. 
/// If the privilege parameter is not provided, it defaults to a predefined value.
SetAgent(agent: MultiLocation, permission: Option<u8>),

/// Removes the specified agent for the origin of the instruction.
///
/// - `agent`: The location that is removed as agent for the origin. 
RemoveAgent(agent: MultiLocation),

/// Apply the encoded transaction specified by `call`, using the converted `origin` field as its dispatch-origin. 
/// The `origin` field is converted based on the origin_kind.
///
/// - `origin`: Specifies the location that the agent acts for.
/// - `origin_kind`: The means of expressing `origin` as a dispatch origin.
/// - `require_weight_at_most`: The weight of `call`; this should be at least the chain's
///   calculated weight and will be used in the weight determination arithmetic.
/// - `call`: The encoded transaction to be applied.
TransactAsAgent(origin: MultiLocation, origin_kind: OriginKind, require_weight_at_most: Weight, mut call: DoubleEncoded<Call>)
```

### Implementation details
These three instruction allow for different implementations of proxying transactions via an
agent. Based on the specifics of the underlying system, the XCVM implementation can choose
among the following approaches:

#### Option 1: Local Agent Execution Only

In this approach, the XCVM implementation would focus on the local execution of transactions.
Specifically, the system would:

- Implement the SetAgent and Remove Agent instructions. These instructions would be responsible
  for designating or removing an agent on the local system.
- Once an agent is set, it would be allowed to execute transactions, but only within the local
  system and not via the XCVM. This means that the agent would not utilize the
  `TransactAsAgent` instruction for these transactions.

A practical example of this approach can be the proxy pallet in FRAME. In this setup, the
SetAgent instruction directly designates the agent within the pallet. Once set, this agent has
the capability to process transactions on the local system using the `proxy` function within
the `proxy` pallet.

#### Option 2: Local Propagation with All Instructions

This approach is a bit more expansive, allowing for a broader range of transaction executions.
Here, the system would:

- Implement all three instructions: `SetAgent`, `RemoveAgent`, and `TransactAsAgent`.

- The `TransactAsAgent` instruction would be designed to forward or propagate the transaction
  call directly to the local system. Once received, the local system would then take over and
  manage proxy validation and the actual execution of the call.

A setup of this method can again be the proxy pallet of FRAME. In this scenario the
`TransactAsAgent` instruction forwards the call to the `proxy` function within the `proxy`
pallet for execution.

#### Option 3: XCM-only Transaction Application

This approach is more restrictive in terms of where transactions can be executed. In this
setup, the system would:

- Implement all three aforementioned instructions.

- However, the agent would be restricted to executing transactions exclusively through XCM.
  This means that all transactions processed by the agent would utilize the `TransactAsAgent`
  instruction.

The TransactAsAgent would be responsible for both the validation of the agent and the
dispatching of the transaction.


### Errors
The following errors are implementation-dependent and may occur when executing the new XCM
instructions:

```rust
/// This error can be thrown for several reasons, for example:
/// 
/// - An attempt is made to set an agent for a location that already has it as an agent.
/// - An attempt is made to reset an agent for a location that does not have an agent.
/// - The system has reached its limit for the number of agents.
/// - An attempt is made to set oneself as an agent.
UnableToSetAgent,
/// This error occurs when an attempt is made to perform a transaction as an agent without the necessary privileges using the TransactAsAgent 
/// instruction. The system should reject the transaction and return a NoPermission error message, 
/// indicating that the agent does not have the required privileges to perform the action.
NoPermission
```

## Security considerations

### Privilege Escalation through misuse of OriginKind
A vulnerability could arise from the potential misuse of the `TransactAsAgent` instruction. If
the conversion from the `origin` to the dispatch origin within the `TransactAsAgent`
instruction is not correctly managed and validated, it could allow a privilege escalation
attack. If the conversion is misconfigured, a user might specify `OriginKind: superuser` and
could gain super user privileges within the system. The same vulnerability exists for the
`Transact` instruction, and the configuration should be handled with the same care. 

## Impact
The impact of this RFC is low as it introduces only new instructions. 

## Alternatives

## Questions and open Discussions (optional)

Should we permit agents to dispatch transactions through XCM? Or would it be preferable to only
allow designating and removing an agent through XCM?

