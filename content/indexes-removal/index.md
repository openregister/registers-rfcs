---
rfc: 0024
start_date: 2018-10-02
decision_date: 2018-10-25
pr: openregister/registers-rfcs#41
status: approved
---

# Indexes removal

## Summary

This RFC proposes the changes to make to the data model and REST API
consequence of removing the general concept of indexes.

This RFC is a breaking change.

This RFC supersedes RFC0009 (Entry hash) due the changes on the data model
that affect entries.

For reference,
[ADR007](https://github.com/openregister/openregister-java/blob/master/doc/arch/adr-007-read-api-changes.md)
is the document that introduced the indexes changes.

## Motivation

After a period of assessment the Registers team concluded that indexes should
not be approached the way they are. As a consequence of this decision, we have
the opportunity to simplify the data model and the REST API, mostly roll back
to the state it was before the indexes work was introduced.

## Explanation

### Data model

The main change on the data model is the entry. It changes from:

```elm
type Entry =
  { number: Integer
  , timestamp: Timestamp
  , key: ID
  , item: Set Hash
  }
```

To:

```elm
type Entry =
  { number: Integer
  , timestamp: Timestamp
  , key: ID
  , blob: Hash
  }
```

So, an entry must always have a single blob reference (hash). This requires
changing the entry hash (RFC0009) algorithm as follows:

1. Let _hashList_ be an empty list.
2. Let _number_ be the string representation of the entry number.
3. Let _key_ be the string representation of the entry key.
4. Let _timestamp_ be the string representation of the entry timestamp.
5. Let _blob_ be the list of bytes of the blob hash.
6. Apply the _hashValue_ function to _number_ tagged with `0x69` (Integer). And
   append the result to _hashList_.
7. Apply the _hashValue_ function to _key_ tagged with `0x75` (String). And
   append the result to _hashList_.
8. Apply the _hashValue_ function to _timestamp_ tagged with `0x74` (Timestamp). And
   append the result to _hashList_.
8. Apply the _hashValue_ function to _blob_ tagged with `0x72` (Hash). And
   append the result to _hashList_.
13. Concatenate the elements of _hashList_ in order (i.e. `[numberHash,
    keyHash, timestampHash, blobHash]`), tag it with `0x6C` (List) and hash the
    result.


### REST API

Note that all explanations assume JSON. Translation to CSV should be
straightforward with the current JSON to CSV rules.

#### Entries

##### Get entry by number

```
GET /entries/{entry-number}
```

Summary:

* Returns an object _instead_ of an array of objects.
* Change `item-hash` (List of Hash) to `blob-hash` (Hash).
* `index-entry-number` gets removed.

Parameters:

|Name|Type|Description|
|-|-|-|
|`entry-number`| Integer|The position of the entry in the log.|

Response attributes:

|Name|Type|Description|
|-|-|-|
|`entry-number`|Integer|The entry number.|
|`entry-timestamp`| Timestamp|The entry timestamp.|
|`key`| ID|The entry key.|
|`blob-hash`| Hash|The blob hash with the data for the entry.|

##### List entries

Same changes that apply to the entry representation:

* Change `item-hash` (List of Hash) to `blob-hash` (Hash).
* `index-entry-number` gets removed.


#### Records

Note that the anomaly of records being entries but not treated as such will be
addressed in another RFC.

##### Get record by key

```
GET /records/{key}
```

Summary:

* Returns an object with the record rather than an object with
  the key and the record as its value.
* Change `item` (List of Blob) to `blob` (Blob).
* `index-entry-number` gets removed.

Parameters:

|Name|Type|Description|
|-|-|-|
|`key`| [ID](/glossary/key#id-type)|The record identifier.|

Response attributes:

|Name|Type|Description|
|-|-|-|
|`entry-number`|Integer|The entry number.|
|`entry-timestamp`| Timestamp|The entry timestamp.|
|`key`| ID|The entry key.|
|`blob`| Blob|The data blob.|

##### List records

```
GET /records
```

Summary:

* Returns an array of record objects instead of an object with keys and
  the record as their value.
* All changes that apply to the record object defined above.

##### List the trail of change for a record

```
GET /records/{key}/entries
```

Summary:

* Any change defined for the list of entries.

### Archive

The archive, should reflect all changes defined above when applicable.

## Consequences

This is a breaking change as important as the one introduced by the ADR007.
