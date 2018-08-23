---
rfc:
start_date: 2018-08-03
pr:
status: draft
---

# Item hash with redaction

## Summary

This RFC proposes an algorithm to hash items such that data can be redacted
at any granularity without breaking the integrity of the register.

**This RFC is not backwards compatible**


## Motivation

The current item hashing algorithm makes impossible to redact a bit of data
inside the item. All you can do is to redact the full blob. This don't end
well when for example a GDPR request forces redacting a piece of information
replicated in many items. Which is likely to happen given that every time a
new item is introduced with a change, the rest of the data is copied.

## Explanation

The following explanation is an implementation of the [objecthash](https://github.com/benlaurie/objecthash)
algorithm.

Items are stored as a set of attribute-value pairs where values are strings.
Types and casting from the string representation are handled in a different
phase of the process. To align with this fact and to align with the features
provided by objecthash values are tagged according to two possible scenarios:
a string value and a set of string values (i.e. a cardinality n value).

```elm
type Tag
  = Dict     -- 'd'
  | String   -- 'u'
  | Set      -- 's'
```

Note that a Set is unordered at origin so order is applied as part of the
hashing process.


For reference, this is the abstract definition of an item:

```elm
type Value
  = VString String    -- cardinality 1
  | VSet (Set String) -- cardinality n

type Item =
  Dict String Value
```

Note: Attribute names should be of type `Fieldname` but it's not yet defined
in a RFC. In any case, `Fieldname` is a subset of `String` so this RFC is
intended to be forward compatible.

### Algorithm

When this algorithm operates on _hashes_ (e.g. tag, concatenate) it is done on
the byte level, not the hexadecimal string representation that the latter
example shows as partial representations.

1. Let _item_ be the blob of data to hash.
2. Let _hashList_ be an empty list.
2. Foreach _(attr, value)_ pair in _item_:
   1. If _value_ is null, continue.
   2. If _value_ is a Set:
      1. Let _elList_ be an empty list.
      2. Foreach _el_ in _value_:
         1. If _el_ starts with `**REDACTED**`, append _el_ without `**REDACTED**`
            to _elList_.
         2. Otherwise, normalise _el_ according to [string normalisation](#string-normalisation-algorithm)
            tag it with `u` (String), hash it and append it to _elList_.
      3. Concatenate _elList_ elements, tag it with `s` (Set), hash it and set
         it to _valueHash_.
   3. If _value_ starts with `**REDACTED**`, set _normValue_ with _value_
      without `**REDACTED**`.
   4. Otherwise, normalise _value_ according to [string normalisation](#string-normalisation-algorithm)
      tag it with `u` (String), hash it and set _valueHash_.
   5. Tag _attr_ with `u` (String), hash it and set _attrHash_.
   6. Concat _attrHash_ and _valueHash_ and append to _hashList_.
3. [Sort](#sorting) _hashList_.
4. Concat _hashList_ elements, tag with `d`, hash it and return.


Note: Any time the algorithm says "tag with" it means to prepend the byte
corresponding to the tag (e.g. 0x70) to the list of bytes of the thing to tag.


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


#### String normalisation algorithm

1. For control characters (codepoints 0x00 - 0x1f):
   1. If it has a short representation (`\b`, `\f`, `\n`, `\r`, or `\t`), that
      short representation MUST be used.
   2. Other control characters (such as `NULL`) MUST be represented as a
      `\u00XX` escape sequence. Hexadecimal digits MUST be upper-case.
2. Backslash (`\`) and double quote (`"`) MUST be escaped as `\\` and `\"`
   respectively.
3. All other characters MUST be included literally (i.e. unescaped). This
   includes forward-slash (`/`).


### Redaction

Redaction works in the same way as
[objecthash](https://github.com/benlaurie/objecthash/blob/d1e3d6079fc16f8f542183fb5b2fdc11d9f00866/README.md#L65).

In summary, these two items are equivalent:

```elm
i_0 =
  Dict [ ("foo", "abc")
       , ("bar", "xyz")
       ]

i_1 =
  Dict [ ("foo", "**REDACTED**2a42a9c91b74c0032f6b8000a2c9c5bcca5bb298f004e8eff533811004dea511")
       , ("bar", "xyz")
       ]
```

The reason they are equivalent is because the hash for `"abc"` is
"2a42a9c91b74c0032f6b8000a2c9c5bcca5bb298f004e8eff533811004dea511". So when
the hashing algorithm is applied to the whole item, the redacted value is used
as is (without the redaction tag).

### Example

Note: To simplify the code, the hashing algorithm `SHA-256` is implied. A full
implementation must acknowledge the possibility of a different algorithm.

To walk through the algorithm I'll use the following item:

```elm
Dict
  [ ("id", "GB")
  , ("official-name", "The United Kingdom of Great Britain and Northern Ireland")
  , ("name", "United Kingdom")
  , ("citizen-names", Set ["Briton", "British citizen"])
  ]
```

Which is serialised as JSON as:

```json
{
  "id": "GB",
  "official-name": "The United Kingdom of Great Britain and Northern Ireland",
  "name": "United Kingdom",
  "citizen-names": ["Briton", "British citizen"]
}
```

The following definitions set the context of operation:

```elm
type Tag
  = Dict
  | Set
  | String

tagToChar : Tag -> Char
tagToChar tag =
  case tag of
    Dict ->
      'd'
    Set ->
      's'
    String ->
      'u'
```

```elm
hashTagged : String -> Hash

hashString : String -> Hash

hashSet : Set -> Hash

hashValue : Value -> Hash

hashPair : (String, Value) -> List Hash
```

```elm
hash : Item -> Hash
```

The `hash` function, step by step would roughly look like:

```elm
id = hashPair ("id", "GB")
id == "17b788a70eeccbdc2fcb2d2d3db216c02fa88ac668beeb164bb2328c864bf3f4fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21"

officialName = hashPair ("official-name", "The United Kingdom of Great Britain and Northern Ireland")
officialName == "cf09bea8c0107bd2150b073150d48db0a5b24c83defc7960ed698378d9f84b93bf1860175c77869938cf9f4b37edb00f2f387be7b361f9c2c4a2ac202c1ba2e5"

name = hashPair ("name", "United Kingdom")
name == "5c0be87ed7434d69005f8bbd84cad8ae6abfd49121b4aaeeb4c1f4a2e298771194099b1e0b9a1e673bafee513080197fa1980895ca27e091fdd4c54fab2bed24"

citizenNames = hashPair ("citizen-names", Set ["Briton", "British citizen"])
citizenNames == "bb3a7ac86d4f90c20d099992de0bd09bf3c4f27169c2cd873836762b01d5a2be16897987a6ee59d9ffdb456ed02df34a79b05346498d4360172568101ae157c1"
```

Then combine the partial results and tag it as a dictionary:

```elm
itemHash = hashDict [id, officialName, name, citizenNames]

itemHash == "45d9392ad17cead3fa46501eba3e5ac237cb46a39f1e175905f00ef6a6667257"
```

### Redaction

Now, let's redact part of the original item. From:


```elm
Dict
  [ ("id", "GB")
  , ("official-name", "The United Kingdom of Great Britain and Northern Ireland")
  , ("name", "United Kingdom")
  , ("citizen-names", Set ["Briton", "British citizen"])
  ]
```

To:

```elm
Dict
  [ ("id", "GB")
  , ("official-name", "**REDACTED**cf09bea8c0107bd2150b073150d48db0a5b24c83defc7960ed698378d9f84b93bf1860175c77869938cf9f4b37edb00f2f387be7b361f9c2c4a2ac202c1ba2e5")
  , ("name", "United Kingdom")
  , ("citizen-names", Set ["Briton", "British citizen"])
  ]
```

The resulting hash would be the same "45d9392ad17cead3fa46501eba3e5ac237cb46a39f1e175905f00ef6a6667257".

### Security considerations

This algorithm is as secure as the previous one.
