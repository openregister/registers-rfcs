---
rfc:
start_date: 2018-08-02
pr:
status: draft
---

# Entry hash

## Summary

This RFC proposes an algorithm to hash entries with a well defined set of
attributes, and a way to hash from the values to the whole entry.

**This RFC is incompatible with the reference implementation**

## Motivation

The current specfication doesn't define what is the algorithm to get from the
entry data to the entry hash for the given hashing algorithm (from now on
SHA-256).

The reference implementation (ORJ) reuses the item canonicalisation
algorithm to canonicalise the entry JSON representation and then hash the
result. This means that any attribute currently part of the reference
implementation `Entry` entity is serialised and used to compute the hash. As
an example, the index entry number, part of the experimental work on Indexes
has affected the result of the entry hash even though it is (a) not stable and
(b) not part of the real identity of an entry.

Finally, the current implementation relies on JSON and on the attribute names
serialised as strings.

This RFC aims to provide an algorithm that doesn't rely on specific attribute
names (e.g. `entry-number` vs `number`, `item-hash` vs `itemHash`, etc) which
supports a better decoupling between the data model and the REST API (we don't
have plans for it but a hypothetical GraphQL interface wouldn't work with the
current attribute names and yet conceptually an entry is the same).

Note that the item hash algorithm is not affected by this RFC.

## Explanation

The algorithm has three parts:

1. tag and hash each value in the entry.
2. order the resulting hashes.
3. hash the result of concatenating all hashes.

For reference, this is the abstract definition of an entry:

```elm
type Entry =
  { number : Integer
  , key : ID
  , timestamp : Timestamp
  , item : Set Hash
  }
```

To walk through the algorithm I'll use the following entry:

```elm
Entry
  { number = 6
  , key = ID "GB"
  , timestamp = Timestamp 2016 4 5 13 23 5 Utc
  , item = Set [Sha256 "6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]
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

```elm
type Tag
  | Hash
  | Integer
  | Set
  | String
  | Timestamp

tagToChar : Tag -> Char
tagToChar tag =
  case tag of
    Hash ->
      'r'
    Integer ->
      'i'
    Set ->
      's'
    String ->
      'u'
    Timestamp ->
      't'
```

```elm
hashValue : Alg -> String -> Tag -> Hash
```

Where:

* `Alg` is the hashing algorithm (e.g. `Sha256`),
* `String` is the string representation of the value to hash,
* `Tag` is the tag for the `Value` type and
* `Hash` is the hexadecimal _lowercase_ string representation of the hashed bytes.

Following the JSON example from above, these are the hashes for each value:

```elm
numberValue = toString 6
numberValue == "6"

numberHash = hashValue Sha256 numberValue Integer
numberHash == "396ee89382efc154e95d7875976cce373a797fe93687ca8a27589116644c4bcd"
```

```elm
timestampValue = toString (Timestamp 2016 4 5 13 23 5 Utc)
timestampValue == "2016-04-05T13:23:05Z"

timestampHash = hashValue Sha256 timestampValue Timestamp
timestampHash == "f22ecc4464f22c8fee624769189665a0afd7ef10a2775a000082c47cbd9f6419"
```

```elm
keyValue = toString (ID "GB")
keyValue == "GB"

keyHash = hashValue Sha256 keyValue String
keyHash == "fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21"
```

The item is a special case:

* It is an array of values.
* Each value is already a hash. Note that if your implementation has the
  prefix `sha-256:` it has to be removed before hasing again.

The algorithm:

1. Foreach value, strip out the prefix.
2. Tag and Hash each value.
3. Concatenate all hashes in the same order.
4. Tag and hash the result.

```elm
hashValueSet : Alg -> Set Hash -> Tag -> Hash

itemHash = hashValueSet Sha256 (Set [Sha256 "6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]) Hash
itemHash == "94923d31e0ee69e40c912b3e0a34b1693d0d23d7049b2a4510c4b33f63850297"
```

These are the steps for the example above:

1. Transform hashes to strings:

```elm
values = map toString (Set [Sha256 "6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"])
values == Set ["6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]
```

2. Tag and hash:

```elm
hashedSet =
  ["6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]
  |> map hashSetValue
  |> sort
  where
    hashSetValue e = hashValue Sha256 e Hash

hashedSet == Set ["63825e4c6f3dd8b229be451b17f57a4e431eb92ae62f9581c9bd268558c27177"]
```

3. Concatenate:

```elm
concatHash = concatAll (Set ["63825e4c6f3dd8b229be451b17f57a4e431eb92ae62f9581c9bd268558c27177"])
concatHash == "63825e4c6f3dd8b229be451b17f57a4e431eb92ae62f9581c9bd268558c27177"
```

4. Tag and hash:

```
itemHash = hashValue Sha256 concatHash Set
itemHash == "94923d31e0ee69e40c912b3e0a34b1693d0d23d7049b2a4510c4b33f63850297"
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
entryHash == "5317fb0c7191e77b9a355bf947e8a2a619121fff65de183709099065b7e143ca"
```
