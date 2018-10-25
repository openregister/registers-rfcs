---
rfc: 0018
start_date: 2018-08-13
pr: openregister/registers-rfcs#31
status: draft
---

# Context resource

## Summary

This RFC proposes a context resource to surface the current metadata of the
register.

Dependencies:

* [RFC0013: Multihash](../multihash/index.md)


## Motivation

Currently, the register metadata is not exposed consistently. Some is exposed
via the register resource, some via the field register and some from the
register register.

Also, the current specification and implementation don't agree on what is the
register resource.

The main driver for this RFC is to have a computed resource for the latest
metadata in a way that makes the register self contained (i.e. a user should
not need to interact with any other register to get the metadata for a
register).


## Explanation

The **context** is the metadata snapshot that apply to a given log size.

### Definition

```elm
type Context =
  { id : Name
  , copyright : Maybe String
  , custodian : Maybe String
  , description : Maybe String
  , hashingAlgorithm : HashingAlgorithm
  , licence : Maybe String
  , rootHash : Hash
  , schema : Schema
  , statistics : Statistics
  , status : Status
  , title : Maybe String
  }
```

See each section below of the explanation for each attribute.


#### Id

* Type: Name (This is defined in the specification as `[a-z][a-z0-9-]*`, the
  future specification will make it more clear).

The register identifier. The current implications for this value make it act
as a unique identifier for the environment it belongs to. This RFC doesn't
pretend to change this fact.


#### Copyright

* Type: Optional String

The register copyright. E.g. `© Crown copyright`. Similar to the `copyright`
field in the Register register. Must be present for any published register.


#### Custodian

* Type: Optional String

The same value currently exposed in the Register resource. Must be present for
any published register.


#### Description

* Type: Optional String

The human readable description of the register.


#### Hashing algorithm

* Type: HashingAlgorithm

The hashing algorithm used throughout the register.

```elm
type HashingAlgorithm =
  { id : Name
  , functionType : UVarint
  , digestLength : Int
  }
```

This depends on RFC0013

|Name|Description|
|-|-|
|`digest-length`|The byte length of the digest. E.g. `0x12`|
|`function-type`|The identifier of the hash function. E.g. `0x20`, `0xb220`|
|`id`| The name of the algorithm. E.g. `sha2-256`|

See also: [Multihash table function type
identifiers](https://github.com/multiformats/multihash/blob/master/hashtable.csv).


#### Licence

* Type: Optional String

The licence that applies to the data. Must be present for any published register.

This could get restricted with a suitable list of licences and validated with
an external tool such that the technology allows free text but each register
controller is able to ensure what is allowed.


#### Root hash

* Type: Hash

The root hash for the register. Note that signatures will be addressed in
another RFC. This is not strictly part of the metadata, it is derived from the
data.

#### Schema

* Type: Schema

The set of attributes that define the data allowed in the register.

```elm
type Attribute =
  { id : Name
  , datatype : Datatype
  , title : Maybe String
  , description : Maybe Text
  }

type Schema =
  Set Attribute
```

|Name|Description|
|-|-|
|Id|Attribute identifier (e.g. `start-date`)|
|Datatype|Datatype identifier (e.g. `datetime`)|
|Title|Human readable attribute name (e.g. `Start date`)|
|Description|Human readable attribute description (e.g. `The date a record first became relevant to a register.`)|


#### Statistics

* Type: Statistics

The summary of objects stored in the register. This overlaps with the Register
resource and it is not strictly part of the metadata, it is derived from the data.


#### Status

* Type: Status (Active, Retired). Defaults to Active.

The status of the register. Either active or retired. This addresses the
problem of being able to retire an entire register (see
[issue #17](https://github.com/openregister/registers-rfcs/issues/17)).

```elm
type Status
  = Active { startDate : Timestamp }
  | Retired { startDate : Timestamp
            , endDate : Timestamp
            , replacement : Maybe Url
            , reason : Text
            }
```

|Name|Description|
|-|-|
|Start date| Date when the register started.|
|End date| Date when the register was retired.|
|Replacement| A URL to a register that replaces the current one.|
|Reason| A human readable explanation of why the register is no longer active.|


#### Title

* Type: Optional String

The human readable name of the register.


### HTTP resource

***
#### Endpoint

```
GET /context
```

#### Parameters

|Name|Type|Description|
|-|-|-|
|`total-entries` | Optional `Integer`| The log size to compute the snapshot from. Note this only applies when the metadata can be versioned.|


#### Response attributes

|Name|Type|
|-|-|
|`id`| Name |
|`copyright`| Optional String |
|`custodian`| Optional String |
|`description`| Optional String |
|`hashing-algorithm`| HashingAlgorithm |
|`licence`| Optional String |
|`root-hash`| Hash |
|`schema`| List Attribute |
|`statistics`| Statistics |
|`status`| Status |
|`title`| Optional String |

#### Hashing algorithm attributes

|Name|Type|
|-|-|
|`digest-length`| Integer |
|`function-type`| Integer |
|`id`| Name |

See also: [Multihash table function type
identifiers](https://github.com/multiformats/multihash/blob/master/hashtable.csv).

#### Attribute attributes

|Name|Type|
|-|-|
|`id`| Name |
|`datatype`| Datatype |
|`cardinality`| Cardinality |
|`title`| Optional String |
|`description`| Optional String |

#### Statistics attributes

|Name|Type|
|-|-|
|`total-entries`| Integer |
|`total-items`| Integer |
|`total-records`| Integer |

#### Status attributes

|Name|Type|
|-|-|
|`start-date`| Datetime |
|`end-date`| Optional Datetime |
|`replacement`| Optional Url |
|`reason`| Optional String |

***

***
**EXAMPLE:**


```http
GET /context HTTP/1.1
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "multihash",
  "title": "The Multihash register",
  "description": "List of multihash codes.",
  "custodian": "IPFS team",
  "hashing-algorithm": {
    "id": "sha2-256",
    "function-type": 18,
    "digest-length": 32
  },
  "statistics": {
    "total-entries": 0,
    "total-items": 0,
    "total-records": 0,
  },
  "copyright": "Copyright (c) 2016 Protocol Labs Inc.",
  "licence": "MIT",
  "root-hash": "1220e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "status": { "start-date": "2018-12" },
  "schema": [
    {"id": "name", "datatype": "name", "cardinality": "1"},
    {"id": "function-type", "datatype": "integer", "cardinality": "1"},
    {"id": "digest-length", "datatype": "integer", "cardinality": "1"}
  ]
}
```

***

### Considerations

The context resource makes the register resource obsolete. Given that users
depend on it the register resource will be available for as long as necessary.

Once the context resource is available, the register resource should use a
warning HTTP header and a Link alternate header.
