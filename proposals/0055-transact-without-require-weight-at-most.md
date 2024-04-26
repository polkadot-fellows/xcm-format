---

Title: Transact to support Unlimited weight limits  
Number: 55  
Status: Draft  
Version: 1  
Authors:
 - Adrian Catangiu
 - Francisco Aguirre

Created: 2024-04-25  
Impact: High  
Requires:  
Replaces:  

---

## Summary

The Transact instruction doesn't mandatorily need a `require_weight_at_most` weight, since the decoder could just get the weight of the call from its dispatch info.

## Motivation

The UX of using `Transact` is not great, and one part of the problem is guesstimating this `require_weight_at_most`.
We've seen multiple `Transact`s on-chain failures caused by the "incorrect" use or the parameter. In practice,
this parameter only adds UX overhead. The vast majority of use cases fall in one of two categories:
1. Unpaid execution of Transacts - in these cases the `require_weight_at_most` is not really useful, caller doesn't 
have to pay for it, and on the call site it either fits the block or not;
2. Paid execution of _single_ Transact - the weight to be spent by the Transact is already covered by the `BuyExecution`
weight limit parameter.

We've had multiple OpenGov `root/whitelisted_caller` proposals initiated by core-devs completely or partially fail
because of incorrect configuration of `require_weight_at_most` parameter. This is a strong indication that the
instruction is hard to use.

## Specification

The proposal is simple: change `Transact` instruction:

```diff
- Transact { origin_kind: OriginKind, require_weight_at_most: Weight, call: DoubleEncoded<Call> },
+ Transact { origin_kind: OriginKind, weight_limit: WeightLimit, call: DoubleEncoded<Call> },
```

With this new API, users who do not need to artificially limit the maximum weight used by the inner `call`,
can pass `weight_limit: Unlimited`; while those who need to do it, still can.

## Security considerations

Currently, an XCVM implementation can weigh a message just by looking at the decoded instructions without decoding the Transact's call, but assuming `require_weight_at_most` weight for it. With the new version it has to decode the inner call to know its actual weight.
But this does not actually change the security considerations, as can be seen below.

When using `weight_limit = Unlimited`, the weighing happens after decoding the inner `call`. The entirety of the XCM program containing this `Transact` needs to be covered by enough bought weight using a `BuyExecution` or the origin allowed to do free execution.

The security considerations around how much can someone execute for free are the same for
both this new version and the old. In both cases, an "attacker" can do the XCM decoding (including Transact inner `call`s) for free by adding a large enough `BuyExecution` without actually having the funds available.
In both cases, decoding is done for free, but execution fails early on `BuyExecution`.

## Impact

- Impact limited to single instruction: `Transact`.
- It is an API breaking change for `Transact`. There is a clean path for version upgrading or downgrading of
`Transact`, but UIs will have to adapt and render the new API.

## Alternatives

Q1: What other designs have been considered and what is the rationale for not choosing them?
- completely removing the Transact `weight_limit` parameter:
  - Defaulting to `weight_limit = Unlimited` for the Transact without having it as a parameter should have no impact on the vast majority or maybe even _all_ current known usecases where XCM programs use a single `Transact`. This single `Transact` weight limit is implicitly covered by the `BuyExecution` weight limit. But what about _theoretical_ XCM programs that do multiple `Transact`s and maybe want finer control over the spend of each with ability to branch logic using error handlers if one goes over its configured weight limit.
  - As far as I can tell, uses of `weight_limit = Limited(X)` are only theoretical, but they do exist.
  - The cost of keeping this dedicated `weight_limit: WeightLimit` parameter should be minimal in terms of implementation and/or cognitive complexity and/or UX. The vast majority of `Transact` usage will happen with `weight_limit = Unlimited`.
  - From an UX point of view, defaulting to `Unlimited` is easy; choosing an actual `Limited(X)` limit implies thinking about `X` ergo using a correct `Limited(X)` only if actually needed. So it's unlikely it will be a foot-gun.
  - `Limited(X)` enum variant might never be used/useful in practice, but better to be future proof? As seen above the cost of keeping it should be very low.

Q2: What is the impact of not doing this?
- UX suffers, we'll keep seeing on-chain failures for scenarios where the param value would have been irrelevant anyway

## Questions and open Discussions (optional)

None yet.
