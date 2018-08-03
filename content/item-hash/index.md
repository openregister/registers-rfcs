---
rfc:
start_date: 2018-08-03
pr:
status: draft
---

# Item hash revisit

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

1. For ASCII control characters (codepoints 0x00 - 0x1f):
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

Which in JSON are represented as:

```json
{
  "foo": "abc",
  "bar": "xyz"
}
```

And

```json
{
  "foo": "**REDACTED**2a42a9c91b74c0032f6b8000a2c9c5bcca5bb298f004e8eff533811004dea511",
  "bar": "xyz"
}
```

### Example

To walk through the algorithm I'll use the following item:

```elm
Dict
  [ ("id", "GB")
  , ("official-name", "The United Kingdom of Great Britain and Northern Ireland")
  , ("name", "United Kingdom")
  , ("citizen-names", ["Briton", "British citizen"])
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
