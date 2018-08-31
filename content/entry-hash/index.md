---
rfc: 0009
start_date: 2018-08-02
decision_date: 2018-08-30
pr: openregister/registers-rfcs#23
status: approved
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
9. Let _itemsBytes_ be an empty set.
10. For each value _item_ in the _items_ set:
   1. Apply the _hashValue_ function to _item_ tagged with `r` (Hash) and
      append the result to _itemsBytes_.
11. [Sort the elements](#sorting) of _itemsBytes_, concatenate them and
12. Apply the _hashValue_ function tagged with `s` (Set). And append the result
    to _hashList_.
13. Concatenate the elements of _hashList_ in order (i.e. `[numberHash,
    keyHash, timestampHash, itemsHash]`), tag it with `l` (List) and hash the result.

#### Sorting

The sorting algorithm for a set of hashes is done by comparing the list of
bytes one by one. For example, given a set `["foo", "bar"]` you'll get the
folllowing byte lists after hashing them as unicode:

```elm
[ [166,166,229,231,131,195,99,205,149,105,62,193,137,194,104,35,21,217,86,134,147,151,115,134,121,181,99,5,242,9,80,56]
, [227,3,206,11,208,244,193,253,254,76,193,232,55,215,57,18,65,226,224,71,223,16,250,97,1,115,61,193,32,103,93,254]
]
```

In this case, the set is already sorted given that `166` is smaller than
`227`.

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

Notice they are not exaclty the same even though they represent the same
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

To keep the explanation simpler `hashValue` assumes SHA-256 as the hashing
algorithm.

```elm
hashValue : Tag -> String -> Hash
```

Where:

* `Tag` is the tag for the `Value` type and
* `String` is the string representation of the value to hash,
* `Hash` is the list of bytes resulting of the hashing algorithm.

Following the JSON example from above, these are the hashes for each value:

```elm
numberValue = toString 6
numberValue == "6"

numberHash = hashValue Integer numberValue
toHex numberHash == "396ee89382efc154e95d7875976cce373a797fe93687ca8a27589116644c4bcd"
```

```elm
timestampValue = toString (Timestamp 2016 4 5 13 23 5 Utc)
toHex timestampValue == "2016-04-05T13:23:05Z"

timestampHash = hashValue Timestamp timestampValue
toHex timestampHash == "f22ecc4464f22c8fee624769189665a0afd7ef10a2775a000082c47cbd9f6419"
```

```elm
keyValue = toString (ID "GB")
keyValue == "GB"

keyHash = hashValue String keyValue
toHex keyHash == "fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21"
```

The item is a set of values rather than a leaf value and each value is already
hashed. This means that each element of the set needs to be tagged with `r`
(Hash) and hashed again.

```elm
itemsBytes = map (hashValue Hash) (Set [Sha256 "6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"])
```

Then sort the set concatenate and tag with `s` (Set) and hash the result.

```elm
itemHash = hashValue Set (concat (sort itemsBytes))
toHex itemHash == "cff910f74878650a3cceb54039bdb62707de9d20e80d4385127732a4e444bd57"
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
hash : List Hash -> Hash

entryHash = hash hashList
entryHash == "51a02cd5692c6a03ba78330cb68f8e26e976c5933af0aa8d779589a1e6264e4b"
```
