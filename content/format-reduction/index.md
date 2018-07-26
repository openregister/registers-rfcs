---
rfc: 0004
start_date: 2018-07-26
pr: openregisters/registers-rfc#21
status: approved
---

# Format reduction

## Summary

This RFC proposes to reduce the amount of representations a registers
implementation should support to JSON and CSV and recommends JSON as the main
representation. In this way, we expect that implementers, who are not sure of
which format to implement, will use it by default.

## Motivation

The Registers data model and its REST API are still subject to change. The
amount of formats which we currently account for unnecessarily complicates the
implementation. For example, we want to evolve who the schema fits into the
data model and how the schema is exposed to the user.


## Explanation

The current suggestion in the specification is to implement JSON, YAML, CSV,
TSV, HTML and Turtle. Although none of them are required, the reference
implementation implements all of them.

Some issues to contextualise this are:

* The specification is not yet mature enough by itself to guide implementers on
  the implementation of Turtle.
* The Turtle implementation is incorrect.
* The YAML representation offers no value on top of the JSON one.
* The CSV and TSV formats are virtually the same.
* The HTML representation has diverged from the other representations in the
  reference implementation, and is no longer an exact analogue. Users have
  also found the HTML representation confusing in the reference
  implementation.

As of 2018, users generally favour JSON (89.9%) and a small fraction benefit
from a tabular form (CSV 3.5%, TSV 1.9%).

In light of that, the suggestion is to remove all formats but JSON and CSV.
We should bring back RDF when the specification has a mature definition of how to
implement it. Also, JSON should be recommended as the main representation if
the implementer has doubts on what format to implement.

Finally, the specification should provide a method for users to ask a register
for available formats.
