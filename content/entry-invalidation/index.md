---
rfc:
start_date: 2018-02-12
pr:
status: draft
---

# Entry Invalidation

## Summary

This RFC proposes a way to invalidate an entry.


## Motivation

Currently, the append-only nature of Registers doesn't cope well with
mistakes. A situation that has arised more than once is getting two records
about the same thing with different identifiers because a human mistake.
Another situation is having a change in the history of a record that is wrong.

We need a way to mark entries as invalid so we can keep compatibility with
other copies of the Register (consumers, replicas, etc).

## Explanation

The proposal is to introduce a new type of entry that allows listing a set of
entry identifiers [TODO: entry number or entry hash?].  This mechanism would
allow invalidating discrete entries and by extension invalidating the full
history for a key.


### Special entry

This approach defines a new type of entry. Caveat is that the shape of the
entry is incompatible with the existing one.

#### RSF

By entry hash

```
invalidate-entry	user	2018-02-12T10:11:12Z	sha-256:0000000000000000000000000000000000000000000000000000000000000000;sha-256:0000000000000000000000000000000000000000000000000000000000000001
```

Or by entry number

```
invalidate-entry	user	2018-02-12T10:11:12Z	3;234;355
```

#### JSON

```json
{
  "entry-number": "356",
  "revoked-entries": ["3", "234", "355"],
  "entry-timestamp": "2018-02-12T10:11:12Z"
}
```


## Alternative approach: Reserved entry key + new item shape

This approach keeps entries uniform but claims a key as reserved. The
description of what entries are revoked is held in a kind of item with a
specific shape.

[TODO: Review if "entries:revoked" is the right reserved key]

#### RSF

```
append-entry	user	entries:revoked	2018-02-12T10:11:12Z	sha-256:0000000000000000000000000000000000000000000000000000000000000000
```

#### JSON

```json
# Entry 356

{
  "index-entry-number": "356",
  "entry-number": "356",
  "key": "entries:revoked",
  "item-hash": [ "sha-256:0000000000000000000000000000000000000000000000000000000000000000"],
  "entry-timestamp": "2018-02-12T10:11:12Z"
}

# sha-256:0000000000000000000000000000000000000000000000000000000000000000

{
  "id": "entries:revoked",
  "entry-numbers": ["3", "234", "355"]
}
```

#### Properties

* Entries are consumed regularly (no breaking changes).
* Knowledge of the reserved key is required to filter it out when collecting
  list data.
* Records don't list reserved keys. You don't get the same information if you
  get it from `/records` or from `/entries`. Emphasises the dichotomy between
  porcelain and plumbing.
* Revoked entries require a mechanism/documentation to explain how to apply
  them when creating the list of records.
* Entry proofs are kept intact and invalidations are part of the tree.
