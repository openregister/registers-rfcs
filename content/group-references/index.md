---
rfc:
start_date: 2018-01-05
status: draft
---

# Group references

## Summary

This RFC proposes an evolution to reference multiple records from one record.

## Motivation

Currently, the Registers Data Model allows references for the following cases:

* a) From one record to another record.
* b) From one record to a explicit list of records. It is the same case as above
  but with cardinality n.
* c) From one record to a register (e.g.
  https://statistical-geography.register.gov.uk).

The first two are well defined in intention and are predictable in expected
behaviour. The name of the field qualifies (or rather should qualify) the type
of relationship so data consumers can understand what are they going to find
if they dereference it.

The third case is less clear on what is intended to express. Could be either
the name of the register, a foreign key to the register register (so a
reference to that record) or something else qualified by the description of
that field. Assuming we all agree it describes a relationship to the group of
things described by that register we could accept it as a valid solution.

There are a few situations not covered by the listed solutions and for some of
them we have evidence that we need to find a solution that either enhances the
current ones or that adds to the list of possible options. We would like to
solve references for:

* From one record to a group of records that exceed the amount of references
  you would consider reasonable to list with a solution like (b). E.g you need
  to reference a few hundreds or records.
* From one record to a group of records that is the combination of multiple
  registers. E.g. you need to reference to "all local authorities".

Any record with the field “organisation” pointing to a CURIe for `register` or
with the field name stating “en bloc” you can find in
https://public-body-account.alpha.openregister.org.


## Explanation

TODO
