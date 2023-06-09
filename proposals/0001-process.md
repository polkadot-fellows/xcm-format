---
Title: RFC process
Number: 1
Status: Enacted
Version: 0
Authors:
 - Francisco Aguirre
Created: 2023-03-28
Impact: Trivial
Requires:
Replaces:
---

## Summary

Attempt to create an RFC (request for comment) process for XCM for making changes to the format.
Credits to the [Polkadot Protocol Proposals (PPP)](https://github.com/w3f/PPPs) and [Rust RFCs](https://github.com/rust-lang/rfcs) for inspiration.

## Motivation

XCM is a format, and while the only current implementation is the Rust one for the polkadot ecosystem, more could be created.
This brings about the importance of having this format be community-driven, and therefore the need for a process to propose changes to it.

## Specification

### When to submit an RFC?

RFCs should be submitted for new features or changes to the format.
These should be created by copying the base template (0000-template.md) and should have its frontmatter (the top part of the markdown file) updated accordingly.

### Versioning

As XCM is versioned, the `Version` field is the XCM version the RFC is planned for. This field is set by the maintainers.
An XCM version is a collection of enacted RFCs that gets solidified in the living document that is the `README.md` file.
All XCM versions can be found by tags on the corresponding commits in the master branch.

### RFC process

- Fork this repo.
- Create your proposal as proposals/0000-my-feature.md (where "my-feature" is descriptive. Don't assign an RFC number yet).
- Make sure to get the discussion going on and your RFC enters the review phase.
- An RFC can be modified based upon feedback from the maintainers and community.
- The review phase ends when a maintainer proposes to start a Final Comment Period (FCP). FCPs last for 10 days, after that, a decision is made about the RFC (approve, decline, postpone).
- Once your RFC is accepted, a PR to master should be made with the RFC changes made to the `README.md` file.
- Once all major implementations have integrated the proposed change into their codebase and the code was merged, the RFC should be updated to its final state, enacted. The PR to master should be merged.
- When all RFCs planned for an XCM version get merged, the final commit should be tagged with the version.

### RFC status

- Draft: The RFC was submitted and is waiting for the maintainers to start the review phase.
- Review: The RFC is being reviewed by the maintainers. The community, maintainers and author(s) are engaged in discussion about the RFC. The document can be edited until consensus is found. This state will end when a maintainer calles for a FCP.
- Declined: The RFC was declined. No further changes can be made to the document. The RFC has reached his final state, there are no further discussions to this RFC.
- Postponed: The decision to accept/decline the RFC was postponed. The RFC is either waiting for another implementation to happen or a certain time period in order to continue the discussions around it again. Either way, the maintainers will inform the author(s).
- Accepted: The RFC was accepted, no further changes to the document can be made. The proposed idea can be implemented and a related PR to the RFC can be opened.
- Enacted: All major implementations have included the RFC. This is the RFC’s final state.

### RFC Impact

- Breaking -> The RFC proposes breaking changes
- High -> The changes have a high impact on existing parts of the spec. E.g. changing an existing instruction.
- Low -> The changes don't have big impact on the existing parts of the spec. E.g. adding a new instruction.
- Trivial -> Mostly documentation/wording changes or small fixes.

## Impact

The impact of this RFC is trivial since it doesn't affect the spec, only the process to change it.

## Alternatives

- One alternative is to use the PPPs to implement XCM changes. The problem is that XCM isn't meant to be tied to Polkadot.
- Another alternative is to just use GitHub issues and PRs to deal with changes to the spec. This is definitely possible but this process adds structure useful in a community-driven standard.

## Questions and open Discussions

Is this structure useful for new proposals?
