---
rfc:
start_date: 2018-08-03
pr:
status: draft
---

# Item hash with redaction

## Summary

This RFC proposes an algorithm to hash items such that data can be redacted
without breaking the integrity of the register.

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
3. Sort _hashList_.
4. Concat _hashList_ elements, tag with `d`, hash it and return.


### String normalisation algorithm

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
hash : Item -> Hash
```

```elm
hashString : String -> Hash

hashSet : Set -> Hash

hashValue : Value -> Hash
hashValue value =
  case value of
    VString str ->
      if startsWith "**REDACTED**" str then
        hashString (dropLeft 12 str)
      else
        hashString str
    VSet set ->
      hashSet set
```

The `hash` function, step by step would roughly look like:

```elm
id =
  concat ["17b788a70eeccbdc2fcb2d2d3db216c02fa88ac668beeb164bb2328c864bf3f4", "fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21"]

id == "d62311d17806761e4195bda80a4ba7f20c8cacc58c3f6f0f1a22e64f47a77651"

officialName =
  concat ["cf09bea8c0107bd2150b073150d48db0a5b24c83defc7960ed698378d9f84b93", "bf1860175c77869938cf9f4b37edb00f2f387be7b361f9c2c4a2ac202c1ba2e5"]

officialName == "103ea25f856f755082745f183b1f6f41c08b1c35df794d1382650637f35d0949"

name =
  concat ["5c0be87ed7434d69005f8bbd84cad8ae6abfd49121b4aaeeb4c1f4a2e2987711", "94099b1e0b9a1e673bafee513080197fa1980895ca27e091fdd4c54fab2bed24"]

name == "df50f7bd9e6af8c5dd009f99e66a0192ca6204c4b7e45ee79e4a89d1d7ace200"

citizenNameSet =
  ["3d76c67f95cb9c4fc8e9dfdaa1d0ac4cbf6feba4dc7521429618afad925a3922", "f59ca1c4c086398b98aa6ad3ab2b14c4c1a15c225da73c706f5023115a7acd50"]
  |> concat
  |> cons 's'
  |> hashTagged

citizenNameSet == "1b68822ac12017ae10eebcce34c4cd5e07d83b6c76bdca8f14eb54ab60096269"

citizenNames =
  concat ["bb3a7ac86d4f90c20d099992de0bd09bf3c4f27169c2cd873836762b01d5a2be", "1b68822ac12017ae10eebcce34c4cd5e07d83b6c76bdca8f14eb54ab60096269"]

citizenNames == "c8a2625212e2bac74346c60ae4988aa9978958918bd1455e35e6d0670af15ff3"
```

Then sort and aggregate the partial results:

```elm
itemHash =
  [id, officialName, name, citizenNames]
  |> sort
  |> concat
  |> cons 'd'
  |> hashTagged

itemHash == "5bc0163d594fb6e958d2758eff074fb4d25cd3f3867ff30e9cbe982c59cb90b5"
```
