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

* `timestamp`: The datetime where this changeset was created.
* `target`: A reference to the data log entry hash it applies.
* `parent`: A reference to the previous changeset hash.
* `delta`: A list of pairs (`key`, `blob`) describing the **delta** of changes where:
  * `key`: Name of the piece of data (e.g. "name", "description",
      "field:country").
  * `blob`: A reference to the relevant **blob** hash.

### Timestamp

The `timestamp` property describes when the changeset was recorded. Similarly
to Entry timestamps, they don't define the order of the metadata log, they are
mostly informative.

### Target

The `target` property is crucial to keep a connection between the two logs. It
uses the data entry hash to prevent unexpected replacements of data that could
occur in the data log.

The first changeset can optionally not provide a `target` and it's assumed it
will apply to the data log from the first entry. This allows recording the
first changeset when the data log doesn't exist which is expected because you
need a schema to validate the data is valid.

---

TODO: Is is acceptable to have multiple changesets with no `target` if none of
the previous changesets have one? It would allow incrementally defining the
schema before the first data entry gets recorded.

---

### Parent

The `parent` property works in a similar way as in Git's commits. The
intention is to explore a linked list structure instead of the ordered list
implemented for the data log.

The first changeset is expected to have no parent.

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

-- TODO: What if each pair has an Action (e.g. Add foo or Remove bar?)
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
      , ("field:country", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
      , ("field:name", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
      ]
```

**Blobs**, similarly to Items, can be treated as a dictionary (hash map):

```elm
type Cardinality
  = One
  | Many

type Blob
  = BlobString String
  | BlobFieldType {cardinality: Cardinality, datatype: Datatype}

blobs: Dict Key Blob
blobs =
  Dict [ ("aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7", BlobString "country")
       , ("701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5", BlobString "Country")
       , ("d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b", BlobFieldType {cardinality: One, datatype: DatatypeString})
       ]
```

Blob hashing uses the same algorithm as the Items.

---

TODO: Blobs that are strings, to be valid JSON need to have quotes. Do we want
this?

---

### Changeset example

This code exemplifies two changesets where the first is the parent of the
second.

```elm
type Changeset =
  { timestamp: DateTime
  , target: Option Hash
  , parent: Option Hash
  , delta: Delta
  }

chs0 =
  { timestamp: DateTime "2018-06-14T15:51:00Z"
  , target: Nothing
  , parent: Nothing
  , delta: [ ("id", "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
           , ("name", "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5")
           , ("field:country", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
           ]
  }

getHash chs0 -- Hash "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6"

chs1 =
  { timestamp: DateTime "2018-06-14T15:59:00Z"
  , target: Some "0000000000000000000000000000000000000000000000000000000000000000"
  , parent: Some "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6"
  , delta: [ ("field:name", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b") ]
  }

getHash chs1 -- Hash "62bf2dae9312a9080f945caaf035fd512c8d5ddd1189cfb7ae04489e564ca379"
```
