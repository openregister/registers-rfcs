---
rfc:
start_date: 2018-08-02
pr:
status: draft
---

# Entry hash

## Summary

This RFC proposes an algorithm to hash entries without relying in JSON or in
the attribute names.

**This RFC is incompatible with the reference implementation**

## Motivation

The current specfication doesn't define what is the algorithm to get from the
entry data to the entry hash for the given hashing algorithm (from now on
SHA-256). The reference implementation (ORJ) reuses the item canonicalisation
algorithm to canonicalise the entry JSON representation and then hash the
result. Finally, relying on the JSON representation has the downside of being
more rigid than it could be.

## Explanation

The algorithm has three parts:

1. tag and hash each value in the entry.
2. order the resulting hashes.
3. hash the result of concatenating all hashes.

For reference, this is the abstract definition of an entry:

```elm
type Entry =
  { number: Integer
  , key: ID
  , timestamp: Timestamp
  , item: List Hash
  }
```

And this is the current serialisation in JSON:

```json
[
  {
    "index-entry-number":"6",
    "entry-number":"6",
    "entry-timestamp":"2016-04-05T13:23:05Z",
    "key":"GB",
    "item-hash":["sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]
  }
]
```

Notice they are not exaclty the same even thought they represent the same
entity. Also notice that the `ID` type is not yet an approved RFC. For this
RFC this shouldn't be a problem as it works the same as if the type were a
string.

### Tag and hash each value in the entry

Each value MUST be tagged by a byte that identifies the type, in a similar way
as the [objecthash](https://github.com/benlaurie/objecthash) algorithm.

Tags:

* `i`: Integer.
* `s`: String (e.g. `ID` type).
* `t`: Timestamp.
* `h`: Hash.

```elm
hashValue : Alg -> Value -> Tag -> Hash
```

Where `Alg` is the hashing algorithm (e.g. SHA-256), `Value` is the string
representation of the value to hash, `Tag` is the tag for the `Value` type and
`Hash` the hexadecimal _lowercase_ string representation of the hashed bytes.

Following the JSON example from above, these are the hashes for each value:

```elm
numberHash = hashValue Sha256 "6" Integer
numberHash == "396ee89382efc154e95d7875976cce373a797fe93687ca8a27589116644c4bcd"


timestampHash = hashValue Sha256 "2016-04-05T13:23:05Z" Timestamp
timestampHash == "f22ecc4464f22c8fee624769189665a0afd7ef10a2775a000082c47cbd9f6419"

keyHash = hashValue Sha256 "GB" String
keyHash == "c8fe89332c2aa2b92234bd43d0fb327c64fdbdce4ea527f51f78111543dd5341"
```

The item is a special case:

* It is an array of values.
* Each value is already a hash.
* And the hashes are prefixed.

The algorithm:

1. Foreach value, strip out the prefix.
2. Concatenate all hashes in strict order.
3. Hash the result.

```elm
hashValueList : Alg -> List Value -> Tag -> Hash

itemHash  = hashValueList Sha256 ["sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"] Hash
itemHash == "2d44caa5719d20a648aff8de575aa4832209ae0274f66edf5c33bcebceefef9a"
```

### order the resulting hashes

The order is:

```elm
hashList =
  [ numberHash
  , keyHash
  , timestampHash
  , itemHash
  ]
```

Note: The order has been chosen arbitrarily.

### hash the result of concatenating all hashes

And finally, hash the result of concatenating the list of hashes:

```elm
hash : Alg -> List Hash -> Hash

entryHash = hash Sha256 hashList
entryHash == "b4d13b604e67209e5a2a50da1bd37bfdb848332b28be3deff2276ec8cb166f94"
```
