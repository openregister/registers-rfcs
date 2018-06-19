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

### Use cases (potential)

TODO

## Explanation

A metadata log is a list of **changesets** where each changeset has:

* `timestamp`: The datetime where this changeset was created.
* `target`: A reference to the data log entry hash it applies.
* `parent`: A reference to the previous changeset hash.
* `delta`: An ordered set of pairs (`key`, `hash`) describing the **delta** of changes where:
  * `key`: Name of the piece of data (e.g. "name", "description",
      "field:country").
  * `hash`: A reference to the relevant **blob** hash.

```elm
type Delta =
  OrdSet (Key, Hash)

type Changeset =
  { timestamp: Timestamp
  , target: Maybe Hash
  , parent: Maybe Hash
  , delta: Delta
  }
```

### Timestamp

The `timestamp` property describes when the changeset was recorded. They don't
define the order of the metadata log.

### Target

The `target` property is crucial to keep a connection between the data log and
the metadata log. It uses the data entry hash to prevent unexpected
replacements of data that could occur in the data log.

The first changeset expects `target` to be nil given that the first item in
the data log must conform to a defined schema. Optionally, new changesets can
be recorded on top of the first one without `target`. Once there is a
changeset with a explicit `target` no more nil `target` properties are
allowed.

Rough algorithm given a new changeset:

1. If it's the first changeset:
  * Succeed if `target` is nil.
  * Fail otherwise.
2. If it's not the first changeset:
  * If `target` is nil:
    * Succeed if parent's `target` is nil.
    * Fail otherwise.

### Parent

The `parent` property works in a similar way as in Git's commits. The
intention is to explore a linked list structure instead of the ordered list
implemented for the data log.

The first changeset is expected to have no parent. Any other changeset will
have a single parent hash informed.

### Delta

The `delta` property has the data to apply on top of the previous metadata
state. A delta allows mutliple bits of data so it can describe an update for
multiple unique keys at the same time. For example:

```elm
type Delta =
  OrdSet Key Hash

a0 : Delta
a0 =
  [ ("custodian", "72bb44793c2e42872ebc892f411dab0f700049231a6a4169f87a20580d7cd516")
  , ("field:country", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
  , ("id", "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
  , ("name", "dcd1d5223f73b3a965c07e3ff5dbee3eedcfedb806686a05b9b3868a2c3d6d50")
  ]
```

Note a delta is an ordered set ordered by key.


TODO: This attempt to describe the resulting metadata state shows that the very
first changeset requires at least `id`, `custodian` and the field acting as
the identifier for data items (i.e. primary key).

```elm
type PrimitiveType
  = StringType
  | IntegerType
  | CurieType
  | ... -- Not all listed for brevity

-- Cardinality renamed here because it's combined with the data type itself as
-- they are two parts of the same thing.
type Datatype
  = One PrimitiveType
  | Many PrimitiveType

type Field =
  { id: String
  , datatype: Datatype
  , label: Maybe String
  , description: Maybe String
  }

type State
  = Empty
  | State { id: String
          , name: Maybe String
          , description: Maybe String
          , custodian: String
          , fields: Set Field
          , primaryKey: FieldId -- TODO: Any other better name to describe the Id?
          }
```

TODO (relevant?): This delta applied to an empty state yields the
derreferenced objects:

```elm
apply : Delta -> State -> State

m0 : State
m0 =
  Empty

m1 : State
m1 =
  apply a0 m0

m1 == State { id: "country"
            , name: Just "Country"
            , description: Nothing
            , custodian: "Foreign & Commonwealth Office"
            , fields: Set [ { id: "country", datatype: One StringType } ]
            , primaryKey: "country"
            }
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

m2 == State { id: "country"
            , name: Just "Country"
            , description: Nothing
            , custodian: "Foreign & Commonwealth Office"
              fields: Set [ Field { id: "country"
                                  , datatype: One StringType
                                  , label: Just "Country"
                                  , description: Just "The country's 2-letter ISO 3166-2 alpha2 code."
                                  }
                          , Field { id: "name"
                                  , datatype: One StringType
                                  , label: Just "Name"
                                  , description: Just "The name of the country."
                                  }
                          ]
            , primaryKey: "country"
            }

```

**Blobs**, similarly to Items, can be treated as a dictionary (hash map):

```elm
-- TODO: Probably better to stick to the current definition and use a
-- JSON-like string
type Blob = String

blobs: Dict Hash Blob
blobs =
  Dict [ ("aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7", "\"country\"")
       , ("701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5", "\"Country\"")
       , ("d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b", "{\"cardinality\":\"1\",\"datatype\":\"string\"}")
       ]
```

Blob hashing uses the same algorithm as the Items.


### Changeset example

This code exemplifies two changesets where the first is the parent of the
second.

```elm
type Changeset =
  { timestamp: DateTime
  , target: Maybe Hash
  , parent: Maybe Hash
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
  , target: Just "0000000000000000000000000000000000000000000000000000000000000000"
  , parent: Just "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6"
  , delta: [ ("field:name", "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b") ]
  }

getHash chs1 -- Hash "62bf2dae9312a9080f945caaf035fd512c8d5ddd1189cfb7ae04489e564ca379"
```

## High-level (porcelain) API

There are new endpoints that surface the computed state for the metadata log.

### Get the computed schema

* Endpoint: `GET /schema/`
* Parameters:
  * `entry-number`: A valid data log entry number (range: 1..[end of log]).

TODO: The schema will not be provided as CSV unless we find it essential and
we find a reasonable way to flatten the structure.

Response example in JSON:

```json
{
  "id": "country",
  "name": "Country",
  "custodian": "Foreign & Commonwealth Office",
  "fields": [
    {
      "id": "country",
      "datatype": "string",
      "label": "Name",
      "description": "The country's 2-letter ISO 3166-2 alpha2 code.",
    }, {
      "id": "name",
      "datatype": "string",
      "label": "Name",
      "description": "The name of the country."
    }
  ],
  "primary-key": "country"
}
```

## Low-level (plumbing) API

The following endpoints are low level
