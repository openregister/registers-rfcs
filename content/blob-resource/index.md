---
rfc: 0016
start_date: 2018-08-22
decision_date: 2018-09-28
pr: openregister/registers-rfcs#29
status: approved
---

# Blob resource

## Summary

This RFC proposes changing the name “item” to “blob” and reflect this renaming
everywhere in the REST API.

## Motivation

Casual observation says that the name “item” is one of the hard concepts to
grasp in the registers data model. My hypothesis is that the common use of
“item” in English, refers to things part of a list and that overlaps with other
terms in registers (e.g. “record”, “entry”).

The informal definition of a register is that it is “a list of things of the
same type”. And then you need to know that it is a list of “records” not a
list of “items”.

The idea is to change the name “item” to another term that at least doesn't
imply it's a thing part of a list.

## Explanation

The term proposes is “blob”. The dictionary defines it as “an indeterminate
roundish mass or shape: a big pink blob of a face was at the window.” and in
technical circles it is used to denote a thing with unknown structure or for
[binary large objects](https://en.wikipedia.org/wiki/Binary_large_object).

The term “blob” fits well the purpose of referring to an unknown piece of data
(the attribute-value set of pairs) because, at this level of abstraction it is
intended to be used without knowing its parts.

### Implications

* Rename `/items` and `/items/{hash}` to `/blobs` and `/blobs/{hash}`. To
minimise disruption, use a 301 redirect.
* Rename the internal directory “item” inside the archive (`/download-register`)
  to “blob”.
* Rename the `item-hash` attribute in the entry resource to `blob-hash`. This
  will disrupt clients using entries so we need to analyse the `/entries` and
  `/entries/{number}` usage before making this change.


These changes can be made independent of each other. In particular the latter
can be delayed until we do a breaking change in the API.

The last mention to “item” is part of the record resource. This resource is
ill-defined given that it is supposed to be “the latest entry for a key” but
it is represented in a different shape as an entry resource. This RFC
recommends keeping the record resource as is and assess the implications of a
change independently of the rest.
