---
rfc: 0020
start_date: 2018-08-30
pr: openregisters/registers-rfc#33
status: draft
---

# Blob normalisation

## Summary

This RFC proposes an algorithm to normalise blobs of data such that the
hashing algorithm (RFC0010) is predictable. This RFC replaces the item
canonicalisation obsoleted by RFC0010.

Note that throughout this RFC _blob_ and _item_ are used interchangeably.


## Motivation

The current specification _suggests_ how null values and empty strings should
be treated in terms of normalisation but it is not clear so it is open for
interpretation.

## Explanation

Blobs of data must have a normal form such that algorithms processing them
(e.g. hashing algorithm (RFC0010)) have a consistent output. Blobs of data are
attribute-value pair associative arrays where values are Strings but nulls
have to be addressed with care.

The following algorithm ensures all nullable values are removed from the
normalised representation:

1. Let _blob_ be the blob to normalise.
2. Let _result_ be an empty dictionary.
3. Foreach _(attr, value)_ pair in _blob_:
   1. If _value_ is null, continue.
   2. If _value_ is an empty String, continue.
   3. If _value_ is an empty Set, continue.
   4. If _value_ is a Set:
      1. Let _normSet_ be an empty Set.
      2. Foreach _el_ in _value_:
         1. If _el_ is null, continue.
         2. If _el_ is an empty String, continue.
         3. Otherwise, [normalise](#string-normalisation) _el_ and append the
            result to to _normSet_.
      3. If _normSet_ is empty, continue.
      4. Otherwise, set _(attr, normSet)_ to _result_.
   5. Otherwise,
      1. Let _normValue_ be null.
      2. [Normalise](#string-normalisation) _value_ and set _normValue_.
      3. Set _(attr, normValue)_ to _result_.


In summary, any value that is an empty String, empty Set or Set with
empty strings in it is normalised as a null value and removed from the
normalised result.

Note that this normalisation allows parity between flexible formats like JSON
and rigid formats like CSV. For example, in CSV having an empty field, emtpy
string, would be normalised as null.

### String normalisation

A string should be in the NFC form as defined by the Unicode standard:
https://en.wikipedia.org/wiki/Unicode_equivalence


### Security considerations

There are no known security implications arising from this proposal.
