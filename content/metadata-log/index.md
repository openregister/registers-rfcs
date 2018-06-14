---
rfc:
start_date: 2018-06-14
pr:
status: draft
---

# Metadata log

## Summary

This RFC proposes a complementary log to record metadata changes without
affecting the current data log.


## Motivation

Registers started as pure data, and slowly they added different bits of
metadata. The reference implementation has a few bits of metadata (e.g.
description, name, fields) but the specification offers no way to consume
them.

This RFC aims to keep backwards compatibility by creating a new metadata log
to encode metadata changes with references to the data log to keep
coordination with the original data log.


## Explanation

A metadata log is a list of **changesets** where each changeset has:

* `target`: A reference to the data log entry hash it applies.
* `parent`: A reference to the previous changeset hash.
* `delta`: A list of pairs (`key`, `blob`) describing the **delta** of changes where:
  * `key`: Name of the piece of data (e.g. "name", "description",
      "field:country").
  * `blob`: A reference to the relevant **blob** hash.
* `timestamp`: The datetime where this changeset was created.



