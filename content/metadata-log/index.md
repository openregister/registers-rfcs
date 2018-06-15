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


### Target

The `target` property is crucial to keep a connection between the two logs. It
uses the data entry hash to prevent unexpected replacements of data that could
occur in the data log.

### Parent

The `parent` property works in a similar way as in Git's commits. The
intention is to explore a linked list structure instead of the ordered list
implemented for the data log.

### Delta

---
TODO: Ensure fields express everything they need to express (e.g. datatype,
cardinality, description). Ideally description should be handled aside from
datatype and cardinality so we could hash-check the pair never changes.
---

The `delta` property keeps the data to apply on top of the previous metadata
state. A delta allows mutliple bits of data so it can describe an update for
multiple keys at the same time. For example:

```elm
a0 : Delta
a0 =
  [ ("id", "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
  , ("name", "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5")
  , ("field:country", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
  ]
```

This delta applied to an empty state yields the same state:

```elm
apply : Delta -> State -> State

m0 : State
m0 =
  []

m1 : State
m1 =
  apply a0 m0

m1 == a0 == [ ("id", "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
            , ("name", "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5")
            , ("field:country", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
            ]
```

A second delta `a1` such as

```elm
a1 : Delta
a1 =
  [ ("field:name", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b") ]
```

is applied to a previous state `m1` as:

```elm
m2 : State
m2 =
  apply a1 m1

m2 == [ ("id", "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
      , ("name", "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5")
      , ("field:country", d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
      , ("field:name", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
      ]
```

**Blobs**, similarly to Items, can be treated as a dictionary (hash map):

```elm
blobs: Dict String Blob
blobs =
  Dict [ ("aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7", Blob.String "country")
       , ("701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5", Blob.String "Country")
       , (d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b", Blob.FieldType {cardinality: "1", datatype: "string"})
       ]
```
