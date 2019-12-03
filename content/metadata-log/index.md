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

Registers started as pure data, but have evolved as metadata has been added.
The reference implementation has some metadata (e.g.  description, name,
fields) but the specification offers no way to consume them.

This RFC proposes a new log to encode metadata changes which has references to
the original data log, in order to maintain backwards compatibility.

### Use cases

#### As a user I want to get a records and validate them against the schema

1. `GET /records/foo`
2. `GET /schema/`
3. validate

#### As a user I want to validate a record I got some time ago and validate it against the schema.

1. `GET /records/foo`
2. (time passes)
3. `GET /schema/`
4. validate


#### As a user I want to get a record at an arbitrary log size and validate it against the schema

1. `GET /records/foo?size=10`
2. `GET /schema/?size=10`
3. validate

---

TODO: Is the last use case artificial? Also, if schemas only allow adding
fields and all fields are optional, the schema at size HEAD should be
sufficient.

---

#### As a user I want to validate a record against the latest (correct) schema

1. `GET /schema/`
2. (time passes)
3. `GET /records/foo`
4. `GET /schema/`
5. validate

Essentially this means that either we provide a way to know if the schema is
the latest version, or we make it a requirement to always fetch a new version.

One potential issue arises when a new record is validated against an old
schema. This happens if (and only if) the new record has fields which were
defined in newer versions of the schema.

#### As a user I want to get a schema version

1. `GET /schema/{hash}`

---

TODO: Is this needed? It's not well defined use case.

---

---

TODO: What are the use cases for metadata outside of field information?

---


## Explanation

A metadata log is a list of **changesets** where each changeset has:

* `timestamp`: The datetime where this changeset was created.
* `target`: A reference to the data log entry hash it applies to.
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

type MetaLog =
  Dict Hash Changeset

previous : Changeset -> MetaLog -> Changeset

-- Computes a schema from the given changeset
schema : Changeset -> MetaLog -> Schema
```

### Timestamp

The `timestamp` property describes when the changeset was recorded. It does
not define the order of the metadata log.

### Target

The `target` property is crucial to keep a connection between the data log and
the metadata log. It uses the data entry hash to prevent unexpected
replacements of data that could occur in the data log.

The first changeset expects `target` to be nil given that the first item in
the data log must conform to a defined schema. Optionally, new changesets can
be recorded on top of the first one without `target`. Once there is a
changeset with a explicit `target`, no more nil `target` properties are
allowed.

A rough algorithm given a new changeset is as follows:

1. If it's the first changeset:
  * Succeed if `target` is nil.
  * Fail otherwise.
2. If it's not the first changeset:
  * If `target` is nil:
    * Succeed if parent's `target` is nil.
    * Fail otherwise.

---

TODO: In a situation (arguably common) where metadata is defined once and not
changed at all, the `target` would be nil for a long time (forever?). Does
it matter?

---

### Parent

The `parent` property works in a similar way as in Git commits. The
intention is to explore a linked list structure instead of the ordered list
implemented for the data log.

The first changeset is expected to have no parent. Any other changeset will
have a single parent hash informed.

### Delta

The `delta` property has the data to apply on top of the previous metadata
state. A delta allows multiple bits of data so it can describe an update for
multiple unique keys at the same time. For example:

```elm
type Delta =
  OrdSet Key Hash

