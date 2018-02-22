---
rfc:
start_date: 2018-02-12
pr:
status: draft
---

# Entry Invalidation

## Summary

This RFC proposes a way to invalidate an entry so previous changes considered
mistakes can be removed from the computed list of records.


## Motivation

Currently, the append-only nature of Registers doesn't cope well with
mistakes. A situation that has arised more than once is getting two records
about the same thing with different identifiers because a human mistake.
Another situation is having a change in the history of a record that is wrong.

We need a way to mark entries as invalid so we can keep compatibility with
other copies of the Register (consumers, replicas, etc) and allow computing
the correct(ed) list of records.


## Explanation

[TODO: Currently two alternatives are explored]


### Alternative A: Special entry

The proposal introduces a new type of entry that allows listing a set of entry
identifiers. This mechanism would allow invalidating discrete entries and by
extension invalidating the full history for a key. It also introduces a new
RSF command to avoid overloading the `append-entry` command.

[TODO: entry number or entry hash?]

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

#### Properties

* New entry type without key. Breaking change diverging from previous designs.
* Records are unaffected.
* Revoked entries require a mechanism/documentation to explain how to apply
  them when creating the list of records.
* Entry proofs are kept intact and invalidations are part of the tree.



### Alternative B: Reserved entry key + new item shape

The proposal introduces a reserved key and a new type of item where entry
identifiers are listed. This mechanism allows invalidating discrete entries
and by extension invalidating the full history for a key.


[TODO: Review if "chore:revoked-entries" is the right reserved key]


#### RSF

```
add-item	{"id":"chore:revoked-entries","entry-numbers":["3","234","355"]}
append-entry	user	chore:revoked-entries	2018-02-12T10:11:12Z	sha-256:0000000000000000000000000000000000000000000000000000000000000000
```

#### JSON

**Entry 356:**

```json
{
  "index-entry-number": "356",
  "entry-number": "356",
  "key": "chore:revoked-entries",
  "item-hash": [ "sha-256:0000000000000000000000000000000000000000000000000000000000000000"],
  "entry-timestamp": "2018-02-12T10:11:12Z"
}
```

**Item sha-256:0000000000000000000000000000000000000000000000000000000000000000:**

```json
{
  "id": "chore:revoked-entries",
  "entry-numbers": ["3", "234", "355"]
}
```

#### CSV

**Entry 356:**

```csv
index-entry-number,entry-number,entry-timestamp,key,item-hash
356,356,2018-02-12T10:11:12Z,chore:revoked-entries,"sha-256:0000000000000000000000000000000000000000000000000000000000000000"
```

**Item sha-256:0000000000000000000000000000000000000000000000000000000000000000:**

```csv
id,entry-numbers
chore:revoked-entries,3;234;355;
```

#### Properties

* Entries are consumed regularly (no breaking changes).
* This approach uses reserved key in the user space. Requires avoiding clashes
  with user data.
* Knowledge of the reserved key is required to filter it out when collecting
  list data.
* Records don't list reserved keys. You don't get the same information if you
  get it from `/records` or from `/entries`. Emphasises the dichotomy between
  porcelain and plumbing.
* Revoked entries require a mechanism/documentation to explain how to apply
  them when creating the list of records.
* Entry proofs are kept intact and invalidations are part of the tree.


