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

This section describes the algorithm twice. First in a high level and then a
more step by step with intermediate values.

For reference, this is the abstract definition of an entry:

```elm
type Entry =
  { number : Integer
  , key : ID
  , timestamp : Timestamp
  , item : Set Hash
  }
```

### The algorithm

When this algorithm operates on _hashes_ (e.g. tag, concatenate) it is done on
the byte level, not the hexadecimal string representation that the latter
example shows as partial representations.

1. Let _number_ be the string representation of the entry number.
2. Let _key_ be the string representation of the entry key.
3. Let _timestamp_ be the string representation of the entry timestamp.
4. Let _items_ be the set of item hashes as bytes. Note that
   the item hashes must not have the prefix (i.e. `sha-256:`) when transformed to
   bytes.
5. Let _hashList_ be an empty list.
6. Apply the _hashValue_ function to _number_ tagged with `i` (Integer). And
   append the result to _hashList_.
7. Apply the _hashValue_ function to _key_ tagged with `u` (String). And
   append the result to _hashList_.
8. Apply the _hashValue_ function to _timestamp_ tagged with `t` (Timestamp). And
   append the result to _hashList_.
9. Map over each value _item_ in the _items_ set:
   1. Apply the _hashValue_ function to _item_ tagged with `r` (Hash).
10. Sort the elements of _items_, concatenate them and
11. Apply the _hashValue_ function tagged with `s` (Set). And append the result
    to _hashList_.
12. Concatenate the elements of _hashList_ in order (i.e. `[numberHash,
    keyHash, timestampHash, itemsHash]`), tag it with `l` (List) and hash the result.


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

Each value MUST be tagged by a byte that identifies the type, in a similar way
as the [objecthash](https://github.com/benlaurie/objecthash) algorithm.

```elm
type Tag
  = Hash
  | Integer
  | Set
  | String
  | Timestamp
  | List

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
    List ->
      'l'
```

```elm
hashValue : Alg -> String -> Tag -> Hash
```

Where:

* `Alg` is the hashing algorithm (e.g. `Sha256`),
* `String` is the string representation of the value to hash,
* `Tag` is the tag for the `Value` type and
* `Hash` is the list of bytes resulting of the hashing algorithm.

Following the JSON example from above, these are the hashes for each value:

```elm
numberValue = toString 6
numberValue == "6"

numberHash = hashValue Sha256 numberValue Integer
toHex numberHash == "396ee89382efc154e95d7875976cce373a797fe93687ca8a27589116644c4bcd"
```

```elm
timestampValue = toString (Timestamp 2016 4 5 13 23 5 Utc)
toHex timestampValue == "2016-04-05T13:23:05Z"

timestampHash = hashValue Sha256 timestampValue Timestamp
toHex timestampHash == "f22ecc4464f22c8fee624769189665a0afd7ef10a2775a000082c47cbd9f6419"
```

```elm
keyValue = toString (ID "GB")
keyValue == "GB"

keyHash = hashValue Sha256 keyValue String
toHex keyHash == "fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21"
```

The item is a special case:

It is a set of values and each value is already hashed. This means that the
content of this set only needs a sorted concatenation. Then, tag it as a set
`s` and hash the result.

```elm
hashValueSet : Alg -> Set Hash -> Tag -> Hash

itemHash = hashValueSet Sha256 (Set [Sha256 "6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]) Hash
toHex itemHash == "94b34bd5705b1a8c31796c00347dad2f5a1364050a1f6b768c7ac289502f62ce"
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

And finally, tag and hash the result of concatenating the list of hashes:

```elm
hash : Alg -> List Hash -> Hash

entryHash = hash Sha256 hashList
entryHash == "cc33f805025fb5bcd43cd9a1d0d77093af443106ff1513a2f5dfff2b81ae4fd3"
```
