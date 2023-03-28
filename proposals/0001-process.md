---
Title: RFC process
Number: 1
Status: Enacted
Version: 0
Authors:
 - Francisco Aguirre
Created: 2023-03-28
Requires:
Replaces:
---

## Summary

Attempt to create an RFC (request for comment) process for XCM.
Credits to the [Polkadot Protocol Proposal (PPP)](https://github.com/w3f/PPPs) for the process.

## Motivation

XCM is a format, and while the only current implementation is the Rust one for the polkadot ecosystem, more could be created.
This brings about the importance of having this format be community-driven, and therefore the need for a process to propose changes to it.

## Specification

### When to submit an RFC?

RFCs should be submitted for new features or changes to the format.
These should be created by copying the base template (0000-template.md) and should have its frontmatter (the top part of the markdown file) updated accordingly.
As XCM is versioned, the `Version` field is the XCM version the RFC is aiming to be accepted in.
An XCM version is a collection of enacted RFCs that gets solidified in the living document that is the `README.md` file.
All XCM versions can be found by tagging the corresponding commits in the main branch.

### RFC process

- Fork this repo.
- Create your proposal as proposed-rfc/0000-my-feature.md (where "my-feature" is descriptive. Don't assign an RFC number yet).
- Make sure to get the discussion going on and your RFC enters the review phase.
- An RFC can be modified based upon feedback from the maintainers and community. Significant modifications may trigger a new final comment period, which is 4 weeks.
- Once your RFC is accepted the specification should be updated to match.
- Once all major implementations integrated the proposed change into their codebase and the code was merged, the RFC should be updated to its final state, enacted.

### RFC status

- Proposed: The RFC was submitted and is waiting for the maintainers to start the review phase.
- On review: The RFC is being reviewed by the maintainers. The community, maintainers and author(s) are engaged in discussion about the RFC. The document can be edited until consensus is found.
- Declined: The RFC was declined. No further changes can be made to the document. The RFC has reached his final state, there are no further discussions to this RFC.
- Postponed: The decision to accept/decline the RFC was postponed. The RFC is either waiting for another implementation to happen or a certain time of period in order to continue the discussions around it again. Either way, the maintainers will inform the author(s).
- Accepted: The RFC was accepted, no further changes to the document can be made. The proposed idea can be implemented and a related PR to the RFC can be opened.
- Enacted: All major implementations have included the RFC. This is the RFC’s final state.

## Alternatives

- One alternative is to use the PPPs to implement XCM changes. The problem with that is that XCM isn't meant to be tied to Polkadot.
- Another alternative is to just use GitHub issues and PRs to deal with changes to the spec. This is definitely possible but this process adds structure useful in a community-driven standard.

## Questions and open Discussions (optional)

Is this structure useful for new proposals?
