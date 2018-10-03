---
rfc: 0025
start_date: 2018-10-03
decision_date:
pr: openregister/registers-rfcs#42
status: draft
---

# Snapshot resource

## Summary

This RFC proposes creating a new Snapshot resource and redefine the Record
resource to have a single responsibility.

This is a breaking change.

## Motivation

One of the most difficult concepts to explain in the Registers ecosystem is
the concept of “record”. The reasons boil down to two things: First, the
concept of “record” has shifted over time as a natural adaptation to how
Registers has evolved and second the current record definition doesn't match
the expectations on the record resource. Probably the second is consequence of
the first.

Let me elaborate on what does “the current record definition doesn't match
the expectations on the record resource” mean.

The current definition of record is: The most recent entry for a given key.
The record resource though is the combination of the most recent entry for a
given key and the blob of data referenced by it.

The reason for having the blob of data merged in is to make the record
resource useful to the type of user that wants to get the data for a given
key.

The reason for having the entry data is to provide the context for how it
relates to the log of entries. Useful to users that need to deal with
entries, blobs and the log itself.

The first reason is “porcelain” in the sense of “don't make me think, give me
the data”. The second is “plumbing” in the sense of “let me handle the low
level bits myself”.

The result is tension between these two needs and an ill-defined concept.
The proposal is to split both needs into dedicated resources.

## Explanation

Let's define “snapshot” to be the set of most recent entries for each unique key.

```elm
type Snapshot =
  Set Entry
```

The operation to compute the snapshot for a given log can be defined as:

```elm
collect : Log -> Snapshot
```

***
**EXAMPLE:**

For example, given log `log` of size 10, we can compute the latest snapshot
with:

```elm
collect log
```

And compute the snapshot at size 5 by slicing the log to that size:

```elm
collect (take 5 log)
```
***

Let's redefine “record” as the combination of the data with the entry key.

```elm
type Record =
  { key : ID
  , data : Dict Name Value
  }
```

Note that the attribute `data` holds the same type of data structure as the
blob. The resason for not using the blob here is because this opens the door
to return data that is not a perfect match with the blob. For example, the RFC
draft for removing fields would benefit from this flexibility.

Note that `Value` is either a string or a set of strings, the same as the blob
definition.

The operation to compute the record for a given entry can be defined as:

```elm
view : BlobStore -> Entry -> Result Error Record
```

Where `BlobStore` is the complete set of known blobs. Given that the blob
store may not contain the blob referenced by the entry, this computation can
fail with an error.

***
**EXAMPLE:**

For example, to compute the latest set of records for a given log:

```elm
map (view store) (collect log)
```

So, first we compute the snapshot for the full log and then we transform each
entry into a record.
***

### REST API

#### Get a snapshot

Returns an array of entry resources.

This resource MAY be paginated like the rest of the collections.

***
#### Endpoint

```
GET /snapshots/{log-size}
```

#### Parameters

|Name|Type|Description|
|-|-|-|
|`log-size`|Integer|The log size to compute the snapshot.|

Note: We can consider some kind of alias for the latest log size like the
`latest` tag used by Docker.
***


#### Get a snapshot entry by key

Returns an entry resource.

***
#### Endpoint

```
GET /snapshots/{log-size}/{key}
```

#### Parameters

|Name|Type|Description|
|-|-|-|
|`log-size`|Integer|The log size to compute the snapshot.|
|`key`|ID|The unique identifier for an element of the register.|
***

#### Get a record by id

Returns a record resource.

***
#### Endpoint

```
GET /records/{id}
```

#### Parameters

|Name|Type|Description|
|-|-|-|
|`id`|ID|The unique identifier for an element of the register.|
***

##### JSON serialisation

The record resource MUST be a JSON object with the data for the record. It
MUST also have the key of the entry using the reserved attribute `_id`. Given
that an attribute name is of type `Name`, it is not possible to have a clash
with `_id`.

***
**EXAMPLE:**

For example, the most recent record for the contry identified by `GB`:

```http
GET /records/GB HTTP/1.1
Accept: application/json
Host: country.register.gov.uk
```

```http
{
  "_id": "GB",
  "name": "United Kingdom",
  "official-name": "The United Kingdom of Great Britain and Northern Ireland",
  "citizen-names": ["Briton", "British citizen"]
}
```
***

##### CSV serialisation

The record resource MUST be a CSV row with the data for the record. It
MUST also have the key of the entry using the reserved attribute `_id`. Given
that an attribute name is of type `Name`, it is not possible to have a clash
with `_id`.

***
**EXAMPLE:**

For example, the most recent record for the contry identified by `GB`:

```http
GET /records/GB HTTP/1.1
Accept: text/csv
Host: country.register.gov.uk
```

```http
_id, name, official-name, citizen-names
GB, "United Kingdom", "The United Kingdom of Great Britain and Northern Ireland", "Briton;British citizen"
```
***

#### List records

Returns an array of record resources.

***
##### Endpoint

```
GET /records
```

### Considerations

This proposal doesn't offer a way to return records for a snapshot other than
the latest one. It could be easily extended with a querystring `{?log-size}`
if there is need for it.

## Consequences

This is a breaking change.
