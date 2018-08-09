---
rfc: 0000
start_date: 2018-08-09
pr:
status: draft
---

# RFC process

## Summary

This RFC proposes a process to handle changes via RFCs (Request For Comments)
and how it works with the ADR (Architecture Decision Record) process.

## Motivation

The Registers project has multiple streams of work but all of them orbit
around the same data model and abstract thinking. Repositories like the
reference implementation have an ADR process that works well to document the
decisions made on it. The specification handles changes through Github issues
and sometimes both repositories need to change based on the same decision.

For the last couple of months we have been using an RFC process independent of
specification, reference implementation or API documentation. The result has
been satifactory and this RFC aims to formalise the process.


## Explanation

### When you need to follow this process

Open an RFC when the change to propose affects the foundations of Registers or
when it affects the whole ecosystem.

For example, if you want to propose a new datatype for periods, change the
meaning of entry timestamps, the addition of metadata or the algorithm for
hashing items.

### When you need to follow the ADR process

Open an ADR for the pertinent repository when the nature of the proposal is to
change the specifics of that piece of software.

For example, an RFC to introduce a new entrypoint would need an ADR in the
reference implementation repository to describe the specifics of the
implementation.

### What do you need to include

You should provide some metadata:

* `rfc` : The next number in sequence.
* `start_date`: The date the RFC started in ISO8601.
* `pr`: Once the RFC has a pull request open, the reference to it.
* `status`: The status of the RFC (`draft`, `approved`, `rejected`).

And the document body split into three sections:

* Title: A descriptive title to refer to the RFC.
* Summary: A short description of what is this RFC for.
* Motivation: An explanation of why you think the RFC is needed.
* Explanation: The proposal. Add the relevant subsections in it to make it
  easier to understand.

For example:

```markdown
---
rfc: 0999
start_date: 2018-08-09
pr: openregisters/registers-rfc#9981
status: draft
---

# An RFC example

## Summary

This RFC is an example to support creating new RFCs.

## Motivation

An example helps supporting the formal definition.

## Explanation

You are seeing it.
```

### Format and place in the RFC repository

Create a new directory under `content/` with a descriptive name (usually a
slug from the title will suffice) and the RFC file inside it with the name
`index.md`.

For example, the RFC for URI Consolidation is placed in
`content/uri-consolidation/index.md`.

If you have diagrams or pictures to complement the RFC, place them under the
same directory as the `index.md`.


### Lifecycle

The lifecycle is very simple as we are a small team. It should be expanded
only when it shows to be insufficient.

#### Status: draft

Create the new RFC file structure, write your proposal and open a Pull
Request. Ask for approval and iterate over the RFC to address any issue.

You should get two approvals before moving the RFC to approved.

#### Status: approved

If the RFC is approved, change the status to `approved`, merge to
`master` and close the pull request.

#### Status: rejected

If the RFC is rejected, change the status to `rejected`, merge to
`master` and close the pull request.
