---
Title: New instructions for staking
Number: 46 
Status: Draft
Version: 0
Authors:
 - Gonçalo Pestana (@gpestana)
 - Ankan (@ank4n)
Created: 2023-10-03
Impact: Trivial
Requires: -
Replaces: -
---

## Summary

This RFC proposed a set of new instructions to be used in the context of cross-consensus staking.

## Motivation

In this section you should describe why this is a valuable problem to solve. What use cases does
it support and who will benefit from this change? Why is this solution the best in the space of
possible solutions?

The proposed staking instructions are meant to define a general interface to perform staking that
requires work across multiple consensus systems. The goal is to standarize a minimal interface for
staking through XCM that can be used in the context of relay-chain staking, system parachain
staking and across the whole ecosystem.

## Specification

Explain the change or feature as you would to another builder in the ecosystem.

Describe the syntax and semantics of any new feature. Any new terminology or new named concepts should be defined in this section.

Explain the design in sufficient detail for somebody familiar blockchain to understand. Consider:

- Its interaction with other features
- Corner cases are dissected by example

Provide some examples of how it will be used.

## Security considerations

The purpose of this section is both to encourage authors to consider security in their designs and to inform the reader of relevant security issues.

Discuss here security implications/considerations relevant to the proposed change. Go through detected threats and risks and how they affected security-relevant design decisions. Add any kind of security concerns that are worth discussing during the process of this RFC.

## Impact

- What impact does this have on the rest of the spec? Does it result in a lot of changes to other parts of it?
- Does this introduce breaking changes? How would they be handled?

## Alternatives

- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

## Questions and open Discussions (optional)

Drop here the questions you require feedback from on this process.
Also consider open questions for further changes.