a0 : Delta
a0 =
  [ ("custodian", Hash "72bb44793c2e42872ebc892f411dab0f700049231a6a4169f87a20580d7cd516")
  , ("field:country", Hash "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
  , ("id", Hash "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
  , ("name", Hash "dcd1d5223f73b3a965c07e3ff5dbee3eedcfedb806686a05b9b3868a2c3d6d50")
  ]
```

Note that a delta is an ordered set ordered by key.


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
  , name: Maybe String
  , description: Maybe String
  }

type Snapshot
  { id: String
  , name: Maybe String
  , description: Maybe String
  , custodian: String
  , hashAlgorithm: HashAlg, -- TODO: Can we introduce this bit of information at the register level?
  , fields: Set Field
  , primaryKey: Field
  }

type State
  = Empty
  | State Snapshot
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
            , fields: Set [] -- Empty set because only the Primary key is defined
            , primaryKey: Field { id: "country"
                                , datatype: One StringType
                                , name: Just "Country"
                                , description: Just "The country's 2-letter ISO 3166-2 alpha2 code."
                                }
            }
```

A second delta `a1`, such as:

```elm
a1 : Delta
a1 =
  [ ("field:name", Hash "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b") ]
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
              fields: Set [ Field { id: "name"
                                  , datatype: One StringType
                                  , name: Just "Name"
                                  , description: Just "The name of the country."
                                  }
                          ]
            , primaryKey: Field { id: "country"
                                , datatype: One StringType
                                , name: Just "Country"
                                , description: Just "The country's 2-letter ISO 3166-2 alpha2 code."
                                }
            }

```

**Blobs**, similarly to Items, can be treated as a dictionary (hash map):

```elm
type Blob = String

blobs: Dict Hash Blob
blobs =
  Dict [ (Hash "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7", "\"country\"")
       , (Hash "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5", "\"Country\"")
       , (Hash "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b", "{\"cardinality\":\"1\",\"datatype\":\"string\"}")
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
  , delta: [ ("id", Hash "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7")
           , ("name", Hash "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5")
           , ("field:country", Hash "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b")
           ]
  }

getHash chs0 -- Hash "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6"

chs1 =
  { timestamp: DateTime "2018-06-14T15:59:00Z"
  , target: Just (Hash "0000000000000000000000000000000000000000000000000000000000000000")
  , parent: Just (Hash "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6")
  , delta: [ ("field:name", Hash "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b") ]
  }

getHash chs1 -- Hash "62bf2dae9312a9080f945caaf035fd512c8d5ddd1189cfb7ae04489e564ca379"
```

## High-level (porcelain) API

There are new endpoints that surface the computed state for the metadata log.

### Get the computed schema

* Endpoint: `GET /schema/`
* Parameters:
  * `entry-number` (Optional): A valid data log entry number (range: 1..[end of log]).

---

TODO: The schema will not be provided as CSV unless we find it essential and
we find a reasonable way to flatten the structure.

Also our datatypes are not compatible with CSVW, XSD or JSON-Schema.

---

```http
GET /schema/ HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "country",
  "name": "Country",
  "custodian": "Foreign & Commonwealth Office",
  "fields": [
    {
      "id": "country",
      "datatype": "string",
      "cardinality": "1",
      "name": "Name",
      "description": "The country's 2-letter ISO 3166-2 alpha2 code.",
    }, {
      "id": "name",
      "datatype": "string",
      "cardinality": "1",
      "name": "Name",
      "description": "The name of the country."
    }
  ],
  "primary-key": "country"
}
```

## Low-level (plumbing) API

The following endpoints are low-level.

### Get the list of changesets

* Endpoint: `GET /meta/changesets/`
* Parameters:
  * `cursor` (Optional): Opaque pointer to the next collection page.
  * `size` (Optional): Collection size. Defaults to 100.

```http
GET /meta/changesets/ HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Link: </meta/changesets/?cursor=2&size=2>; rel="next"

[
  {
    "id": "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6",
    "timestamp": "2018-06-14T15:51:00Z",
    "delta": {
      "id": "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7",
      "name": "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5",
      "field:country": "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b"
    }
  }, {
    "id": "62bf2dae9312a9080f945caaf035fd512c8d5ddd1189cfb7ae04489e564ca379",
    "timestamp": "2018-06-14T15:59:00Z",
    "target": "0000000000000000000000000000000000000000000000000000000000000000",
    "parent": "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6",
    "delta": { "field:name": "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b" }
  }
]
```

### Get a single changeset

* Endpoint: `GET /meta/changesets/{id}`
* Parameters:
  * `id` (Required): The changeset hash.

```http
GET /meta/changesets/adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6 HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "adcd501c027ad83fbdf4c3423630da89b2c013b9e8641ec0c2679ed33b2cc0d6",
  "timestamp": "2018-06-14T15:51:00Z",
  "delta": {
    "id": "aff64e4fd520bd185cb01adab98d2d20060f621c62d5cad5204712cfa2294ef7",
    "name": "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5",
    "field:country": "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b"
  }
}
```

### Get the list of blobs

* Endpoint: `GET /meta/blobs/`
* Parameters:
  * `cursor` (Optional): Opaque pointer to the next collection page.
  * `size` (Optional): Collection size. Defaults to 100.

---

TODO: Parameters are informative and implementation dependant in the current
specification.

TODO: This pagination diverges from the ones offered by the data log. This is
to explore the idea of paginating naturally unordered collections in an
ordered way. In SQL this could take the form of a keyset pagination such as:

```sql
CREATE TABLE blobs (
  n SERIAL,
  value VARCHAR(255) NOT NULL CHECK (value <> ''),
  id bytea PRIMARY KEY
);

CREATE INDEX n_idx ON blobs USING btree (n);

SELECT id, value
FROM blobs
WHERE n > ?cursor
ORDER BY n ASC
LIMIT 2;
```

Size could be fixed if we don't need to provide this flexibility. Cursor could
be more obscure so users don't attempt to change the querystring by hand.

The value of using something like keyset pagination is to ensure pages are
always the same because the set has a forced order that doesn't depend on the
hash but an incremental number (could be a timestamp) so effectively only
half-empty pages would change when adding a new element.

---


```http
GET /meta/blobs/?size=2 HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Link: </meta/blobs/?cursor=2&size=2>; rel="next"

[
  {
    "id": "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5",
    "value": "\"country\""
  }, {
    "id": "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b",
    "value": "{\"cardinality\":\"1\",\"datatype\":\"string\"}"
  }
]
```

---

TODO: Blobs return their value stringified. What are the implications of
returning the value in JSON instead?

Should blobs be annotated by type?

```
type Blob
  = Value String -- Leaf like name, description or custodian
  | Field String
  | ... ?
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Link: </meta/blobs/?cursor=2&page-size=2>; rel="next"

[
  {
    "id": "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5",
    "type": "value",
    "value": "country"
  }, {
    "id": "d22869a1fd9fc929c2a07f476dd579af97691b2d0f4d231e8300e20c0326dd6b",
    "type": "field",
    "value": {"cardinality": "1", "datatype": "string"}
  }
]
```

---

### Get a blob by id

* Endpoint: `GET /meta/blobs/{id}`
* Parameters:
  * `id` (Required): The Blob hash.


```http
GET /meta/blobs/701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5 HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "701d021d08c54579f23343581e45b65ffb1150b2c99f94352fdac4b7036dbbd5",
  "value": "\"country\""
}
```

## Benefits

* It's not a breaking change.
  * Users consuming from `/records/` are not affected.
  * Users consuming from `/entries/` are not affected.
* Users interested in data changes don't need to care about metadata changes.
