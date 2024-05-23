---
Title: Remove SetFeesMode instruction
Number: 0
Status: Draft
Version: 5
Authors:
 - Francisco Aguirre
Created: 2024-05-23
Impact: Medium
Requires:
Replaces:
---

## Summary

The `SetFeesMode` instruction and the `fees_mode` register allow for the existence of JIT withdrawal.
This mode complicates the fee mechanism and leads to bugs.
The proposal is to remove said functionality.
This RFC doesn't require RFC#58 but it works alongside it to simplify fee handling.

## Motivation

The JIT withdrawal mechanism creates bugs such as not being able to get fees when all assets are put into holding and none left in the origin location.
This is a confusing behavior, since there are funds for fees, just not where the XCVM wants them.
The XCVM should have only one entrypoint to fee payment, the holding register.
That way there is also less surface for bugs.

## Specification

The `SetFeesMode` instruction will be removed, meaning its corresponding paragraph on the spec will also be removed.
The `Fees Mode` register is not listed in the XCVM Registers list on the spec, so that'll be unaffected.

## Security considerations

The proposal aims to improve security by simplifying and reducing bug and attack surface.

## Impact

The impact will be bigger than that of introducing a new instruction, but it will still not be high as it will go in a new XCM version.
XCMs before this RFC that use the removed instruction might work in unexpected ways if the XCM is converted to the new version.

## Alternatives

None.

## Questions and open discussions (optional)

Was this feature adding any value to warrant the added complexity?
