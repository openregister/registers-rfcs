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

The proposal is to introduce a new type of entry under the “user” type that
allows listing a set of entry identifiers [TODO: entry number or entry hash?].
This mechanism would allow invalidating discrete entries and by extension
invalidating the full history for a key.

### RSF

TODO

By entry hash

```
append-entry	user	??	2018-02-12T10:11:12Z	sha-256:0000000000000000000000000000000000000000000000000000000000000000;sha-256:0000000000000000000000000000000000000000000000000000000000000001
invalidate-entry	user	2018-02-12T10:11:12Z	sha-256:0000000000000000000000000000000000000000000000000000000000000000;sha-256:0000000000000000000000000000000000000000000000000000000000000001
```

Or by entry number

```
append-entry	user	2018-02-12T10:11:12Z	3;234;355
invalidate-entry	user	2018-02-12T10:11:12Z	3;234;355
```


### JSON

```json
{
  "entry-number": "356",
  "revoked-entries": ["3", "234", "355"],
  "entry-timestamp": "2018-02-12T10:11:12Z"
}
```

TODO: It could be modelled as regular entries and special items.
