---
Title: ExecuteWithOrigin instruction
Number: 38
Status: Draft
Version: 4
Authors:
 - Francisco Aguirre
Created: 2023-06-21
Impact: Low
Requires:
Replaces:
---

## Summary

The proposed change is the introduction of an `ExecuteWithOrigin` instruction.
The instruction allows the XCVM to execute a few instructions with a particular origin and then return to the original origin when those instructions are done.
It can be seen as pushing a new origin on a stack for the execution of a block of instructions and then popping it back when the block is done.
The new origin can either be empty (clear the origin) or a child of the current origin.

The goal is to give developers more flexibility and a better experience when handling origins in their messages.

## Motivation

Right now, XCM has some good instructions for manipulating the origin during execution in same ways, like `ClearOrigin` and `DescendOrigin`.
However, these instructions are final, once you use them, there's no standard way of going back to the original origin.
This results in a complicated developer experience, where the order of operations needs to be highly taken into account to perform the operations needed, with the correct origins for each.

This new instruction, `ExecuteWithOrigin`, provides a way of executing a set of instructions with a particular origin.
It makes scenarios where multiple operations need to be performed by multiple origins much easier to do.
It gives a standard way of doing things like buying execution from a user's account and then returning to the previous origin to perform other operations that require the priviledges associated with it.
It also allows for previously impossible scenarios like acting on behalf of many sibling origins.

## Specification

The instruction looks like this:

```rust
ExecuteWithOrigin { origin: Option<InteriorMultiLocation>, xcm: Xcm }
```

If the `origin` parameter is `None`, then `ClearOrigin` will be called before executing the inner `xcm`.
The previous origin will be restored once the inner `xcm` has finished executing.

If the `origin` parameter is `Some(interior_location)`, then `DescendOrigin(interior_location)` will be called before executing the inner `xcm`.
The previous origin will be restored once the inner `xcm` has finished executing.

### Examples

#### Clearing the origin

```rust
// Withdraw assets from the origin
WithdrawAsset(/* ...snip... */);
ExecuteWithOrigin {
 origin: None,
 xcm: Xcm(vec![
  // Deposit assets without an origin
  DepositAsset { /* ...snip... */ }
 ]),
}
```

#### Buying execution with an account

```rust
/* Origin: ../Parachain(1000) */

ExecuteWithOrigin {
 origin: Some(AccountId32 { /* ...snip... */ }),
 xcm: Xcm(vec![
  BuyExecution { /* ...snip... */ },
 ].into()),
}
// Transact with the parachain origin
Transact {
 origin_kind: OriginKind::SovereignAccount,
 /* ...snip... */
}
```

#### Acting on behalf of two sibling accounts

Here we withdraw some assets from two different accounts and then deposit them into another account.

```rust
/* Origin: ../Parachain(1000) */

ExecuteWithOrigin {
 origin: Some(AccountId32 { /* First account */ }),
 xcm: Xcm(vec![
  WithdrawAsset(/* ...snip... */),
 ].into()),
}
ExecuteWithOrigin {
 origin: Some(AccountId32 { /* Second account */ }),
 xcm: Xcm(vec![
  WithdrawAsset(/* ...snip... */),
 ].into()),
}
DepositAsset { /* Third account */ }
```

## Security considerations

This instruction does not allow for arbitrary origin manipulation, which would be a serious issue.
It only mixes the current `DescendOrigin` and `ClearOrigin` instructions in an easier-to-use way.

## Impact

The impact is Low since it introduces a new instruction. XCVM implementations would need to be updated.

## Alternatives

The alternative right now is to use `ClearOrigin` or `DescendOrigin` by themselves, which depends on the order of operations, which is subject to barriers.
There are some situations that are impossible to express without this new instruction, like dealing with multiple sibling origins.
